# BTRFS 误删恢复实践记录

## 事故背景

- **文件系统**: BTRFS，RAID1（/dev/sda1 + /dev/sdb1）
- **挂载点**: /srv
- **挂载选项**: `compress=zstd,autodefrag,discard=async`
- **误删原因**: `rsync --delete` 误操作删除 /srv/data/ 下大量文件
- **误删后操作**: 立即 umount，但随后重新挂载了 /srv

## 关键时间线（本次事故）

```
gen 10644  →  chunk tree 被更新（backup root: 838640713728）
gen 10646  →  MineCraft 文件完整存在于 fs tree 中（root: 839601291264）
gen 10647  →  rsync --delete 误删发生
gen 10665  →  chunk tree 被重写（当前 chunk root: 838640762880）
gen 10667  →  当前文件系统状态
```

---

## 核心问题：chunk tree 更新导致恢复失败

### 现象

使用标准 `btrfs restore` 恢复 gen 10646 的旧 root 时：
- 小文件（< ~112MB）可完美恢复
- 大文件（> ~140MB）恢复后内容全为 0，md5 不匹配
- 错误信息：`Invalid mapping for X-Y, got Z-W` + `exhausted mirrors trying to read`

### 根因分析

1. `btrfs restore -t <old_root>` 使用**当前 chunk tree** 进行逻辑地址→物理地址的映射
2. gen 10665 的 chunk tree 与 gen 10646 的 chunk tree **映射不同**
3. 旧 root 中的 extent 指针指向的逻辑地址，在当前 chunk tree 中被映射到了错误的物理位置
4. 这些物理位置上的数据已被后续写入覆盖或重新分配
5. 对于 RAID1，两个镜像上的数据都不匹配，导致 `exhausted mirrors` 错误

### 关键证据（本次）

```bash
$ sudo btrfs inspect-internal dump-super /dev/sda1 | grep chunk
chunk_root_generation  10665
chunk_root             838640762880

# sys_chunk_array 中发现旧 chunk tree backup
$ sudo btrfs inspect-internal dump-super -fFa /dev/sda1 | grep backup_chunk_root
backup_chunk_root: 838640762880  gen: 10665  level: 1
backup_chunk_root: 838640713728  gen: 10644  level: 1  <-- 旧 chunk tree
```

旧 chunk tree (gen 10644, 838640713728) 的 CHUNK_ITEM 映射与当前版本显著不同。

---

## 通用诊断流程（适用于任何类似场景）

### 步骤 1：确认 chunk tree 是否被修改

```bash
sudo btrfs inspect-internal dump-super /dev/sdX1 | grep -E "chunk_root_generation|chunk_root$"
```

如果 `chunk_root_generation` 等于当前 `generation`，说明 chunk tree 最近被修改过，这是大文件恢复失败的高危信号。

### 步骤 2：在 superblock backup 中查找旧 chunk tree

```bash
sudo btrfs inspect-internal dump-super -fFa /dev/sdX1 | grep -E "backup_chunk_root|chunk_root"
```

`sys_chunk_array` 可能保留多个历史 chunk tree root。记录所有 `gen` 小于当前 generation 的 backup。

### 步骤 3：扫描所有历史 B-tree root

```bash
sudo btrfs-find-root -a /dev/sdX1 2>&1 | grep "Well block"
```

这会列出所有可识别的历史 B-tree 节点。你需要找到：
- **fs tree root**（owner 通常是 FS_TREE=5，但 btrfs-find-root 可能只显示 level 和 generation）
- 误删发生前最近的 generation（如 gen 10646）

如果 btrfs-find-root 只找到 level 0 的节点，这些通常是叶子节点。可以尝试用它们作为 `-t` 参数，btrfs restore 会自动解析其中的 root 引用。

### 步骤 4：验证旧 chunk tree 是否可读

```bash
sudo btrfs inspect-internal dump-tree -b <old_chunk_root_addr> /dev/sdX1 | head -10
```

如果输出显示 `owner CHUNK_TREE` 和对应的 generation，说明旧 chunk tree 物理上还存在。

### 步骤 5：对比新旧 chunk tree 的映射差异

```bash
# 当前 chunk tree
sudo btrfs inspect-internal dump-tree -t chunk /dev/sdX1 2>/dev/null | grep "CHUNK_ITEM" | head -20

# 旧 chunk tree
sudo btrfs inspect-internal dump-tree -b <old_chunk_root_addr> /dev/sdX1 2>/dev/null | grep "CHUNK_ITEM" | head -20
```

关注 `gen` 字段不同的 CHUNK_ITEM。如果旧 tree 中有一些 chunk 的 generation 早于 chunk_root_generation，而新 tree 中这些 chunk 被替换成了更高 generation 的新 chunk，说明这些区域的映射已经改变。

### 步骤 6：dry-run 验证

先用旧 chunk tree + 旧 fs tree 做 dry-run，确认能找到目标文件：

```bash
sudo ./btrfs restore --chunk-root <old_chunk_addr> -t <old_fs_root_addr> -D -ivv \
  --path-regex '^/(|data(|/target(|/.*))))$' /dev/sdX1 /tmp/
```

---

## 解决方案：修改 btrfs-progs 使用旧 chunk tree

### 原理

btrfs-progs 的 `open_ctree_flags` 结构体原生支持 `chunk_tree_bytenr` 字段，
`read_chunk_root()` 函数也支持传入自定义 chunk root 地址。

### 修改步骤

1. 下载对应版本的 btrfs-progs 源码：
```bash
wget https://www.kernel.org/pub/linux/kernel/people/kdave/btrfs-progs/btrfs-progs-v6.2.tar.xz
tar xf btrfs-progs-v6.2.tar.xz
cd btrfs-progs-v6.2
```

2. 修改 `cmds/restore.c`：
   - 添加静态变量：`static u64 chunk_tree_location = 0;`
   - 在 `long_options[]` 中添加：`{ "chunk-root", required_argument, NULL, GETOPT_VAL_CHUNK_ROOT }`
   - 在 `enum` 中添加：`GETOPT_VAL_CHUNK_ROOT`
   - 在 switch-case 中处理：`case GETOPT_VAL_CHUNK_ROOT: chunk_tree_location = arg_strtou64(optarg); break;`
   - 修改 `open_fs()` 签名，添加 `u64 chunk_tree_location` 参数
   - 在 `open_fs()` 中设置：`ocf.chunk_tree_bytenr = chunk_tree_location;`
   - 修改调用点，传入 `chunk_tree_location`

3. 编译：
```bash
sudo apt-get install -y build-essential libext2fs-dev libzstd-dev liblzo2-dev libblkid-dev uuid-dev libudev-dev pkg-config
./autogen.sh
./configure --disable-documentation
make -j$(nproc)
```

### 恢复命令模板

```bash
sudo ./btrfs restore \
  --chunk-root <OLD_CHUNK_ROOT_ADDR> \
  -t <OLD_FS_ROOT_ADDR> \
  -ivo \
  --path-regex '^/(|data(|/target(|/.*))))$' \
  /dev/sdX1 \
  /path/to/restore/
```

参数说明：
- `--chunk-root`: 旧 chunk tree root 地址（从 superblock backup 或 btrfs-find-root 获得）
- `-t`: 旧 fs tree root 地址（从 btrfs-find-root 获得，选择误删前最近的 generation）
- `-i`: 忽略 transid 验证错误（旧 root 的 generation 与当前 superblock 不匹配，这是正常的）
- `-v`: verbose 输出
- `-o`: 覆盖已存在的文件
- `--path-regex`: 只恢复匹配路径的文件（必须精确匹配完整路径）

---

## 恢复结果（本次）

### MineCraft 目录

- **文件总数**: 53（26 数据文件 + 26 md5 + 1 脚本）
- **总大小**: 108GB
- **md5 验证**: 26/26 全部匹配
- **关键发现**: 之前用当前 chunk tree 只能恢复 7 个文件（且大文件损坏），使用旧 chunk tree 后全部成功

---

## 重要发现

### 误删后写入的影响

重新挂载后约 1.3TB 的写入导致：
- chunk tree 在 gen 10665 被重写
- 旧 extent 的物理映射失效
- 但**小文件**（extent 完全落在旧 chunk 内）仍可恢复
- **大文件**（extent 跨越新旧 chunk 边界）必须用旧 chunk tree 才能恢复

### 文件系统层面的不可逆性

- RAID1 两个设备的 md5 完全一致，说明镜像已同步损坏
- 没有 snapshot
- 块级扫描（PhotoRec）对 zstd 压缩无效
- **旧 chunk tree 是唯一可行的恢复途径**

---

## 工具与环境

- **主机**: Debian 12 Bookworm
- **btrfs-progs 版本**: v6.2（修改后编译）
- **修改后的 btrfs 二进制**: `/opt/workdir/btrfs-progs-v6.2/btrfs`
- **恢复日志**: `/opt/workdir/restore_full2.log`
- **恢复目录**: `/opt/workdir/restore_attempt_3/`

---

## 经验教训

1. **BTRFS 误删后绝对不能重新挂载或继续写入**，chunk tree 更新会彻底封死恢复途径
2. `btrfs restore` 的 `-t` 选项只指定旧 fs tree root，不解决 chunk tree 映射问题
3. `undelete-btrfs` 等脚本使用 `btrfs restore -i` 忽略错误，会**静默产生损坏文件**（md5 不匹配但不会报错）
4. 小文件因 extent 不跨越 chunk 边界，可能幸运地完整恢复
5. superblock 的 `sys_chunk_array` 可能保留旧 chunk tree 的 backup，这是最后的希望
6. **验证是关键**：恢复后必须立即用 md5 或文件格式校验验证完整性，不能仅凭文件存在就认为是完整的

---

记录时间: 2026-04-25
