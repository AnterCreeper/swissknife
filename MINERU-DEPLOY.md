# MinerU 本地推理后端部署指南

> 目标读者：负责在一台 Linux 主机（含 NVIDIA GPU）上部署 MinerU 本地推理后端的 AI agent。
> 部署完成后，主机将提供 OpenAPI 兼容的 PDF 解析服务（pipeline 后端）+ 可选 VLM 后端。
> 本文基于 **2026-04-25 在 Debian 12 + RTX 2060 SUPER 8GB + driver 535.216.01** 上实际跑通的部署过程整理。

---

## 1. 整体架构

部署完成后主机上将运行两个 systemd 服务：

```
┌──────────────────────────────────────────────────────────┐
│  mineru-api.service       (FastAPI, 0.0.0.0:8000)        │
│  └─ pipeline backend  →  本地 GPU + ModelScope 缓存模型  │
│  └─ vlm-http-client   →  转发到 llama-server             │
└──────────────────────────────────────────────────────────┘
                           ↓ HTTP
┌──────────────────────────────────────────────────────────┐
│  llama-server.service     (llama.cpp, 127.0.0.1:8080)    │
│  └─ MinerU2.5-Pro-2604-1.2B GGUF (Q8_0 + mmproj)         │
└──────────────────────────────────────────────────────────┘
```

调用方（如 ScholarAIO）只需访问 `:8000/file_parse`，pipeline 与 VLM 路径都在该服务内部分发。

---

## 2. 硬件 / 系统前置条件

| 项目 | 最低要求 | 实测配置 |
|------|----------|----------|
| GPU | 任意 NVIDIA，VRAM ≥ 6GB | RTX 2060 SUPER 8GB |
| 内存 | ≥ 16GB | 32GB |
| 磁盘 | ≥ 10GB（模型 + 编译缓存） | — |
| OS | Linux + systemd | Debian 12 (bookworm) |
| Python | 3.10–3.11 | 3.11.2 |
| CUDA driver | ≥ 520（决定 torch 选型，见下） | 535.216.01 |

---

## 3. ⚠️ 关键决策：driver / CUDA / torch 兼容性

**这是部署中最容易翻车的地方**。`pip install mineru[pipeline]` 默认会拉**最新** torch，但最新 torch 通常需要更新的 driver。如果不预先选好 torch 版本，会出现 `torch.cuda.is_available() == False`，整个 pipeline 退化为 CPU。

### 3.1 兼容性矩阵

| Driver 主版本 | 支持的最高 CUDA Runtime | 可用 PyTorch wheel |
|---------------|------------------------|---------------------|
| 525.x         | CUDA 12.0              | torch 2.x + cu118  |
| 535.x         | CUDA 12.2              | torch 2.x + **cu118** ✅ 实测路径 |
| 545.x         | CUDA 12.3              | torch 2.x + cu118 / cu121（如有） |
| 550.x         | CUDA 12.4              | torch 2.x + cu118 / cu124 |
| 555+          | CUDA 12.5+             | 最新 torch 默认即可 |

**注意**：PyTorch 2.6 没有 cu121 wheel。组合 `driver 535 + torch 2.6` 时唯一可行的 CUDA build 是 **cu118**。

### 3.2 决策流程

1. `nvidia-smi` 看 driver 版本
2. 如 driver ≥ 555，跳过 3.3，直接默认安装 mineru 即可
3. 如 driver 在 525–550 之间，**先用 cu118 装 torch，再装 mineru** —— 详见步骤 5

---

## 4. 步骤 1：系统准备

```bash
# Debian 12 默认 Python 3.11；其他发行版按需调整
sudo apt update
sudo apt install -y python3 python3-pip python3-venv git build-essential cmake \
                    libcurl4-openssl-dev pkg-config

# CUDA toolkit（仅 nvcc，可选，编译 llama.cpp 需要）
sudo apt install -y nvidia-cuda-toolkit  # Debian 自带 11.8，与 driver 535 兼容

# 验证
nvidia-smi
nvcc --version  # 应该输出 cuda_11.8 或更高
```

PEP 668（Debian 12）禁止 pip 直接装到系统目录，本指南统一用 `--user --break-system-packages` 安装到用户目录 `/home/<user>/.local/`。如果你倾向 venv，把所有 `pip3 install --user --break-system-packages` 替换为在 venv 内 `pip install` 即可。

---

## 5. 步骤 2：安装 PyTorch（cu118）

> 仅当 driver < 555 时需要本步骤。driver 较新者直接跳到步骤 3。

PyPI 默认源会拉最新 torch，必须显式指定 cu118 wheel。**建议用阿里云镜像**（848MB 国内 50 秒下完）：

```bash
pip3 install --user --break-system-packages \
  -i https://mirrors.aliyun.com/pytorch-wheels/cu118/ \
  torch==2.6.0+cu118 torchvision==0.21.0+cu118
```

阿里云该路径是 PEP 503 simple index，pip 会自动找到正确 wheel。**不要**手动下载后重命名 wheel 文件 —— PEP 491 文件名格式必须严格保留 `<name>-<ver>+<build>-<py>-<abi>-<platform>.whl`，pip 会拒绝任何短名。

验证：

```bash
python3 -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
# 期望输出：2.6.0+cu118 11.8 True
```

---

## 6. 步骤 3：安装 MinerU pipeline

```bash
pip3 install --user --break-system-packages \
  "mineru[pipeline]" \
  -i https://mirrors.aliyun.com/pypi/simple
```

**坑 1**：`pip install mineru` 不带 extras 时 **不会** 装 pipeline 依赖（不会拉 paddlepaddle、unimernet 等），必须显式 `mineru[pipeline]`。

**坑 2**：如果先于 torch 安装了 mineru-api 进程并启动过，进程会缓存 "name 'torch' is not defined" 错误。装完 torch 必须重启 mineru-api。

验证：

```bash
which mineru-api  # 期望 /home/<user>/.local/bin/mineru-api
mineru --version
```

---

## 7. 步骤 4：模型镜像源切换为 ModelScope

MinerU 默认从 HuggingFace 拉模型（环境变量 `MINERU_MODEL_SOURCE` 默认值 `huggingface`）。在中国大陆，HF 通常超时 / 失败。**必须设为 `modelscope`**：

```bash
export MINERU_MODEL_SOURCE=modelscope
```

后续 systemd 单元里也要带这个环境变量。首次调用时 mineru 会自动从 ModelScope 拉 `OpenDataLab/PDF-Extract-Kit-1___0` 模型组（约 2.4GB），缓存到：

```
~/.cache/modelscope/hub/models/OpenDataLab/PDF-Extract-Kit-1___0/
```

包含：layout 模型 (PP-DocLayoutV2)、OCR (paddle)、formula (UniMERNet)、table (SlanetPlus)。

---

## 8. 步骤 5（可选）：部署 VLM 后端 llama.cpp

> 仅当需要 VLM 路径时部署。**纯 pipeline 已能满足绝大多数 PDF 解析需求**，VLM 主要在 layout 极端复杂、公式多的场景下质量更优，但每页推理慢得多。

### 8.1 编译 llama.cpp（CUDA）

```bash
sudo mkdir -p /opt/mineru && sudo chown $USER /opt/mineru
cd /opt/mineru
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
cmake -B build -DGGML_CUDA=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j -t llama-server
# 产物：/opt/mineru/llama.cpp/build/bin/llama-server (~9MB)
```

### 8.2 下载 GGUF 模型

```bash
mkdir -p /opt/mineru/models && cd /opt/mineru/models
# 从 modelscope 下载（HF 同名，国内访问受阻）
# 文件名固定：MinerU2.5-Pro-2604-1.2B.Q8_0.gguf  (507MB)
#            MinerU2.5-Pro-2604-1.2B.mmproj-Q8_0.gguf (677MB, 多模态投影层)
# 推荐用 modelscope CLI 或直接 wget modelscope 仓库 raw URL
```

### 8.3 启动参数（重要）

```
-c 4096             # context, 减小以减少内存消耗
--parallel 3        # 并发 slot 数
-ngl 99             # 全部层放 GPU
--no-context-shift  # 禁用滑动窗口，避免 layout token 错乱
--image-min-tokens 1024  # 图像最少 token 数（layout 质量保证）
--special           # ✅ 关键！保留 <|box_start|> 等 special tokens
                    # 如果不加，MinerU 解析 layout 输出会失败
```

显存占用：约 2.2GB（VLM 模型）。在 8GB 卡上与 pipeline (1.3GB) 共存可行。

---

## 9. 步骤 6：MinerU 配置文件

创建 `~/mineru.json`（路径固定，mineru 启动时自动读取）：

```json
{
    "bucket_info": {
        "bucket-name-1": ["ak", "sk", "endpoint"],
        "bucket-name-2": ["ak", "sk", "endpoint"]
    },
    "latex-delimiter-config": {
        "display": {"left": "$$", "right": "$$"},
        "inline":  {"left": "$",  "right": "$"}
    },
    "llm-aided-config": {
        "title_aided": {
            "api_key": "",
            "base_url": "",
            "model": "",
            "enable_thinking": false,
            "enable": false
        }
    },
    "models-dir": {
        "pipeline": "/home/<user>/.cache/modelscope/hub/models/OpenDataLab/PDF-Extract-Kit-1___0",
        "vlm": ""
    },
    "config_version": "1.3.1"
}
```

**关于 `llm-aided-config.title_aided`**：MinerU 用 LLM 修正章节标题层级（H1/H2/H3）。亲测 **deepseek-v4-flash 不严格遵循 prompt 的 JSON 数字格式**（会返回 `"2,1"` 之类导致 `int()` ValueError），retry 5 次后整个文档跳过 —— md 主体不受影响，只是大纲层级保持原样。建议：

- **默认 `enable: false`**（不影响 md 主体内容）
- 如必须启用，换强格式遵循模型如 deepseek-v4-pro / gpt-4o-mini

---

## 10. 步骤 7：systemd 部署

### 10.1 `/etc/systemd/system/llama-server.service`（仅 VLM 用户需要）

```ini
[Unit]
Description=llama.cpp HTTP server (MinerU VLM backend)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=dell
Group=dell
WorkingDirectory=/opt/mineru
Environment=GGML_CUDA_DISABLE_GRAPHS=1
ExecStart=/opt/mineru/llama.cpp/build/bin/llama-server \
  -m /opt/mineru/models/MinerU2.5-Pro-2604-1.2B.Q8_0.gguf \
  --mmproj /opt/mineru/models/MinerU2.5-Pro-2604-1.2B.mmproj-Q8_0.gguf \
  --host 127.0.0.1 --port 8080 \
  -c 4096 --parallel 3 -ngl 99 \
  --no-context-shift --image-min-tokens 1024 --special
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
OOMScoreAdjust=-100

[Install]
WantedBy=multi-user.target
```

> 把 `User=dell` / `Group=dell` 替换为你的部署账户。`/opt/mineru/...` 路径若改变请同步替换。

### 10.2 `/etc/systemd/system/mineru-api.service`

```ini
[Unit]
Description=MinerU FastAPI server (PDF parsing API)
After=network-online.target llama-server.service
Wants=network-online.target llama-server.service

[Service]
Type=simple
User=dell
Group=dell
WorkingDirectory=/home/dell
Environment=MINERU_MODEL_SOURCE=modelscope
Environment=PATH=/home/dell/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=/home/dell/.local/bin/mineru-api --host 0.0.0.0 --port 8000 --allow-public-http-client
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
OOMScoreAdjust=-100

[Install]
WantedBy=multi-user.target
```

**关键参数**：

- `--allow-public-http-client`：允许在非 localhost 接收请求（默认只允许 127.0.0.1）
- `Environment=MINERU_MODEL_SOURCE=modelscope`：避免 systemd 启动时仍走 huggingface
- `Environment=PATH=...`：包含 `/home/dell/.local/bin` 否则找不到 mineru 子命令
- `After=llama-server.service`：保证 VLM 后端先就绪（即使纯 pipeline 用户保留也无害）
- 如果**不部署 VLM**，把 `After=` / `Wants=` 中的 `llama-server.service` 删掉即可

### 10.3 启用与启动

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now llama-server.service mineru-api.service

# 等约 30 秒模型加载完成，然后看状态
sudo systemctl status mineru-api llama-server
```

---

## 11. 步骤 8：验证

### 11.1 端口与服务状态

```bash
sudo systemctl is-active mineru-api llama-server
# 期望两行 active

ss -tlnp | grep -E ':(8000|8080)\s'
# 期望:
# tcp   LISTEN  0  ...  0.0.0.0:8000  ...  python (mineru-api)
# tcp   LISTEN  0  ...  127.0.0.1:8080  ...  llama-server
```

### 11.2 API 可达性

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8000/docs
# 期望 HTTP 200

curl -s http://localhost:8080/health
# 期望 {"status":"ok"} 或类似
```

### 11.3 端到端 PDF 解析

```bash
# 准备一个测试 PDF（例如 1 页的论文摘要）
TEST_PDF=/path/to/test.pdf

curl -X POST http://localhost:8000/file_parse \
  -F "files=@$TEST_PDF" \
  -F "backend=pipeline" \
  -F "lang=en" \
  -F "return_md=true" \
  -o /tmp/result.json

python3 -c "import json; r=json.load(open('/tmp/result.json')); print('md_len:', len(r['results'][0]['md_content']))"
```

预期性能基线（RTX 2060 SUPER + cu118）：

| 模式 | 1 页 | 5 页（稳态） |
|------|------|--------------|
| pipeline GPU | 15.7s（首次含模型加载） | 3.3s |
| pipeline CPU | 15s | 71s（8 页） |
| VLM | 2.87s | 42.6s |

### 11.4 远程访问

如果调用方在另一台机器（如通过 Tailscale），用对应主机 IP 替换 `localhost`：

```bash
curl http://<host-ip>:8000/docs  # 应返回 HTTP 200
```

---

## 12. 故障排查

### 12.1 `torch.cuda.is_available() == False`

→ driver 与 cu** wheel 不匹配。`nvidia-smi` 看 driver 版本，回到第 3 节决策矩阵，重新选 cu** 重装。

### 12.2 `LocalEntryNotFoundError: ...unimernet_hf_small_2503...`

→ 走了 huggingface 而不是 modelscope。检查：

- systemd 单元里有没有 `Environment=MINERU_MODEL_SOURCE=modelscope`
- 不要写在 `~/.bashrc` —— systemd 不读 bash 配置

### 12.3 `name 'torch' is not defined`

→ mineru-api 进程在 torch 安装前已启动且常驻。`sudo systemctl restart mineru-api`。

### 12.4 VLM 输出 `<|box_start|>` 出现在 md 中

→ llama-server 启动参数缺 `--special`。这个 flag 会保留特殊 token 不被字符串化。改 unit 后 `daemon-reload` + `restart llama-server`。

### 12.5 LLM 辅助标题层级 retry 5 次后跳过

→ 选用的 LLM 不严格遵循 JSON 数字格式（实测 deepseek-v4-flash 会返回字符串 `"2,1"`）。换强格式遵循模型，或保持 `enable: false`。

### 12.6 pipeline 速度只有 CPU 水平

→ 可能没真正用上 GPU。检查：

```bash
# 跑一个 PDF 同时另一终端
watch -n0.5 nvidia-smi
# 期望看到 mineru python 进程占用 1–1.5GB 显存
```

如果显存空闲，回到 12.1。

### 12.7 GPU OOM

→ 8GB 卡上 pipeline + VLM 共存基本是显存极限。如出现：

- 关掉 VLM (`stop llama-server`)，纯 pipeline 跑
- 或把 VLM `-ngl` 调小，部分层放 CPU

---

## 13. 日常运维

```bash
# 查看实时日志
sudo journalctl -u mineru-api -f
sudo journalctl -u llama-server -f

# 重启（mineru.json 改动后）
sudo systemctl restart mineru-api

# 查看资源占用
sudo systemctl status mineru-api  # CPU / 内存
nvidia-smi                        # GPU
```

模型缓存位置：

- pipeline: `~/.cache/modelscope/hub/models/OpenDataLab/PDF-Extract-Kit-1___0/` (~2.4GB)
- VLM GGUF: `/opt/mineru/models/` (~1.2GB)
- llama.cpp 编译产物: `/opt/mineru/llama.cpp/build/` (可在编译完后清理 `build/CMakeFiles` 节省空间)

---

## 14. 调用方示例（参考）

任何 OpenAPI 客户端都可使用，简化 Python 示例：

```python
import requests
r = requests.post(
    "http://<host>:8000/file_parse",
    files={"files": open("paper.pdf", "rb")},
    data={"backend": "pipeline", "lang": "en", "return_md": "true"},
    timeout=600,
)
md = r.json()["results"][0]["md_content"]
```

`backend` 可选 `pipeline` / `vlm-http-client`（后者要求步骤 5 已部署）。`lang` 中文 PDF 用 `ch`，英文用 `en`，混合也用 `ch`。

---

## 附录 A：实测部署的版本快照（2026-04-25）

| 组件 | 版本 |
|------|------|
| OS | Debian 12 (bookworm), kernel 6.10.11+bpo-amd64 |
| Python | 3.11.2 |
| NVIDIA driver | 535.216.01 |
| CUDA | 11.8 |
| PyTorch | 2.6.0+cu118 |
| mineru | 3.1.2 |
| GPU | RTX 2060 SUPER 8GB |

## 附录 B：本指南未覆盖的内容

- 多 GPU 部署（mineru-api 暂不原生支持 multi-GPU 负载均衡，需上层反向代理）
- HTTPS / 鉴权（`--allow-public-http-client` 不带认证；如对外暴露建议套 nginx + basic auth）
- Docker 镜像（官方有提供，但本指南聚焦裸机部署，便于 GPU 直通）
- Windows / WSL2（理论可行但路径与 systemd 都需大量改动）
