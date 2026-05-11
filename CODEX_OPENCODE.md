# Codex Plugin for OpenCode

在 OpenCode 中直接调用 OpenAI Codex，无需离开当前会话。

这是 [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc)（官方 Claude Code 插件）的 OpenCode 移植版本。完全复用上游核心脚本，仅适配命令和代理配置格式。

---

## 前置要求

- **Node.js** ≥ 18.18
- **OpenCode** ≥ 1.14
- **Codex CLI** 已安装并登录

检查命令：
```bash
node --version        # v20+
opencode --version    # 1.14+
codex --version       # 应显示版本号
codex login           # 确保已认证
```

---

## 一键安装

复制以下命令到终端执行：

```bash
curl -fsSL https://raw.githubusercontent.com/your-username/codex-plugin-opencode/main/install.sh | bash
```

> 如果无法访问网络，见下方「手动安装」章节。

---

## install.sh

将以下内容保存为 `install.sh`，然后 `bash install.sh` 执行：

```bash
#!/usr/bin/env bash
set -euo pipefail

# ============================================================
# Codex Plugin for OpenCode - One-line Installer
# ============================================================

PLUGIN_SRC="https://github.com/openai/codex-plugin-cc.git"
PLUGIN_DIR="${HOME}/.config/opencode/plugins/codex-plugin-cc"
COMMANDS_DIR="${HOME}/.config/opencode/commands"
AGENT_DIR="${HOME}/.config/opencode/agent"
CONFIG_FILE="${HOME}/.config/opencode/opencode.json"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log_info()  { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }

# --- 1. Check prerequisites ---
log_info "Checking prerequisites..."

command -v git >/dev/null 2>&1 || { log_error "git is required but not installed."; exit 1; }
command -v node >/dev/null 2>&1 || { log_error "Node.js is required but not installed."; exit 1; }
command -v opencode >/dev/null 2>&1 || { log_error "OpenCode is required but not installed."; exit 1; }
command -v codex >/dev/null 2>&1 || { log_warn "Codex CLI not found. Install with: npm install -g @openai/codex"; }

# --- 2. Clone upstream repository ---
if [ -d "${PLUGIN_DIR}/.git" ]; then
    log_info "Updating existing codex-plugin-cc..."
    git -C "${PLUGIN_DIR}" pull --ff-only
else
    log_info "Cloning codex-plugin-cc from GitHub..."
    mkdir -p "$(dirname "${PLUGIN_DIR}")"
    git clone --depth 1 "${PLUGIN_SRC}" "${PLUGIN_DIR}"
fi

# --- 3. Create command files ---
log_info "Creating OpenCode commands..."
mkdir -p "${COMMANDS_DIR}"

COMPANION="${PLUGIN_DIR}/plugins/codex/scripts/codex-companion.mjs"

cat > "${COMMANDS_DIR}/codex-setup.md" << 'EOF'
---
description: Check whether the local Codex CLI is ready and optionally toggle the stop-time review gate
---

Run the Codex setup check to verify installation and authentication status.

Execute:
```bash
node COMPANION_PATH setup --json $ARGUMENTS
```

If the result shows Codex is unavailable:
- Report the exact status to the user
- If npm is available, suggest installing with: `npm install -g @openai/codex`
- If Codex is installed but not authenticated, instruct the user to run: `codex login`

Present the final setup output clearly, including:
- Ready status
- Node and npm availability
- Codex CLI availability
- Authentication status
- Any next steps or warnings
EOF
sed -i "s|COMPANION_PATH|${COMPANION}|g" "${COMMANDS_DIR}/codex-setup.md"

cat > "${COMMANDS_DIR}/codex-review.md" << 'EOF'
---
description: Run a Codex code review against local git state
---

Run a Codex review on the current code changes.

First, check what changes are available to review:
```bash
git status --short --untracked-files=all
git diff --shortstat
git diff --shortstat --cached
```

If there are no changes to review, inform the user and stop.

If the user specified `--base <ref>`, review against that branch. Otherwise review the working tree changes.

Run the review:
```bash
node COMPANION_PATH review "$ARGUMENTS"
```

Return the command output verbatim. Do not:
- Paraphrase or summarize the review
- Fix issues mentioned in the review
- Add commentary before or after the output

If `--background` was specified, inform the user: "Codex review started in the background. Check `/codex-status` for progress."
EOF
sed -i "s|COMPANION_PATH|${COMPANION}|g" "${COMMANDS_DIR}/codex-review.md"

cat > "${COMMANDS_DIR}/codex-adversarial-review.md" << 'EOF'
---
description: Run a Codex review that challenges the implementation approach and design choices
---

Run an adversarial Codex review that questions the implementation, design choices, tradeoffs, and assumptions.

First, check what changes are available to review:
```bash
git status --short --untracked-files=all
git diff --shortstat
git diff --shortstat --cached
```

If there are no changes to review, inform the user and stop.

If the user specified `--base <ref>`, review against that branch. Otherwise review the working tree changes.

Run the adversarial review:
```bash
node COMPANION_PATH adversarial-review "$ARGUMENTS"
```

Return the command output verbatim. Do not:
- Paraphrase or summarize the review
- Fix issues mentioned in the review
- Add commentary before or after the output
- Weaken the adversarial framing

If `--background` was specified, inform the user: "Codex adversarial review started in the background. Check `/codex-status` for progress."
EOF
sed -i "s|COMPANION_PATH|${COMPANION}|g" "${COMMANDS_DIR}/codex-adversarial-review.md"

cat > "${COMMANDS_DIR}/codex-rescue.md" << 'EOF'
---
description: Delegate a coding task to Codex for investigation, fixing, or implementation
---

Delegate a task to Codex to investigate, fix, or implement something.

Check if there's a previous task to resume:
```bash
node COMPANION_PATH task-resume-candidate --json
```

If a resumable task exists and the user didn't specify `--resume` or `--fresh`, ask whether to continue the previous task or start fresh.

Run the task:
```bash
node COMPANION_PATH task "$ARGUMENTS"
```

If `--background` is specified, the task will run in the background. Inform the user to check `/codex-status` for progress.

If `--write` is not specified and the user wants Codex to make changes, suggest adding `--write`.

Return Codex's output verbatim. Do not paraphrase or add commentary.
EOF
sed -i "s|COMPANION_PATH|${COMPANION}|g" "${COMMANDS_DIR}/codex-rescue.md"

cat > "${COMMANDS_DIR}/codex-status.md" << 'EOF'
---
description: Show active and recent Codex jobs for this repository
---

Show the status of Codex jobs.

Run:
```bash
node COMPANION_PATH status "$ARGUMENTS"
```

Present the output in a clear format. If a specific job ID was provided, show the full details. If no job ID was provided, show a summary table of recent jobs.
EOF
sed -i "s|COMPANION_PATH|${COMPANION}|g" "${COMMANDS_DIR}/codex-status.md"

cat > "${COMMANDS_DIR}/codex-result.md" << 'EOF'
---
description: Show the stored final output for a finished Codex job
---

Show the result of a completed Codex job.

Run:
```bash
node COMPANION_PATH result "$ARGUMENTS"
```

Present the full output to the user without summarizing or condensing. Preserve all details including:
- Job ID and status
- Complete result payload
- File paths and line numbers
- Any error messages
- Follow-up commands
EOF
sed -i "s|COMPANION_PATH|${COMPANION}|g" "${COMMANDS_DIR}/codex-result.md"

cat > "${COMMANDS_DIR}/codex-cancel.md" << 'EOF'
---
description: Cancel an active background Codex job
---

Cancel a running Codex job.

Run:
```bash
node COMPANION_PATH cancel "$ARGUMENTS"
```

Report the cancellation status to the user.
EOF
sed -i "s|COMPANION_PATH|${COMPANION}|g" "${COMMANDS_DIR}/codex-cancel.md"

# --- 4. Create agent file ---
log_info "Creating Codex rescue agent..."
mkdir -p "${AGENT_DIR}"

cat > "${AGENT_DIR}/codex-rescue.md" << 'EOF'
---
description: Thin forwarding wrapper for Codex rescue tasks
mode: subagent
model: openai/gpt-4o-mini
permission:
  bash: allow
  edit: deny
  read: deny
  glob: deny
  grep: deny
  list: deny
---

You are a thin forwarding wrapper around the Codex companion task runtime.

Your only job is to forward the user's rescue request to the Codex companion script. Do not do anything else.

Forwarding rules:

- Use exactly one Bash call to invoke the Codex companion script with the `task` subcommand.
- If the user did not explicitly choose `--background` or `--wait`, prefer foreground for small, clearly bounded requests.
- If the task looks complicated, open-ended, multi-step, or likely to run for a long time, prefer background execution.
- Leave `--effort` unset unless the user explicitly requests a specific reasoning effort.
- Leave model unset by default. Only add `--model` when the user explicitly asks for a specific model.
- If the user asks for `spark`, map that to `--model gpt-5.3-codex-spark`.
- Default to a write-capable Codex run by adding `--write` unless the user explicitly asks for read-only behavior.
- Treat `--resume` and `--fresh` as routing controls:
  - `--resume` means add `--resume-last`
  - `--fresh` means do not add `--resume-last`
- Preserve the user's task text as-is apart from stripping routing flags.
- Return the stdout of the command exactly as-is.
- If the Bash call fails or Codex cannot be invoked, report the error clearly.

Do NOT:
- Inspect the repository
- Read files
- Grep or search
- Monitor progress
- Poll status
- Fetch results
- Cancel jobs
- Summarize output
- Do any follow-up work

Command to run:
```bash
node COMPANION_PATH task $ARGUMENTS
```
EOF
sed -i "s|COMPANION_PATH|${COMPANION}|g" "${AGENT_DIR}/codex-rescue.md"

# --- 5. Update opencode.json ---
log_info "Updating opencode.json..."

if [ ! -f "${CONFIG_FILE}" ]; then
    log_error "OpenCode config not found at ${CONFIG_FILE}. Please run opencode at least once."
    exit 1
fi

# Backup
backup_file="${CONFIG_FILE}.backup.$(date +%s)"
cp "${CONFIG_FILE}" "${backup_file}"
log_info "Backup created: ${backup_file}"

# Use node to safely merge JSON
node -e "
const fs = require('fs');
const config = JSON.parse(fs.readFileSync('${CONFIG_FILE}', 'utf8'));

// Add agent
config.agent = config.agent || {};
config.agent['codex-rescue'] = {
  description: 'Thin forwarding wrapper for Codex rescue tasks',
  mode: 'subagent',
  model: 'openai/gpt-4o-mini',
  permission: {
    bash: 'allow',
    edit: 'deny',
    read: 'deny',
    glob: 'deny',
    grep: 'deny',
    list: 'deny'
  }
};

// Add commands
config.command = config.command || {};
config.command['codex-setup'] = {
  description: 'Check whether the local Codex CLI is ready',
  template: 'Run the Codex setup check to verify installation and authentication status. Execute: node ${COMPANION} setup --json \$ARGUMENTS'
};
config.command['codex-review'] = {
  description: 'Run a Codex code review against local git state',
  template: 'Run a Codex review on the current code changes. First check what changes are available: git status --short --untracked-files=all && git diff --shortstat && git diff --shortstat --cached. If there are changes, run: node ${COMPANION} review \\"\$ARGUMENTS\\". Return the output verbatim.'
};
config.command['codex-adversarial-review'] = {
  description: 'Run a Codex review that challenges implementation and design',
  template: 'Run an adversarial Codex review. First check changes: git status --short --untracked-files=all && git diff --shortstat && git diff --shortstat --cached. If there are changes, run: node ${COMPANION} adversarial-review \\"\$ARGUMENTS\\". Return the output verbatim.'
};
config.command['codex-status'] = {
  description: 'Show active and recent Codex jobs',
  template: 'Show Codex job status by running: node ${COMPANION} status \\"\$ARGUMENTS\\". Present the output clearly.'
};
config.command['codex-result'] = {
  description: 'Show the stored final output for a finished Codex job',
  template: 'Show the result of a completed Codex job by running: node ${COMPANION} result \\"\$ARGUMENTS\\". Present the full output without summarizing.'
};
config.command['codex-cancel'] = {
  description: 'Cancel an active background Codex job',
  template: 'Cancel a running Codex job by running: node ${COMPANION} cancel \\"\$ARGUMENTS\\". Report the status.'
};
config.command['codex-rescue'] = {
  description: 'Delegate a coding task to Codex',
  template: 'Delegate a task to Codex. First check for resumable tasks: node ${COMPANION} task-resume-candidate --json. Then run: node ${COMPANION} task \\"\$ARGUMENTS\\". Return output verbatim.'
};

fs.writeFileSync('${CONFIG_FILE}', JSON.stringify(config, null, 2) + '\n');
console.log('Config updated successfully.');
"

# --- 6. Verify ---
log_info "Verifying installation..."
node "${COMPANION}" setup --json > /dev/null 2>&1 && log_info "Codex companion is working!" || log_warn "Codex companion test failed. Check your setup."

log_info "Installation complete!"
echo ""
echo "Available commands:"
echo "  /codex-setup"
echo "  /codex-review"
echo "  /codex-adversarial-review"
echo "  /codex-rescue"
echo "  /codex-status"
echo "  /codex-result"
echo "  /codex-cancel"
echo ""
echo "Available agent:"
echo "  @codex-rescue"
echo ""
echo "Run '/codex-setup' in OpenCode to verify everything is ready."
```

---

## 手动安装

如果无法运行脚本，按以下步骤手动安装：

### 1. 克隆上游仓库

```bash
git clone --depth 1 https://github.com/openai/codex-plugin-cc.git \
  ~/.config/opencode/plugins/codex-plugin-cc
```

### 2. 创建命令文件

在 `~/.config/opencode/commands/` 下创建 7 个 `.md` 文件（内容见上文 install.sh 中的 `cat >` 部分）。

### 3. 创建代理文件

在 `~/.config/opencode/agent/` 下创建 `codex-rescue.md`（内容见上文 install.sh）。

### 4. 更新配置

在 `~/.config/opencode/opencode.json` 中添加：

```json
{
  "agent": {
    "codex-rescue": {
      "description": "Thin forwarding wrapper for Codex rescue tasks",
      "mode": "subagent",
      "model": "openai/gpt-4o-mini",
      "permission": {
        "bash": "allow",
        "edit": "deny",
        "read": "deny",
        "glob": "deny",
        "grep": "deny",
        "list": "deny"
      }
    }
  },
  "command": {
    "codex-setup": { ... },
    "codex-review": { ... },
    ...
  }
}
```

---

## 命令速查

| 命令 | 功能 |
|------|------|
| `/codex-setup` | 检查 Codex CLI 安装和认证状态 |
| `/codex-review` | 标准代码审查（工作区或分支对比） |
| `/codex-adversarial-review` | 对抗性审查，质疑设计决策 |
| `/codex-rescue` | 委派编码任务给 Codex |
| `/codex-status` | 查看所有任务状态 |
| `/codex-result [job-id]` | 查看已完成任务的结果 |
| `/codex-cancel [job-id]` | 取消后台任务 |

### 常用参数

- `--base <ref>` — 对比指定分支（默认审查工作区修改）
- `--background` — 后台运行
- `--wait` — 前台等待结果
- `--model <model>` — 指定模型（如 `gpt-5.4-mini`）
- `--effort <level>` — 推理强度（`none` 到 `xhigh`）
- `--write` — 允许 Codex 修改文件（rescue 默认只读）
- `--resume` / `--fresh` — 继续上一个任务 / 开始新任务

---

## 使用示例

### 审查当前修改
```
/codex-review
```

### 审查相对于 main 分支的修改
```
/codex-review --base main
```

### 后台运行审查
```
/codex-review --background
```

### 对抗性审查
```
/codex-adversarial-review challenge whether this caching design is safe
```

### 委派修复任务
```
/codex-rescue fix the memory leak in the data processor
```

### 使用特定模型快速修复
```
/codex-rescue --model gpt-5.4-mini --effort low fix the typo
```

### 继续上一个任务
```
/codex-rescue --resume apply the top fix from the last run
```

### 查看所有任务
```
/codex-status
```

### 查看特定任务结果
```
/codex-result task-abc123
```

### 使用代理直接委派
```
@codex-rescue investigate why the tests are failing
```

---

## 架构说明

```
用户命令 (/codex-review)
        │
        ▼
OpenCode command (*.md)
        │
        ▼
codex-companion.mjs  ←── 100% 复用上游官方脚本
        │
        ▼
   Codex App Server
        │
        ▼
   OpenAI Codex API
```

**我们改动的只有外壳**：把 Claude Code 的 `commands/*.md` frontmatter 和 `agents/*.md` 配置翻译为 OpenCode 格式。底层 1000+ 行的 `codex-companion.mjs` 业务逻辑零修改。

---

## 与原版差异

| 特性 | Claude Code 原版 | OpenCode 移植版 |
|------|-----------------|----------------|
| 命令前缀 | `/codex:` | `/codex-` |
| Review Gate | 支持（Stop hook） | 不支持 |
| 后台任务 | `Bash(run_in_background)` | 依赖 OpenCode session |
| Agent 调用 | `Agent(codex:codex-rescue)` | `@codex-rescue` |
| 安装方式 | `/plugin install` | 脚本/手动复制 |

---

## 故障排除

### `/codex-setup` 显示 Codex 不可用
```bash
npm install -g @openai/codex
codex login
```

### 命令找不到
确保 `~/.config/opencode/commands/` 下的 `.md` 文件名正确，且 opencode.json 已更新。

### 权限错误
检查 `codex-companion.mjs` 的路径是否正确（应指向 `~/.config/opencode/plugins/codex-plugin-cc/...`）。

### 上游更新
```bash
cd ~/.config/opencode/plugins/codex-plugin-cc
git pull
```
命令文件无需改动，自动使用新版 companion 脚本。

---

## 许可证

与原项目一致：Apache-2.0
