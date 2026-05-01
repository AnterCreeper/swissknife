# EDA 工具链兼容性踩坑复盘

**请以实际情况为准，此文档仅供参考**

> 主机：Debian 13 (trixie) / kernel 6.12.74 / x86_64 / glibc 2.41
> 初始记录：2026-04-25（Cadence + Synopsys）
> 更新记录：2026-04-27（新增 Mentor/Siemens、通用诊断、AI Agent 指南）
> 覆盖工具：Cadence (IC617/618/GENUS/INNOVUS/SPECTRE/XCELIUM)、Synopsys (DC/VCS/LC/SpyGlass/Verdi)、Mentor/Siemens (Calibre/Tessent)

---

## 目录

1. [环境总览](#环境总览)
2. [Cadence 工具链](#一-cadence-工具链)
   - 安装阶段踩坑
   - 运行时踩坑（Virtuoso/Genus/Innovus）
3. [Synopsys 工具链](#二-synopsys-工具链)
   - 安装/环境踩坑
   - 运行时踩坑（DC/VCS/LC/SpyGlass）
4. [Mentor/Siemens 工具链](#三-mentorsiemens-工具链)
   - 环境/库踩坑（Calibre/Tessent）
5. [通用规律与预防清单](#四-通用规律与预防清单)
6. [快速诊断速查表](#五-快速诊断速查表)
7. [给其他 AI Agent 的利用指南](#六-给其他-ai-agent-的利用指南)
8. [已知一次性修复状态](#七-已知一次性修复状态)
9. [附录：环境配置参考](#附录环境配置参考)

---

## 环境总览

### 已安装 EDA 工具

| 厂商 | 工具 | 版本 | 安装路径 |
|------|------|------|---------|
| Synopsys | Design Compiler (DC) | L-2016.03-SP1 | `/opt/synopsys/syn_L-2016.03-SP1` |
| Synopsys | SpyGlass | L2016.06 | `/opt/synopsys/SpyGlass-L2016.06` |
| Synopsys | Library Compiler (LC) | O-2018.06-SP1 | `/opt/synopsys/lc_O-2018.06-SP1` |
| Synopsys | VCS | O-2018.09-SP2 | `/opt/synopsys/vcs-mx_O-2018.09-SP2` |
| Synopsys | Verdi | O-2018.09-SP2 | `/opt/synopsys/Verdi_O-2018.09-SP2` |
| Mentor/Siemens | Tessent | 2021.2 | `/opt/mentor/tessent_2021.2` |
| Mentor/Siemens | Calibre | 2020.3_16.11 | `/opt/mentor/aoj_cal_2020.3_16.11` |
| Cadence | IC (Virtuoso) | 617/618 | `/opt/cadence/IC617` `/opt/cadence/IC618` |
| Cadence | Spectre | 181 | `/opt/cadence/SPECTRE181` |
| Cadence | Genus | 201 | `/opt/cadence/GENUS201` |
| Cadence | Innovus | 191 | `/opt/cadence/INNOVUS191` |
| Cadence | Xcelium | 2109 | `/opt/cadence/XCELIUM2109` |

### 许可证配置

| 工具链 | 许可证文件 | 环境变量 |
|--------|-----------|---------|
| Synopsys | `/opt/synopsys/11.9/admin/license/license.dat` | `SNPSLMD_LICENSE_FILE=27000@<license_host>`, `LM_LICENSE_FILE=...` |
| Mentor/Siemens | `/opt/mentor/LICENSE.DAT` | `MGLS_LICENSE_FILE=/opt/mentor/LICENSE.DAT` |
| Cadence | `/opt/cadence/LICENSE.DAT` | `CDS_LIC_FILE=/opt/cadence/LICENSE.DAT` |

### X11 Display 环境

主机配有真实 X server (`Xorg :0`) + TigerVNC (`x0vncserver` on :0, port 5900)。
GUI 工具（Calibre DRV/RVE、Virtuoso、Innovus、Verdi 等）**必须在有 `$DISPLAY` 的环境下启动**。
无 display 时 `calibredrv` 会优雅退出报 `no $DISPLAY`，但 `calibre -rve` 会在 `QXcbConnection` 初始化时崩溃。

### 环境配置脚本

```bash
# Synopsys + Tessent
source /etc/profile.d/synopsys.sh

# Cadence + Calibre
source /etc/profile.d/cadence.sh
```

**注意**: `cadence.sh` 第 39 行有 `if (! $prompt); then exit; fi` 的 csh 风格检查。
在非交互式 shell / 某些 bash 配置下 source 可能直接 exit。
若遇此问题，改为手动 export 关键变量即可（见附录"快速绕过"）。

---

## 一、Cadence 工具链

### 1.1 Iscape 安装阶段

#### 1.1a 32 位 libc 运行时缺失

**现象：** Iscape 安装程序内置 32 位 Java 1.6（`/runtime/LNX86/bin/java`），直接运行报缺失 32 位库。

**修复：**
```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install libc6:i386
```

**注意：** 不能替换为系统自带的高版本 Java，否则安装过程会**卡死**。

---

#### 1.1b 32 位编译头文件缺失

**现象：** 安装脚本编译 32 位 native 代码时报错：
```
/usr/include/features-time64.h:20:27: fatal error: bits/wordsize.h: No such file or directory
```

**修复：**
```bash
sudo apt install libc6-dev-i386
```

---

#### 1.1c `/bin/sh` 默认是 dash 导致语法错误

**现象：** 安装脚本（如 `xmvhdl64b.ins`）shebang 为 `#!/bin/sh`，但使用了 Bash 数组语法：
```bash
clean_these=( \
    "IEEE/*lnx8664.*.pak" \
    ...
)
```
Debian 上 `/bin/sh` 指向 `dash`，报错 `Syntax error: "(" unexpected`。

**修复：**
```bash
sudo dpkg-reconfigure dash   # 选 "No"，将 /bin/sh 指向 bash
```

---

#### 1.1d Hotfix 安装时 `xmpack -readonly` 导致 `*E,DLPAKW`

**现象：** Base 安装完成后 `transrecord` 的 `sdilib` 被设为只读。打 hotfix 时 `ncvhdl.ins` 尝试 `xmvhdl -update` 重新编译该库：
```
xmvhdl_p: *E,DLPAKW: Attempt to write 'package sdilib.sdilib (AST)' into a read-only library.
```

**修复：** 在以下两个文件中各 4 处（32/64 bit x regular/IEEE_pure），在 `xmvhdl` 调用前插入解锁命令：

文件：
- `/opt/cadence/XCELIUM2109/tools.lnx86/transrecord/files/install/ncvhdl.ins`
- `/opt/cadence/XCELIUM2109/tools.lnx86/transrecord/files/install/ncvhdl64b.ins`

32-bit 版本插入：
```bash
xmpack -unpack sdilib || true
```

64-bit 版本插入：
```bash
xmpack -64BIT -unpack sdilib || true
```

`|| true` 兼容全新安装场景（此时库尚未打包，`-unpack` 会报 `NOTPAK` 但应继续执行）。

---

### 1.2 运行时阶段

#### 1.2a `sysname` 脚本不认识 kernel 6.x

**现象：** Cadence IC617/618 的 OpenAccess 自带 `/share/oa/bin/sysname` 只识别 `3.*` 和 `4.4.*`，在 kernel 6.x 上返回 `unknown`，Virtuoso 无法启动。

```bash
$ /opt/cadence/IC617/share/oa/bin/sysname
unknown
```

**修复：** 在 `check_linux()` 的 `case $version` 中增加 `6.*)` 分支：

```bash
        6.*)
              if [ "$OA_COMPILER" = "" ] ; then
                  compiler="_gcc48x";
              fi
              sysname="linux_rhel50$compiler"; sysnames="$sysname $sysnames";;
```

修改文件：
- `/opt/cadence/IC617/share/oa/bin/sysname`
- `/opt/cadence/IC618/share/oa/bin/sysname`

---

#### 1.2b bundled `libstdc++.so.6` 太旧

**现象：** Virtuoso 启动报错：
```
virtuoso: .../libstdc++.so.6: version `GLIBCXX_3.4.26' not found (required by .../libvisadev.so)
virtuoso: .../libstdc++.so.6: version `CXXABI_1.3.9' not found (required by /lib/x86_64-linux-gnu/libGLU.so.1)
```

**根因：** Cadence 自带老版本 `libstdc++.so.6`（IC617 为 6.0.19， circa 2014），而系统版本（Debian 13 为 6.0.33）带有新符号。Xcelium/Innovus 等工具的共享库（如 `libvisadev.so`）以及系统 `libGLU.so.1` 都需要这些新符号。

**修复：** 全局屏蔽 bundled libstdc++，强制回退系统版本：
```bash
for tool in IC617 IC618 GENUS201 INNOVUS191 SPECTRE181 XCELIUM2109; do
  find /opt/cadence/$tool -name "libstdc++.so*" ! -name "*-gdb.py" -exec mv {} {}.bak \;
done
```

**验证：**
```bash
ldd /opt/cadence/IC617/tools/dfII/bin/64bit/virtuoso | grep libstdc++
# 应显示 => /lib/x86_64-linux-gnu/libstdc++.so.6
```

**何时回滚（判断条件）：**
屏蔽后如果某个工具启动时报 `GLIBCXX_*` 或 `CXXABI_*` **not found**，说明该工具本身编译时就需要较新的 C++ ABI，而系统 libstdc++ 反而不够新（极少见，通常发生在系统极老的情况下）。此时只恢复该工具的备份：
```bash
# 示例：仅恢复 SPECTRE 的 bundled libstdc++
cd /opt/cadence/SPECTRE181
find . -name "libstdc++.so*.bak" -exec sh -c 'f="{}"; mv "$f" "${f%.bak}"' \;
```

**不要回滚的情况：**
- 报 Qt/Mesa/GLX 相关的 segfault（那是 1.2c 的问题，与 libstdc++ 无关）
- 报 `libpng12`、`libxcb`、`BadCursor`（那是其他问题）
- 报 `undefined symbol: xcb_get_reply_fds`（那是 1.2e 的问题）

---

#### 1.2c Genus 启动段错误：Qt5 XCB GLX 与 Mesa 25.x 不兼容

**现象：** Genus 在有 DISPLAY 时启动立即 `Segmentation fault`，无输出即崩溃。`coredumpctl` 栈顶：
```
#0 libgallium-25.0.7-2.so
#4 amdgpu_winsys_create / driCreateNewScreen3
...
#14 glXGetFBConfigs
#15 libqxcb-glx-integration.so
#16 QXcbConnection::QXcbConnection
#25 tcltk_create_TqApplication
```

**根因：** Cadence 自带的 Qt5 XCB 平台插件与系统 Mesa 25.x 的 GLX 初始化不兼容。
- NVIDIA 专有驱动无此问题（`libGLX_nvidia.so` 走独立路径）
- AMD/Intel 核显使用 Mesa，在 Qt5 XCB 初始化 GLX 时触发 segfault
- `LIBGL_ALWAYS_SOFTWARE=1` 也无效，崩溃发生在 `libqxcb-glx-integration.so`

**修复：** 修改 Genus 入口 wrapper 脚本 `/opt/cadence/GENUS201/tools.lnx86/bin/genus`，在 `unset LS_COLORS` 之后添加：
```bash
export QT_XCB_GL_INTEGRATION=none
```

这仅影响当前工具，不影响系统其他 Qt5 程序。

---

#### 1.2d Genus 启动崩溃：systemd NSS 模块导致内存分配器损坏

**现象：** Genus 在 **普通用户** 下启动时崩溃：
```
Loading tool scripts...No more memory. Current memory consumption: 1941MB.
  Requested 18446744071562067968 bytes, errno=12.
TclStackFree: incorrect freePtr (0x7f... != 0x7f...). Call out of sequence?
Segmentation Fault accessing address 600000001.
```

**注意：** root 用户下运行正常。差异在于 root 的某些环境或缓存路径避免了 NSS 模块被加载。

**根因：** `/etc/nsswitch.conf` 的 `hosts` 行默认配置了 systemd NSS 模块：
```
hosts: files myhostname resolve [!UNAVAIL=return] dns
```

当 genus 进行主机名解析时，glibc 动态加载 `libnss_myhostname.so.2` 和 `libnss_resolve.so.2`。这两个模块由 systemd 编译，依赖新 glibc 的内存分配器 ABI。老二进制（genus 20.12 编译时 glibc 2.7）与这些 NSS 模块共存时，`malloc`/`free` 的 ABI 不匹配，导致堆损坏（请求字节数 `0xFFFFFFFF80000000` 是有符号溢出特征）。

**修复：** 修改 `/etc/nsswitch.conf`：
```bash
sudo cp /etc/nsswitch.conf /etc/nsswitch.conf.bak
# 修改 hosts 行为：
hosts: files dns
```

仅去掉 `myhostname` 和 `resolve`，不影响正常 DNS 解析和 `/etc/hosts`。

**影响范围：** 任何由老版本工具链编译的 EDA 二进制在 systemd + glibc 2.34+ 系统上运行时，一旦触发主机名解析都可能崩溃。

---

#### 1.2e Innovus 额外问题：旧版 `libxcb` / `libX11` 冲突 + 缺失库路径

**现象 1：** 启动报错 `libpng12.so.0: cannot open shared object file`

**现象 2：** 加载自带旧版 `libxcb.so.1` 后触发：
```
undefined symbol: xcb_get_reply_fds
```

**现象 3：** KDE Plasma 下光标变成持续旋转的忙状态，控制台大量 `QXcbConnection: BadCursor`

**修复（仅影响 Innovus）：**

1. 屏蔽旧版 libxcb / libX11：
```bash
for f in /opt/cadence/INNOVUS191/tools.lnx86/voltus_components/xp_tools/anls/lib/libxcb.so \
         /opt/cadence/INNOVUS191/tools.lnx86/voltus_components/xp_tools/anls/lib/libxcb.so.1 \
         /opt/cadence/INNOVUS191/tools.lnx86/voltus_components/xp_tools/anls/lib/libxcb.so.1.1.0 \
         /opt/cadence/INNOVUS191/tools.lnx86/voltus_components/xp_tools/anls/lib/libX11.so \
         /opt/cadence/INNOVUS191/tools.lnx86/voltus_components/xp_tools/anls/lib/libX11.so.6 \
         /opt/cadence/INNOVUS191/tools.lnx86/voltus_components/xp_tools/anls/lib/libX11.so.6.3.0; do
  mv "$f" "$f.bak"
done
```

2. 修改 wrapper 脚本 `/opt/cadence/INNOVUS191/tools.lnx86/innovus/bin/innovus`，在 `unset LS_COLORS` 之后添加：
```bash
export QT_XCB_GL_INTEGRATION=none
export LD_LIBRARY_PATH=/lib/x86_64-linux-gnu:/opt/cadence/INNOVUS191/tools.lnx86/voltus_components/xp_tools/anls/lib:/opt/cadence/INNOVUS191/tools.lnx86/tbb/lib/64bit:${LD_LIBRARY_PATH}
```

**注意：** `/lib/x86_64-linux-gnu` 必须排在 `voltus_components` 之前，否则旧版 `libX11.so.6` 会导致 `BadCursor` 错误。

---

## 二、Synopsys 工具链

### 2.1 安装/环境阶段

#### 2.1a `lmgrd` 需要 LSB 兼容链接器

**现象：** `lmgrd` 的 ELF interpreter 硬编码为 `/lib64/ld-lsb-x86-64.so.3`，Debian 13 默认只有 `/lib64/ld-linux-x86-64.so.2`，直接运行报错找不到解释器。

**修复：**
```bash
sudo ln -s /lib64/ld-linux-x86-64.so.2 /lib64/ld-lsb-x86-64.so.3
```

---

#### 2.1b SpyGlass 平台检测脚本不支持 kernel 6.x

**现象：** SpyGlass 的 `perl` 启动脚本和 `standard-environment.sh` 中的 `platform_species()` 只识别 `Linux-2*` / `Linux-3*`，在 kernel 6.x 上返回 `UNKNOWN` 导致启动失败。

**修复：** 修改两处：
1. `/opt/synopsys/SpyGlass-L2016.06/perl/bin/perl`
2. `/opt/synopsys/SpyGlass-L2016.06/SPYGLASS_HOME/lib/SpyGlass/standard-environment.sh`

在各自的 `case` 语句中增加：
```bash
     Linux-4*)
            ...  # 同 Linux-3* 代码块
     Linux-[5-6]*)
            ...  # 同 Linux-3* 代码块
```

bookworm 上的 patch 还注释掉了 Ubuntu IFS 相关代码（第11-27行），Debian 上不需要那段逻辑。

---

#### 2.1c DC / LC 缺少 `libpng12.so.0`

**现象：** Design Compiler 和 Library Compiler 启动时报找不到 `libpng12.so.0`。Debian 13 只有 `libpng16.so.16`，简单的 `ln -s` 不够，因为二进制需要 `PNG12_0` 版本符号。

**修复：** 从 Debian snapshot 下载旧版 deb 并手动提取：
```bash
wget "http://snapshot.debian.org/archive/debian/20151118T214328Z/pool/main/libp/libpng/libpng12-0_1.2.54-1_amd64.deb"
dpkg-deb -x libpng12-0_1.2.54-1_amd64.deb libpng12-extracted
sudo cp libpng12-extracted/lib/x86_64-linux-gnu/libpng12.so.0.54.0 /usr/lib/x86_64-linux-gnu/
sudo ln -sf libpng12.so.0.54.0 /usr/lib/x86_64-linux-gnu/libpng12.so.0
```

---

#### 2.1d License 环境变量

**注意：** 所有 Synopsys 工具启动前必须 source 环境配置：
```bash
source /etc/profile.d/synopsys.sh
```

否则 DC 会报 `Fatal: Design Compiler is not enabled. (DCSH-1)`。

---

### 2.2 运行时阶段

#### 2.2a Library Compiler：`GLIBC_PRIVATE` 符号缺失

**现象：** `lc2_shell_exec` 链接了 glibc 私有符号 `__pthread_unwind@GLIBC_PRIVATE`，在 glibc 2.34+ 上报错：
```
/lib/x86_64-linux-gnu/libpthread.so.0: version `GLIBC_PRIVATE' not found
```

**根因：** glibc 2.34 将 libpthread 合并进 libc.so 后，`libpthread.so.0` 变成空 stub，保留了公开版本标签作为占位符，但**彻底移除了 `GLIBC_PRIVATE` 版本定义**。任何链接了 `__pthread_unwind@GLIBC_PRIVATE` 的老二进制都无法通过动态链接器的版本检查。

**修复：** 编写 `libpthread-shim`（源码仓库：https://github.com/AnterCreeper/eda-glibc-compat），包含：
1. 所有 glibc pthread 版本标签（`GLIBC_2.2.5` ~ `GLIBC_2.41`）
2. 缺失的 `GLIBC_PRIVATE` 版本定义
3. `__pthread_unwind` 符号，内部通过 `dlsym(RTLD_NEXT, "__pthread_unwind_next")` 转发到 libc 实现
4. 备用 `pthread_once` 实现

部署：
```bash
cd eda-glibc-compat/libpthread-shim
./build.sh
sudo cp libpthread.so.0 /opt/synopsys/O-2018.06-SP1/linux64/lc/shlib/
```

**验证状态：** Debian 12 (glibc 2.36) 和 Debian 13 (glibc 2.41) 均已验证，LC 可正常启动并进入交互 shell。

---

#### 2.2b Design Compiler：启动 segfault

**现象：** DC 2016.03-SP1 在 Debian 13 (glibc 2.41) 上启动即 segfault：
```
SIGSEGV {si_code=SEGV_MAPERR, si_addr=0x8004c69dee8}
```

崩溃发生在 `pthread_once` init routine 调用链中。

**排查误区：** 表面像 glibc 版本不兼容。但同一版本 DC 在 Debian 12 (glibc 2.36) 上正常运行，说明不是 glibc 本身的问题。

**真正根因：** 系统 `libstdc++.so.6` 太新。Debian 13 的 libstdc++ 带有 `GLIBCXX_3.4.33`（对应 glibc 2.38+），与 DC 2016 编译时链接的 C++ runtime ABI 不兼容，导致初始化阶段内存布局错位、segfault。

**修复：** 从兼容主机（Debian 12 / bookworm）复制两个库到 DC 安装目录：
```bash
mkdir -p /opt/synopsys/syn_L-2016.03-SP1/lib/AMD.64/
cp /path/to/bookworm/libstdc++.so.6   /opt/synopsys/syn_L-2016.03-SP1/lib/AMD.64/
cp /path/to/bookworm/libgcc_s.so.1    /opt/synopsys/syn_L-2016.03-SP1/lib/AMD.64/
```

**无 bookworm 主机时的替代来源（从 Debian 12 snapshot 提取）：**
```bash
# 下载 Debian 12 的 libstdc++6 包
wget "http://snapshot.debian.org/archive/debian/20230612T000000Z/pool/main/g/gcc-12/libstdc%2B%2B6_12.2.0-14_amd64.deb" -O libstdc++6_12.2.0-14_amd64.deb

# 提取
mkdir -p libstdcxx-extracted
dpkg-deb -x libstdc++6_12.2.0-14_amd64.deb libstdcxx-extracted

# 复制到 DC 目录
mkdir -p /opt/synopsys/syn_L-2016.03-SP1/lib/AMD.64/
cp libstdcxx-extracted/usr/lib/x86_64-linux-gnu/libstdc++.so.6   /opt/synopsys/syn_L-2016.03-SP1/lib/AMD.64/
cp libstdcxx-extracted/usr/lib/x86_64-linux-gnu/libgcc_s.so.1    /opt/synopsys/syn_L-2016.03-SP1/lib/AMD.64/
```

**验证：**
```bash
source /etc/profile.d/synopsys.sh
dc_shell -x 'puts "Hello from DC"; exit'
# 期望输出：初始化成功，打印 Hello from DC，正常退出
```

**为什么这个目录有效：** `snps_common.sh:96-102` 的 `setLibraryPath()` 函数对 `amd64|linux*` 平台自动将 `${install_root}/lib/AMD.64` prepend 到 `LD_LIBRARY_PATH`。DC 启动时会优先加载这里的库，无需 wrapper 脚本、无需 `patchelf`。

**曾验证但不需要的方案：**
| 方案 | 结果 | 结论 |
|------|------|------|
| `patchelf --set-interpreter` + bookworm 完整 glibc | DC 能启动 | 太重，且 ld-linux 和 libc 必须同版本，否则会 "stack smashing detected" |
| wrapper 脚本注入 `LD_LIBRARY_PATH` | 可行 | 修改 vendor 脚本不优雅，`dc_shell` 还被 `design_vision` 等共用 |
| `lib/AMD.64/` 放两个库 | **最优** | 零脚本改动，利用 Synopsys 已有约定 |

---

#### 2.2c VCS：`pthread_yield` 符号缺失

**现象：** VCS 编译阶段报错：
```
undefined reference to 'pthread_yield'
```

**根因：** `pthread_yield()` 在 NPTL 早期为 libpthread 独立函数，glibc 2.34+ 合并 libpthread 进 libc.so 后，该独立符号不再保证存在。其功能与 `sched_yield()` 完全等价。

**修复：** 对 VCS 编译生成的目标文件做符号重定向，无需改源码：
```bash
cd $VCS_HOME/linux64/lib
cp vcs_save_restore_new.o vcs_save_restore_new.o.bak
objcopy --redefine-sym pthread_yield=sched_yield vcs_save_restore_new.o
```

`sched_yield()` 是 POSIX 标准函数，在所有 glibc 版本上稳定存在。

---

#### 2.2d VCS：UVM UTF-8 弯引号导致 GCC 解析错误

**现象：** VCS 编译 UVM 库时 GCC 报错，指向 `uvm_hdl.c` 中的字符无法识别。

**根因：** UVM 源码中混入了 UTF-8 编码的弯引号：
- `0xE2 0x80 0x9C` — 左双引号 `"`
- `0xE2 0x80 0x99` — 右单引号 `'`

GCC 在 C 源码中遇到这些多字节字符时按词法错误处理。

**修复：**
```bash
# 查找含 UTF-8 弯引号的文件
find $VCS_HOME -name "*.c" -exec grep -l $'\xe2\x80\x9c' {} \;

# 替换（针对具体文件）
sed -i 's/\xe2\x80\x9c/"/g; s/\xe2\x80\x9d/"/g; s/\xe2\x80\x98/\x27/g; s/\xe2\x80\x99/\x27/g' uvm_hdl.c
```

---

#### 2.2e SpyGlass：bundled `libfreetype.so.6` 与系统 `libfontconfig` 冲突

**现象：** 运行 `spyglass`（非 `-help`，后者不加载 Qt GUI 路径）时崩溃：
```
/opt/synopsys/SpyGlass-L2016.06/SPYGLASS_HOME/obj/check.Linux4:
symbol lookup error: /lib/x86_64-linux-gnu/libfontconfig.so.1: undefined symbol: FT_Done_MM_Var
```

**根因：** `standard-environment.sh` 将 `${SPYGLASS_HOME}/../dotty/lib/Linux4` prepend 到 `LD_LIBRARY_PATH`。该目录下捆绑了 FreeType 2.3.x (`libfreetype.so.6.3.7`， circa 2007)，远早于 `FT_Done_MM_Var` 加入 FreeType 的 2.9 版本。当系统 `libfontconfig.so.1`（被 Qt4 动态加载）解析 `libfreetype.so.6` 时，LD_LIBRARY_PATH 优先级导致旧版捆绑库被加载，符号缺失。

**修复：**
```bash
cd /opt/synopsys/SpyGlass-L2016.06/dotty/lib/Linux4
for f in libfreetype.so libfreetype.so.6 libfreetype.so.6.3.7; do
  mv "$f" "$f.bak"
done
```

同目录下的 `libpng.so.3.1.2.5` 和 `libexpat.so.0.5.0` SONAME 与系统不同，不会冲突，无需处理。

---

## 三、Mentor/Siemens 工具链

### 3.1 环境/启动阶段

#### 3.1a `/tmp` 权限问题（影响 SpyGlass + 所有写 tmp 的工具）

**现象：**
```
Can't open /tmp/cmdfile_738741 for reading: Permission denied
SpyGlass Rule Checking Aborted.
```

**根因：** `/tmp` 被错误设为 `drwxr-xr-x` (755) 属主 `1001:1001`，普通用户无写入权。

**修复：**
```bash
chmod 1777 /tmp
chown root:root /tmp
```

---

#### 3.1b Calibre — `Invalid Linux operating system`

**现象：** `calibre -h` 直接报错。

**根因：** `calibre_vco` 脚本只白名单 RHEL 5-8 和 SLES 11-12，Debian 13 不在列表中。

**修复：** 在 `/etc/profile.d/cadence.sh` 中添加：
```bash
export USE_CALIBRE_VCO=aoj
```
脚本第 77-79 行：若 `USE_CALIBRE_VCO` 已设置，直接强制使用该 VCO，跳过全部 OS 检测。

---

#### 3.1c Calibre 环境脚本 source 时直接 exit

**现象：** `source /etc/profile.d/cadence.sh` 后直接退出当前 shell。

**根因：** `cadence.sh` 第 39 行 `if (! $prompt); then exit; fi`，`$prompt` 是 csh 变量，bash 中可能为空导致 `exit`。

**修复：** 非交互 shell 下改为手动 export 关键变量（见附录"Calibre 快速绕过"）。

---

### 3.2 运行时阶段

#### 3.2a Calibre RVE — `grep: symbol lookup error: pcre2_set_compile_extra_options_8`

**现象：** 启动 Calibre RVE 或任何内部调用 `grep` 的流程时崩溃。

**根因：** Calibre 2020.3 内嵌 Julia 0.6 运行时，自带旧版 `libpcre2-8.so.0.5.0`（2019-10，PCRE2 ~10.33）。
`set_shared_library_path` 脚本第 159-160 行把 Julia 的 lib 目录注入 `LD_LIBRARY_PATH`：
```
$CALIBRE_HOME/pkgs/icv/julia/0.6/lib
$CALIBRE_HOME/pkgs/icv/julia/0.6/lib/julia
```

系统 `grep` 3.11 编译时链接到系统 `libpcre2-8.so.0.14.0`（PCRE2 10.46），
需要符号 `pcre2_set_compile_extra_options_8`，旧版 Julia pcre2 没有该符号。
`remove_conflicting_libraries` 函数的 `library_list` 中**未包含 `libpcre2`**，
且 `dirs_to_ignore` 保留了 `$CALIBRE_HOME/` 下的目录，因此旧版 pcre2 逃过清理。

**修复：** 备份移走 Julia 自带的旧版 pcre2：
```bash
mv /opt/mentor/aoj_cal_2020.3_16.11/pkgs/icv.aoj/julia/0.6/lib/julia/libpcre2-8.so.0.5.0 \
   /opt/mentor/aoj_cal_2020.3_16.11/pkgs/icv.aoj/julia/0.6/lib/julia/libpcre2-8.so.0.5.0.bak
```

**安全说明：** PCRE2 C API 向后兼容。Julia 0.6 编译时链接到旧版 pcre2，但运行时使用系统新版 pcre2 安全，
因为旧程序调用新库不会触发 ABI 破坏（新版库包含全部旧符号）。

---

#### 3.2b Tessent shell / Calibre GUI — GTK2 缺失

**现象：**
```
tessent_shell.exe: error while loading shared libraries: libgdk-x11-2.0.so.0
calibredrv: error while loading shared libraries: libgtk-x11-2.0.so.0
```

**根因：** Debian 13 默认不安装 GTK2。

**修复：**
```bash
apt-get install -y libgtk2.0-0
```

---

#### 3.2c Calibre GUI — GStreamer 0.10 缺失

**现象：** `libgstbase-0.10.so.0: cannot open shared object file`

**根因：** Debian 13 仓库只有 GStreamer 1.x（1.26.2），Calibre 2020.3 GUI 依赖已淘汰的 GStreamer 0.10。

**修复：** 从 Debian bullseye 归档手动下载并提取 so 文件到 Calibre lib 目录：

```bash
mkdir -p /tmp/gst010 && cd /tmp/gst010

curl -LO http://archive.debian.org/debian/pool/main/g/gstreamer0.10/\
    libgstreamer0.10-0_0.10.36-1.5_amd64.deb
curl -LO http://archive.debian.org/debian/pool/main/g/gst-plugins-base0.10/\
    libgstreamer-plugins-base0.10-0_0.10.36-2_amd64.deb

dpkg-deb -x libgstreamer0.10-0_0.10.36-1.5_amd64.deb gstreamer
dpkg-deb -x libgstreamer-plugins-base0.10-0_0.10.36-2_amd64.deb plugins-base

cp gstreamer/usr/lib/x86_64-linux-gnu/libgst*.so.0* \
   /opt/mentor/aoj_cal_2020.3_16.11/pkgs/icv_lib/lib64/
cp gstreamer/usr/lib/x86_64-linux-gnu/libgstreamer-0.10.so.0* \
   /opt/mentor/aoj_cal_2020.3_16.11/pkgs/icv_lib/lib64/
cp plugins-base/usr/lib/x86_64-linux-gnu/libgst*.so.0* \
   /opt/mentor/aoj_cal_2020.3_16.11/pkgs/icv_lib/lib64/
```

---

#### 3.2d Calibre RVE 无 display 崩溃（环境问题，非库缺失）

**现象：** 无 `$DISPLAY` 时 `calibre -rve` 在 `QXcbConnection` 处 SIGABRT；
`calibredrv` 则优雅退出报 `no $DISPLAY`。

**根因：** 主机运行 Xorg :0 + TigerVNC，GUI 工具必须在有 display 的环境下启动。
RVE 的 Qt XCB 初始化在无 display 时不做 graceful fallback，直接 crash。

**修复：** 确保 `$DISPLAY` 指向可用 X server（VNC 已连接或 X forwarding 已设置）。

**验证：**
```bash
DISPLAY=:0 calibredrv      # 正常：license 获取、workspace 创建、8 cores
DISPLAY=:0 calibre -rve    # 正常："GUI startup Complete"、"RVE authorized"
```

---

## 四、通用规律与预防清单

### 老 EDA 工具迁移到新 glibc 系统的典型问题模式

| 问题类型 | 典型表现 | 修复策略 | 涉及工具 |
|----------|----------|----------|----------|
| C++ runtime ABI 不兼容 | 启动 segfault / 异常崩溃 | 放置兼容版 `libstdc++.so.6` 到工具私有 lib 目录 | DC, Cadence 全系列 |
| 已移除/弱化的 libpthread 符号 | `undefined reference` | `objcopy --redefine-sym` 转发到 libc 等价符号 | VCS |
| `GLIBC_PRIVATE` 缺失 | `version 'GLIBC_PRIVATE' not found` | 编译 libpthread-shim 提供版本标签占位 | LC |
| 源码编码问题 | GCC parser error on non-ASCII | 替换 UTF-8 弯引号为 ASCII 直引号 | VCS UVM |
| bundled 库与系统库冲突 | `undefined symbol` / `BadCursor` | 重命名/屏蔽 bundled 旧库，强制回退系统版本 | SpyGlass, Innovus |
| Qt5 XCB GLX 与 Mesa 不兼容 | 有 DISPLAY 时启动即 segfault | `export QT_XCB_GL_INTEGRATION=none` | Genus, Innovus |
| systemd NSS 模块 ABI 冲突 | 普通用户下内存分配器崩溃 | `/etc/nsswitch.conf` 去掉 `myhostname resolve` | Genus, 潜在影响所有老二进制 |
| 内核版本检测脚本过时 | `unknown` / `UNKNOWN` | 在 `case` 中增加 `6.*)` 分支 | Cadence sysname, SpyGlass |
| 32 位运行时缺失 | 无法启动内置 32 位 Java | `apt install libc6:i386 libc6-dev-i386` | Cadence Iscape |
| 旧版共享库缺失 | `cannot open shared object file` | 从 Debian snapshot 下载旧版 deb 提取 | DC/LC (libpng12) |
| bundled 旧库劫持系统命令 | `symbol lookup error`（grep/其他系统工具） | 重命名/屏蔽 bundled 旧库 | Calibre (pcre2) |
| OS 版本白名单拒绝 | `Invalid Linux operating system` | 强制指定 VCO 绕过检测 | Calibre |
| 图形框架版本淘汰 | GUI 启动报库缺失 | 从旧版 OS 归档提取 so | Calibre (GTK2, GStreamer 0.10) |

### 新系统安装老 EDA 工具的预防性 checklist

1. [ ] `/bin/sh` 指向 bash（Cadence 安装脚本需要）
2. [ ] `dpkg --add-architecture i386` + `libc6:i386 libc6-dev-i386`（Cadence 安装需要）
3. [ ] `/lib64/ld-lsb-x86-64.so.3` 软链接（Synopsys lmgrd 需要）
4. [ ] `/etc/nsswitch.conf` 的 `hosts` 行去掉 `myhostname resolve`（预防老二进制 NSS 崩溃）
5. [ ] 安装完成后全局屏蔽 bundled `libstdc++.so*`（Cadence）
6. [ ] 检查 `sysname` / `platform_species` 是否支持 kernel 6.x
7. [ ] 检查是否需要 `libpng12.so.0` 等旧版系统库
8. [ ] 有 DISPLAY 的 Qt5 工具测试时准备 `QT_XCB_GL_INTEGRATION=none`
9. [ ] 启动前 source 环境配置（Synopsys 需要 license 变量）
10. [ ] Calibre 启动前确认 `USE_CALIBRE_VCO=aoj` 已设置
11. [ ] GUI 工具测试前确认 `$DISPLAY` 指向可用 X server

---

## 五、快速诊断速查表

| 症状 | 最可能原因 | 诊断命令 |
|------|-----------|---------|
| `Permission denied` on /tmp | /tmp 权限不对 | `ls -ld /tmp` |
| `Invalid Linux operating system` | Calibre OS 白名单不包含 Debian 13 | `cat /etc/os-release` |
| `grep: undefined symbol: pcre2_set_compile_extra_options_8` | Julia 旧版 pcre2 劫持系统 grep | `ldd $(which grep) \| grep pcre2` |
| `libgtk-x11-2.0.so.0: No such file` | GTK2 未安装 | `dpkg -l \| grep libgtk2` |
| `libgstbase-0.10.so.0: No such file` | GStreamer 0.10 已淘汰 | `apt list --installed \| grep gstreamer` |
| `Unable to resolve all shared libraries` | 多库缺失，通常始于 GTK2 | `ldd <binary> \| grep "not found"` |
| `no display name and no $DISPLAY` | 无 X11 display，GUI 无法启动 | `echo $DISPLAY` |
| `QXcbConnection` + crash / SIGABRT | 无 display 时 calibre -rve 崩溃（有 display 时正常） | 确认 `$DISPLAY` 指向可用 X server |
| `version 'GLIBCXX_*' not found` | bundled libstdc++ 太旧 | `ldd <binary> \| grep libstdc++` |
| `undefined symbol: xcb_get_reply_fds` | Innovus 旧版 libxcb 冲突 | 检查 `voltus_components/xp_tools/anls/lib/` |
| `No more memory...errno=12` | systemd NSS 模块 ABI 冲突 | 检查 `/etc/nsswitch.conf` 的 `hosts` 行 |
| `GLIBC_PRIVATE' not found` | glibc 2.34+ 合并 libpthread 后移除私有版本标签 | `ldd <binary> \| grep libpthread` |

---

## 六、给其他 AI Agent 的利用指南

### 启动前必做（按顺序）

```bash
# 1. 修复 /tmp（一次性，已修复则跳过）
chmod 1777 /tmp && chown root:root /tmp

# 2. 确认 X11 display（GUI 工具必需）
echo "DISPLAY=$DISPLAY"
# 若无 display 且需要 GUI，启动 xvfb 或确保 VNC/X forwarding 已连接

# 3. 加载环境
source /etc/profile.d/synopsys.sh
source /etc/profile.d/cadence.sh
# 注意: cadence.sh 的 $prompt 检查可能导致非交互 shell exit

# 4. 验证核心库
grep --version          # 应正常，无 pcre2 错误
ldd $(which grep) | grep pcre2   # 应指向 /lib/x86_64-linux-gnu/
```

### cadence.sh 快速绕过（非交互 shell 场景）

若 `source /etc/profile.d/cadence.sh` 直接 exit，手动 export：

```bash
export CALIBRE_HOME=/opt/mentor/aoj_cal_2020.3_16.11
export MGLS_LICENSE_FILE=/opt/mentor/LICENSE.DAT
export USE_CALIBRE_VCO=aoj
export PATH=$PATH:$CALIBRE_HOME/bin
export MGC_LIB_PATH=$CALIBRE_HOME/lib
export MGC_CALIBRE_REALTIME_VIRTUOSO_ENABLED=1
export MGC_CALIBRE_SAVE_ALL_RUNSET_VALUES=1
export MGC_PDF_READER=evince
```

### 新增工具时的检查清单

1. **OS 兼容性**: 检查工具的 OS 白名单，Debian 13 通常需要 `USE_CALIBRE_VCO` 或类似 bypass
2. **/tmp 依赖**: 任何创建临时文件的工具都可能触发 /tmp 权限问题
3. **GTK2 依赖**: GUI 工具大概率需要 `libgtk2.0-0`
4. **GStreamer 0.10 依赖**: 老版本 GUI 工具（尤其 2018-2020 年发布）可能需要从旧版 Debian 提取
5. **库劫持**: 检查工具是否自带旧版系统库（pcre2, glib, xml2, freetype 等），可能通过 `LD_LIBRARY_PATH` 劫持系统命令
6. **libstdc++ 兼容**: 老工具在新系统上常因 C++ ABI 不匹配 segfault，优先屏蔽 bundled libstdc++
7. **Qt5 GLX**: 有 DISPLAY 的 Qt5 工具在 Mesa 驱动下可能 segfault，准备 `QT_XCB_GL_INTEGRATION=none`
8. **systemd NSS**: 老二进制触发主机名解析时可能因 `libnss_myhostname` 崩溃，检查 `/etc/nsswitch.conf`
9. **许可证**: 确认 `LM_LICENSE_FILE` / `MGLS_LICENSE_FILE` / `SNPSLMD_LICENSE_FILE` / `CDS_LIC_FILE` 指向正确

---

## 七、已知一次性修复状态

| 修复项 | 状态 | 持久性 | 涉及文件/路径 |
|--------|------|--------|--------------|
| `/tmp` 权限 1777 | 已修复 | 持久（除非被重置） | `/tmp` |
| `/bin/sh` 指向 bash | 已修复 | 持久 | `/bin/sh` |
| `ld-lsb-x86-64.so.3` 软链接 | 已修复 | 持久 | `/lib64/ld-lsb-x86-64.so.3` |
| `/etc/nsswitch.conf` 去掉 myhostname/resolve | 已修复 | 持久 | `/etc/nsswitch.conf` |
| Cadence bundled `libstdc++.so*` 全局屏蔽 | 已修复 | 持久 | `/opt/cadence/*/.../libstdc++.so*` |
| Cadence `sysname` kernel 6.x 支持 | 已修复 | 持久 | `IC617/IC618/share/oa/bin/sysname` |
| `libpng12.so.0` 部署到系统 lib | 已修复 | 持久 | `/usr/lib/x86_64-linux-gnu/libpng12.so.0` |
| DC `lib/AMD.64/` 兼容库 | 已修复 | 持久 | `/opt/synopsys/syn_L-2016.03-SP1/lib/AMD.64/` |
| LC `libpthread-shim` | 已修复 | 持久 | `/opt/synopsys/lc_O-2018.06-SP1/linux64/lc/shlib/libpthread.so.0` |
| VCS `pthread_yield` → `sched_yield` | 已修复 | 持久 | `vcs_save_restore_new.o` |
| VCS UVM UTF-8 弯引号替换 | 已修复 | 持久 | `uvm_hdl.c` 等 |
| SpyGlass kernel 6.x 平台检测 | 已修复 | 持久 | `perl/bin/perl`, `standard-environment.sh` |
| SpyGlass bundled `libfreetype` 屏蔽 | 已修复 | 持久 | `dotty/lib/Linux4/libfreetype.so*` |
| Genus `QT_XCB_GL_INTEGRATION=none` | 已修复 | 持久 | `GENUS201/tools.lnx86/bin/genus` |
| Innovus libxcb/libX11 屏蔽 + lib 路径 | 已修复 | 持久 | `INNOVUS191/tools.lnx86/voltus_components/...` |
| `USE_CALIBRE_VCO=aoj` | 已写入 cadence.sh | 持久 | `/etc/profile.d/cadence.sh` |
| Julia 旧版 pcre2 移走 | 已备份为 .bak | 持久（升级 Calibre 需重新处理） | `/opt/mentor/.../julia/.../libpcre2-8.so.0.5.0.bak` |
| GTK2 安装 | 已 apt install | 持久（除非系统重装） | `libgtk2.0-0` |
| GStreamer 0.10 so 文件 | 已放入 Calibre lib64 | 持久（升级 Calibre 需重新处理） | `/opt/mentor/.../pkgs/icv_lib/lib64/` |

---

## 附录：环境配置参考

### `/etc/profile.d/synopsys.sh`

```bash
# License Server
export SNPSLMD_LICENSE_FILE=27000@<license_host>
export LM_LICENSE_FILE=/opt/synopsys/11.9/admin/license/license.dat
export PATH=$PATH:/opt/synopsys/11.9/amd64/bin

# Design Compiler
export DC_HOME=/opt/synopsys/syn_L-2016.03-SP1
export PATH=$PATH:$DC_HOME/bin

# SpyGlass
export SPYGLASS_HOME=/opt/synopsys/SpyGlass-L2016.06/SPYGLASS_HOME
export PATH=$PATH:$SPYGLASS_HOME/bin

# Library Compiler
export LC_HOME=/opt/synopsys/lc_O-2018.06-SP1
export SYNOPSYS_LC_ROOT=$LC_HOME
export PATH=$PATH:$LC_HOME/bin

# VCS
export VCS_HOME=/opt/synopsys/vcs-mx_O-2018.09-SP2
export PATH=$PATH:$VCS_HOME/bin

# Verdi
export VERDI_HOME=/opt/synopsys/Verdi_O-2018.09-SP2
export PATH=$PATH:$VERDI_HOME/bin
```

### 快速验证命令

```bash
# DC
dc_shell -x 'puts "Hello from DC"; exit'

# LC
lc_shell -version

# VCS
vcs -version

# Verdi
verdi -version

# SpyGlass
spyglass -help

# lmgrd 状态
lmstat -a -c 27000@silicon
```

---

### `/etc/profile.d/cadence.sh`（Calibre 段关键变量）

```bash
# Mentor Graphics Calibre
export CALIBRE_HOME=/opt/mentor/aoj_cal_2020.3_16.11
export MGLS_LICENSE_FILE=/opt/mentor/LICENSE.DAT
export USE_CALIBRE_VCO=aoj        # Debian 13 OS 检测 bypass
export PATH=$PATH:$CALIBRE_HOME/bin
export MGC_LIB_PATH=$CALIBRE_HOME/lib
export MGC_CALIBRE_REALTIME_VIRTUOSO_ENABLED=1
export MGC_CALIBRE_SAVE_ALL_RUNSET_VALUES=1
export MGC_PDF_READER=evince
```

### Calibre 快速绕过（cadence.sh source 失败时）

```bash
export CALIBRE_HOME=/opt/mentor/aoj_cal_2020.3_16.11
export MGLS_LICENSE_FILE=/opt/mentor/LICENSE.DAT
export USE_CALIBRE_VCO=aoj
export PATH=$PATH:$CALIBRE_HOME/bin
export MGC_LIB_PATH=$CALIBRE_HOME/lib
export MGC_CALIBRE_REALTIME_VIRTUOSO_ENABLED=1
export MGC_CALIBRE_SAVE_ALL_RUNSET_VALUES=1
export MGC_PDF_READER=evince
```

### 快速验证命令（补充 Mentor）

```bash
# Tessent
tessent -shell <<<'puts "Tessent OK"; exit'

# Calibre 命令行
calibre -h

# Calibre GUI（需要 DISPLAY）
DISPLAY=:0 calibredrv
DISPLAY=:0 calibre -rve
```

---

> 本文档覆盖截至 2026-04-27 的所有已知兼容性踩坑。后续如遇到新问题，按「现象 → 根因 → 修复 → 验证」四段式追加。
