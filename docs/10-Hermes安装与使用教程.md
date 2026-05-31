# 10 Hermes Agent 安装与使用教程

> 基于 Hermes Agent 官方文档（v0.8.0）撰写的安装与使用指南。官网：https://hermes-agent.nousresearch.com/docs/zh-Hans

---

## 一、安装

### 1.1 环境要求

| 依赖 | 最低版本 |
|------|---------|
| Python | 3.11+ |
| Node.js | 22+（部分功能需要）|
| Git | 任意（安装器可自动处理可移植版）|
| 操作系统 | Linux / macOS / WSL2 / Windows / Android (Termux) |

### 1.2 安装方式

#### Linux / macOS / WSL2 / Termux（推荐 —— 跟踪 main 分支）

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

安装完成后运行：

```bash
source ~/.bashrc
```

#### Windows（PowerShell，早期 Beta）

```powershell
iex (irm https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1)
```

安装程序自动处理：`uv`、Python 3.11、Node.js 22、ripgrep、ffmpeg 以及可移植 Git。仓库克隆到 `%LOCALAPPDATA%\hermes\hermes-agent`，创建虚拟环境，将 `hermes` 添加到用户 PATH。

> **注意：** Windows 原生支持除浏览器仪表板聊天面板外的所有功能（仪表板仅限 WSL2，因使用 POSIX PTY）。

#### pip 安装（跟踪带标签的发布版本）

```bash
pip install hermes-agent
hermes postinstall
```

### 1.3 安装目录结构

| 安装方式 | 代码位置 | 二进制文件 | 数据目录 |
|----------|---------|-----------|---------|
| pip install | Python site-packages | `~/.local/bin/hermes` | `~/.hermes/` |
| 用户级 git | `~/.hermes/hermes-agent/` | `~/.local/bin/hermes`（符号链接）| `~/.hermes/` |
| 根模式 | `/usr/local/lib/hermes-agent/` | `/usr/local/bin/hermes` | `/root/.hermes/` 或 `$HERMES_HOME` |

### 1.4 验证安装

```bash
hermes --version
hermes doctor
```

`hermes doctor` 会检查所有依赖、配置完整性，以及安全公告（如供应链扫描）。

---

## 二、快速开始

### 2.1 配置 LLM Provider

安装完成后首次运行 `hermes` 会进入交互式配置向导。你也可以直接编辑配置文件：

```yaml
# ~/.hermes/config.yaml
api:
  provider: openrouter       # 或 anthropic / openai / nous
  model: anthropic/claude-sonnet-4-6
  base_url: https://openrouter.ai/api/v1
  api_key: ${OPENROUTER_API_KEY}   # 支持环境变量引用
```

支持 15+ Provider 的开箱即用配置，包括 OpenAI、Anthropic、OpenRouter、Nous Portal、DeepSeek、Qwen 等。

### 2.2 开始对话

```bash
# 经典 CLI（基于 prompt_toolkit，多行编辑、斜杠命令补全）
hermes

# 现代 TUI（模态覆盖层、鼠标选择）
hermes --tui

# 单次查询（非交互模式）
hermes chat -q "解释这段代码：cat main.py"

# 恢复最近的会话
hermes --continue
# 或
hermes -c
```

---

## 三、CLI 使用详解

### 3.1 状态栏

输入区上方实时显示：

- **模型名称**（当前使用的 LLM）
- **令牌计数**（已用 / 最大）
- **上下文进度条**：绿色 <50%，黄色 50-80%，橙色 80-95%，红色 ≥95%
- **估计成本**（美元）
- **压缩次数** `[N]`（当前会话中自动压缩次数）
- **后台任务数**
- **会话时长**
- **YOLO 模式警告**

### 3.2 键盘快捷键

| 快捷键 | 功能 |
|--------|------|
| `Enter` | 发送消息 |
| `Alt+Enter` / `Ctrl+J` / `Shift+Enter` | 换行 |
| `Ctrl+B` | 切换语音录制 |
| `Ctrl+G` | 在 `$EDITOR` 中打开输入缓冲区 |
| `Ctrl+C` | 中断 Agent（2 秒内双击强制退出）|
| `Ctrl+D` | 退出 |
| `Ctrl+Z` | 将 Hermes 置于后台（Unix）|

### 3.3 核心斜杠命令

| 命令 | 功能 |
|------|------|
| `/help` | 显示帮助 |
| `/model` | 切换当前模型 |
| `/tools` | 查看可用工具 |
| `/skills browse` | 浏览已安装技能 |
| `/background <prompt>` | 在后台运行独立 Agent 任务 |
| `/skin` | 更换主题 |
| `/voice on` | 开启语音输入 |
| `/voice off` | 关闭语音输入 |
| `/reasoning high` | 开启高推理模式 |
| `/reasoning off` | 关闭推理模式 |
| `/title <名称>` | 为当前会话命名 |
| `/compress` | 手动触发上下文压缩 |
| `/yolo` | 切换 YOLO 模式（跳过命令审批）|
| `/busy` | 设置忙时输入模式 |

### 3.4 忙时输入模式

通过 `/busy` 或配置 `display.busy_input_mode` 设置：

| 模式 | 行为 |
|------|------|
| `interrupt`（默认）| 立即中断正在进行的操作 |
| `queue` | 静默排队，待 Agent 完成后发送 |
| `steer` | 通过 `/steer` 注入到当前运行中（不中断）|

### 3.5 后台会话

在独立守护线程中运行 Agent 任务，完全非阻塞：

```bash
# 启动后台任务
/background 帮我研究一下这个项目里所有的 API 端点

# 启动时预加载技能
hermes -s skill1,skill2
```

支持多个并发后台任务，每个有独立 ID。

### 3.6 隔离 Worktree 模式

在隔离的 git worktree 中运行 Agent：

```bash
hermes -w
```

---

## 四、高级功能

### 4.1 内存系统

**自动记忆：** Agent 在对话中自动通过 `memory` 工具管理 MEMORY.md 和 USER.md。

**手动管理：**

```bash
# 查看当前记忆
cat ~/.hermes/memories/MEMORY.md
cat ~/.hermes/memories/USER.md

# 对话中操作记忆
# Agent 会自行调用 memory 工具，也可以直接要求：
"把"这个项目用的是 Go 1.22 和 PostgreSQL"记到记忆里"
```

**配置外部 Memory Provider：**

```bash
# 列出可用的记忆后端
hermes memory list

# 配置 Honcho 作为外部后端
hermes memory setup honcho
```

### 4.2 技能系统 (Skills)

Agent 可以在复杂任务成功后**自动创建技能**，把方法论固化下来。

技能目录结构：

```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md           # YAML frontmatter + Markdown 正文
│   ├── references/        # 参考资料
│   ├── templates/         # 模板文件
│   ├── scripts/           # 脚本
│   └── assets/            # 其他资源
```

### 4.3 会话管理

```bash
# 查看会话列表
hermes sessions list

# 恢复指定会话
hermes sessions resume <session_id>

# 搜索历史对话
# 在对话中使用 session_search 工具
```

所有会话存储在 SQLite `~/.hermes/state.db` 中，带 FTS5 全文搜索。

### 4.4 Gateway 模式（多平台消息网关）

Hermes 可以作为消息网关运行，同时服务 11 个平台：

```bash
hermes gateway
```

支持平台：Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Email、HomeAssistant、Mattermost、DingTalk（钉钉）、Feishu（飞书）。

Gateway 配置见 `~/.hermes/config.yaml` 的 `gateway` 部分。

### 4.5 安全模式

```bash
# YOLO 模式（跳过命令审批，有红色横幅警告）
hermes --yolo
# 或在对话中输入 /yolo

# 通过环境变量启用
HERMES_YOLO_MODE=1 hermes

# Docker 沙箱模式（推荐生产环境使用）
# 在 config.yaml 中设置:
# terminal:
#   backend: docker
```

---

## 五、配置参考

### 5.1 核心配置项

```yaml
# ~/.hermes/config.yaml

# Agent 配置
agent:
  max_turns: 90                          # 最大迭代次数
  tool_delay: 1.0                        # 工具调用延迟

# 压缩配置
compression:
  threshold: 0.50                        # 触发阈值（50% 上下文窗口）
  protect_last_n: 20                     # 尾部保护消息数
  target_ratio: 0.20                     # 摘要目标比例
  summary_model: ""                      # 摘要专用模型（留空用主模型）

# 终端配置
terminal:
  backend: local                         # local / docker / ssh / modal / daytona / singularity
  env_passthrough: []                    # 传递的环境变量
  docker:
    memory_limit: 5GB
    disk_limit: 50GB

# 安全配置
security:
  approvals:
    mode: manual                         # manual / smart / off
  allow_private_urls: false              # 允许访问私有网络
  allow_lazy_installs: true              # 允许惰性安装依赖

# 记忆配置
memory:
  provider: ""                           # 外部 provider，留空只用内置
```

### 5.2 凭证池配置

支持同一 Provider 使用多个 API Key，调度策略可选：

```yaml
api:
  provider: openrouter
  credentials:
    - key: ${KEY_1}
    - key: ${KEY_2}
  credential_strategy: round_robin       # fill_first / round_robin / random / least_used
```

### 5.3 Fallback Provider 链

```yaml
api:
  provider: anthropic
  model: claude-sonnet-4-6
  fallback_providers:
    - provider: openrouter
      model: anthropic/claude-sonnet-4-6
    - provider: openai
      model: gpt-5
```

---

## 六、批量训练数据生成

### 6.1 准备数据集

```jsonl
{"prompt": "用 Python 写一个快速排序算法"}
{"prompt": "创建一个 Flask REST API，包含用户 CRUD 操作"}
{"prompt": "用 React 写一个 Todo 列表组件"}
```

### 6.2 运行批量生成

```bash
hermes batch-run --dataset dataset.jsonl --output trajectories.jsonl --workers 4
```

### 6.3 压缩训练轨迹

```bash
hermes compress-trajectories --input trajectories.jsonl --output compressed.jsonl --target-tokens 15250
```

---

## 七、常见问题

### 如何查看日志？

```bash
# 实时查看
tail -f ~/.hermes/logs/hermes.log

# 或通过 doctor 检查
hermes doctor
```

### 如何更新？

```bash
# git 安装
hermes update

# pip 安装
pip install --upgrade hermes-agent
hermes postinstall
```

### 上下文窗口不足怎么办？

Hermes 会自动触发压缩（上下文字符数 > 50% 窗口时）。也可以手动触发：

```bash
/compress
```

### 如何在 Docker 中运行？

```bash
# 使用 Docker 后端
hermes --sandbox docker

# 或在 config.yaml 中设置 terminal.backend: docker
# 然后正常运行
hermes
```

### Windows 下的限制？

Windows 原生支持所有核心功能。基于浏览器的仪表板聊天面板仅限 WSL2（依赖 POSIX PTY）。

---

## 八、完整命令行参考

```bash
# 基础
hermes                        # 启动经典 CLI
hermes --tui                  # 启动现代 TUI
hermes chat -q "问题"          # 单次查询
hermes -c                     # 恢复最近会话
hermes -w                     # Worktree 隔离模式
hermes -s skill1,skill2      # 预加载技能

# 安全
hermes --yolo                 # YOLO 模式

# 会话
hermes sessions list          # 列出会话
hermes sessions resume <id>   # 恢复会话

# 记忆
hermes memory list            # 列出记忆后端
hermes memory setup <name>    # 配置外部后端

# 网关
hermes gateway                # 启动消息网关

# 批量
hermes batch-run ...          # 批量生成训练数据

# 维护
hermes doctor                 # 系统诊断
hermes update                 # 更新
hermes postinstall            # 安装后配置
```

---

> 更多信息请查看 [Hermes Agent 官网](https://hermes-agent.nousresearch.com/docs/zh-Hans)
