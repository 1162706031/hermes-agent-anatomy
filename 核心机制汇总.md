# 09 Hermes 核心机制汇总

> 本文汇总 Hermes Agent 四大核心机制：Agent 核心循环、上下文压缩、Memory 系统、沙箱安全。适合快速建立对 Hermes 技术架构的整体认知。

---

## 一、Agent 核心循环

### 1.1 架构概览

核心引擎是 `run_agent.py` 中的 **AIAgent** 类（~10,500 行），采用**同步 while 循环 + 回调**的架构风格。

```
用户输入 → 构建 system prompt → 调 LLM API
                                    ↓
                              模型返回响应
                                    ↓
                        ┌─── tool_use? ───┐
                        ↓ Yes             ↓ No
                   权限检查            输出结果
                        ↓              结束循环
                   执行工具
                        ↓
                   结果追加到消息历史
                        ↓
                   压缩检查 → 回到调 API
```

### 1.2 单轮生命周期

1. 生成 `task_id`
2. 将用户消息追加到对话历史
3. 通过 `prompt_builder.py` 构建或复用缓存的系统提示
4. 检查预压缩（上下文超过窗口 50% 时触发）
5. 构建 API 消息，按模式适配格式
6. 注入临时提示层（预算警告、上下文压力提示）
7. 应用 Anthropic prompt caching
8. 发起可中断的流式 API 调用
9. 解析响应：若有 tool_calls → 执行工具 → 循环；若为文本响应 → 持久化 → 返回

### 1.3 三种 API 模式

| 模式 | 用途 | 客户端 |
|------|------|--------|
| `chat_completions` | OpenAI 兼容端点 | `openai.OpenAI` |
| `codex_responses` | OpenAI Codex / Responses API | `openai.OpenAI` |
| `anthropic_messages` | 原生 Anthropic Messages API | `anthropic.Anthropic` |

三种模式收敛到相同的内部消息格式（OpenAI 风格的 role/content/tool_calls 字典）。

### 1.4 IterationBudget：油量表机制

```python
class IterationBudget:
    max_total: int        # 默认 90
    _used: int            # 线程安全计数器
    consume() -> bool     # 消费一次迭代
    refund() -> None      # 轻量级 RPC 调用免费
```

两级预算预警系统：
- **70%**：轻推 —— "开始收尾"
- **90%**：最后通牒 —— "立刻给出最终回答"

警告注入到最后一条工具结果中，旧警告每轮清除，防止模型在后续 turn 里持续受干扰。

### 1.5 并行工具执行

**三阶段安全检查：**

1. **黑名单拦截**：`clarify` 等交互式工具，出现在批次中则整批降级为串行
2. **白名单放行**：`read_file`、`web_search` 等只读工具无条件并行
3. **路径冲突检测**：`read_file`、`write_file`、`patch` 只有目标路径不重叠时才能并行

```python
_NEVER_PARALLEL_TOOLS = frozenset({"clarify"})
_PARALLEL_SAFE_TOOLS = frozenset({
    "read_file", "search_files", "session_search",
    "web_extract", "web_search", "vision_analyze", ...
})
_MAX_TOOL_WORKERS = 8
```

通过安全检查后走 `ThreadPoolExecutor`，最多 8 个 worker，结果按原始顺序收集。

### 1.6 工具结果三级防御体系

防止单个工具结果撑爆上下文窗口：

| 层级 | 机制 | 说明 |
|------|------|------|
| **第 1 级** | 工具自限 | `search_files` 等工具预先截断输出 |
| **第 2 级** | 单结果落盘 | 超阈值的写入 `/tmp/hermes-results/`，替换为 preview + 文件路径 |
| **第 3 级** | 整轮预算 | 本轮所有结果 > 200K 字符，最大的强制落盘 |

### 1.7 消息清洗

每轮 API 调用前的关键清洗操作：

| 操作 | 触发场景 | 处理方式 |
|------|----------|----------|
| Surrogate 消毒 | 富文本粘贴 | 替换为 U+FFFD |
| Budget Warning 剥离 | 每轮开头 | JSON del / 正则移除 |
| 孤儿 tool_call 修复 | 压缩/手动编辑后 | 删孤儿 result / 补 stub |
| reasoning 字段转换 | 多轮推理上下文 | reasoning → reasoning_content |

### 1.8 错误重试与 Fallback

**Jittered Backoff**：用 `time_ns ^ (tick * 0x9E3779B9)` 做独立种子，避免多个 Gateway 会话同步重试产生雷同抖动值。

**Fallback 链**：主 provider → fallback[0] → fallback[1] → ... 逐级降级。每个新 turn 恢复主 provider。

**关键错误处理**：

| 错误类型 | 处理 |
|----------|------|
| 429 Rate Limit | jittered_backoff → fallback |
| 上下文超限 | 压缩 → 降级上下文窗口探测 |
| 工具名幻觉 | 自动修复 → 注入错误消息让模型自纠 |
| 流式连接僵死 | 90s stale-stream 检测 + 60s 读超时 |

### 1.9 始终流式 (Always Streaming)

Hermes 始终使用 streaming 路径，即使没有流式消费者。非流式调用存在隐蔽问题：provider 用 SSE keep-alive ping 保持连接但不返回数据，Agent 无限挂住。流式路径自带 90 秒健康检查和 60 秒读超时。

### 1.10 与 Claude Code 循环架构对比

| 维度 | Hermes Agent (Python) | Claude Code (TypeScript) |
|------|----------------------|--------------------------|
| 核心循环 | 同步 `while` + callback | `async function*` yield |
| 并发模型 | `ThreadPoolExecutor` | 原生 `Promise.all` |
| 中断机制 | `_interrupt_requested` 标志位 | AbortController signal |
| 流式传播 | 8 个 callback 插槽 | yield 到上层 for-await-of |
| 文件组织 | 10,500 行单文件 | 分散在多个模块 |

---

## 二、上下文压缩

### 2.1 压缩触发策略

| 触发条件 | 阈值 | 时机 |
|----------|------|------|
| **预压缩** | 上下文超过窗口 50% | API 调用前 |
| **网关自动压缩** | 上下文超过 85% | 轮次之间 |

50% 触发看似激进，但配合迭代摘要策略（早压缩、多次压缩、每次增量更新），单次被压缩的内容量更小，摘要质量反而更高。

### 2.2 压缩流程

```
Step 0: Memory Flush（防止压缩丢失信息）
    ↓
Step 1: 裁剪工具输出（>200 字符的旧 tool result → 占位符，零成本）
    ↓
Step 2: 确定压缩边界
    head_end = protect_first_n (3 条，对齐到非 tool 消息)
    tail_start = _find_tail_cut_by_tokens (≥20 条 或 ≥20K tokens)
    ↓
Step 3: LLM 结构化摘要
    首次：从零生成七段式结构化摘要
    后续：在 _previous_summary 基础上增量更新
    预算：max(2000, min(内容量×20%, 窗口×5%))
    ↓
Step 4: 组装压缩后消息
    HEAD (3条) + SUMMARY (1条) + TAIL (~20K tokens)
    _sanitize_tool_pairs 修复孤儿配对
    ↓
Step 5: Session 分裂
    旧 session → end_reason="compression"
    新 session → parent_session_id = 旧 session
```

### 2.3 头尾保护策略

- **头部保护**：固定 3 条消息（system prompt、第一条用户消息、第一条 assistant 回复）
- **尾部保护**：Token 预算制（默认 20K tokens），同时保留 `protect_last_n=20` 作为保底下限

这种双重保护兼顾两种情况：短消息场景能保留更多上下文，长 tool result 场景能兜底保护最近 20 条。

### 2.4 七段式结构化摘要模板

```
## Goal
[用户在做什么]

## Constraints & Preferences
[用户偏好、编码风格、约束条件]

## Progress
### Done
[完成了什么 — 文件路径、执行命令、具体结果]
### In Progress
[正在做什么]
### Blocked
[卡在哪里]

## Key Decisions
[技术决策和理由]

## Relevant Files
[读过/改过/创建的文件列表]

## Next Steps
[下一步该做什么]

## Critical Context
[不能丢的具体值：错误信息、配置参数、数据]
```

结构化模板强制 LLM 按类别填充，避免自由格式摘要中关键信息被选择性丢弃。第二次压缩时不从零开始，而是在前一次摘要基础上增量更新。

### 2.5 Tool Call/Result 配对完整性

压缩不破坏消息格式的核心保障：

- `_sanitize_tool_pairs`：收集存活 tool_call ID → 删孤儿 result → 补缺失 stub
- `_align_boundary_forward`：起始边界跳过连续 tool result
- `_align_boundary_backward`：结束边界将 assistant + tool_results 整组拉进压缩区

### 2.6 Session 分裂链

```
Session_A (original)
  └─ end_reason: "compression"
      └─ Session_B (parent=A)
          └─ end_reason: "compression"
              └─ Session_C (parent=B) ← 当前活跃
```

三个好处：历史可追溯（session_search）、标题自动继承编号、成本归集可沿链聚合。

### 2.7 与 Claude Code / OpenClaw 压缩策略对比

| 维度 | Hermes | Claude Code | OpenClaw |
|------|--------|-------------|----------|
| 压缩方式 | LLM 结构化摘要 | 三层递进（微/中/全量）压缩 | 固定窗口裁剪 |
| 触发阈值 | 50% | 80%/90%/95% | 上下文接近上限 |
| 摘要迭代 | 增量更新 previous_summary | 每次从零生成 | 无 |
| Session 管理 | parent_session_id 链式 | 无 | 无 |
| 信息保留 | 摘要保留关键信息 | 分层保留 | 被裁剪的丢失 |
| 计算成本 | 每次 LLM 调用 | 多级 LLM 调用 | 零额外成本 |

---

## 三、Memory 系统

### 3.1 内置双文件记忆

| 文件 | 用途 | 字符限制 | 估计 Token |
|------|------|----------|-----------|
| **MEMORY.md** | Agent 个人笔记 — 环境事实、约定、经验 | 2,200 字符 | ~800 |
| **USER.md** | 用户档案 — 偏好、沟通风格、期望 | 1,375 字符 | ~500 |

**设计要点：**
- 会话开始时以**冻结快照**形式注入系统提示词，保证 prompt cache 稳定
- 写操作立即持久化到磁盘，但直到下次会话启动才出现在提示中
- `memory` 工具支持三种操作：`add`、`replace`（子字符串匹配）、`remove`
- 完全重复的条目静默拒绝
- 写入前扫描：提示注入、凭据泄露、SSH 后门、不可见 Unicode 字符

### 3.2 MemoryManager：插件化架构

```python
class MemoryManager:
    _providers: List[MemoryProvider]     # builtin 永远在第一位
    _tool_to_provider: Dict[str, ...]    # O(1) 路由
    _has_external: bool                  # 外部 provider 最多 1 个
```

**完整生命周期钩子：**

| 钩子 | 时机 | 用途 |
|------|------|------|
| `initialize` | Session 开始 | 连接后端、注入 hermes_home |
| `system_prompt_block` | System prompt 构建 | 静态记忆注入 |
| `on_turn_start` | 每轮开始 | 计数、scope 管理 |
| `prefetch` | API 调用前 | 基于当前 query 召回记忆 |
| `sync_turn` | 每轮完成后 | 异步写入对话记录 |
| `on_memory_write` | 内置记忆写入时 | 镜像到外部后端（跳过 builtin 自身）|
| `on_pre_compress` | 压缩前 | 从即将丢弃的消息中提取洞察 |
| `on_session_end` | 会话结束 | 端到端事实抽取 |
| `shutdown` | 关闭 | 反序关闭（后注册的先关）|

关键设计：每个钩子有 try-except 包裹，一个 provider 崩溃不拖垮另一个。`on_memory_write` 跳过 builtin 自身，防止写入循环。`on_pre_compress` 在压缩前抢救即将丢失的知识。

### 3.3 会话搜索 (session_search)

与持久化记忆互补的独立系统：

| 特性 | 持久化记忆 | 会话搜索 |
|------|-----------|---------|
| 容量 | ~1,300 tokens | 无限制 |
| 速度 | 即时（已在系统提示中）| ~20ms FTS5 |
| 成本 | 每次提示有 token 开销 | 免费（无 LLM 调用）|
| 使用场景 | 始终可用的关键事实 | 查找特定历史对话 |

基于 SQLite FTS5 全文搜索，直接从数据库返回原始消息，无需 LLM 摘要。

### 3.4 SQLite 持久化：WAL + 抖动重试

```python
class SessionDB:
    _WRITE_MAX_RETRIES = 15
    _WRITE_RETRY_MIN_S = 0.020    # 20ms
    _WRITE_RETRY_MAX_S = 0.150    # 150ms
    _CHECKPOINT_EVERY_N_WRITES = 50
```

- **WAL 模式**：多 reader + 单 writer 并发，读写不互斥
- **抖动重试**：20-150ms 随机抖动打破 convoy effect
- **BEGIN IMMEDIATE**：事务开始时抢写锁，冲突第一时间暴露
- **PASSIVE checkpoint**：每 50 次写入触发，不阻塞读取

### 3.5 8 种外部 Memory Provider

| Provider | 特点 | 部署 | 费用 |
|----------|------|------|------|
| **Honcho** | 辩证推理的用户建模，5 工具 | 云/自托管 | 付费/免费 |
| **OpenViking** | 字节跳动，文件系统层级上下文数据库 | 自托管 | 免费，AGPL-3.0 |
| **Mem0** | 服务端 LLM 事实提取，语义搜索+重排序 | 云 | 付费 |
| **Hindsight** | 知识图谱+实体解析+多策略检索 | 云/本地 | 免费/付费 |
| **Holographic** | 本地 SQLite+FTS5+HRR 代数，零依赖 | 本地 | 免费 |
| **RetainDB** | 混合搜索（向量+BM25+重排序），7 种记忆类型 | 云 | $20/月 |
| **ByteRover** | 分层知识树，层级检索 | 本地/云 | 免费/付费 |
| **Supermemory** | 语义长期记忆+用户档案+会话图导入 | 云 | 付费 |

内置系统与外部 provider **并行运行**（非替代），内置记忆全量注入 + 外部 provider 按需召回。

### 3.6 四层记忆体系

1. **声明性记忆**（MEMORY.md / USER.md）：Agent 知道什么事实
2. **程序性记忆**（SKILL.md）：Agent 会做什么方法论
3. **会话记忆**（SQLite + FTS5）：Agent 做过什么历史
4. **训练记忆**（batch_runner + trajectory_compressor）：Agent 如何变得更好

四层协同构成从运行时能力到进化能力的完整闭环。

---

## 四、沙箱安全机制

### 4.1 七层纵深防御

```
┌──────────────────────────────────────────────┐
│  第 1 层：危险命令审批                          │
│  20+ 模式匹配，三档策略（manual/smart/off）      │
│  硬阻止列表永不过滤                            │
├──────────────────────────────────────────────┤
│  第 2 层：用户授权（Gateway）                    │
│  分层检查 → DM 配对码（8 字符，1h 过期）         │
├──────────────────────────────────────────────┤
│  第 3 层：容器隔离                              │
│  Docker: --cap-drop ALL, no-new-privileges    │
│  资源限制: CPU/内存(5GB)/磁盘(50GB)             │
├──────────────────────────────────────────────┤
│  第 4 层：环境变量过滤                           │
│  剥离 KEY/TOKEN/SECRET/PASSWORD 等敏感变量      │
├──────────────────────────────────────────────┤
│  第 5 层：MCP 凭证处理 + SSRF 防护              │
│  工具结果中凭证模式替换为 [REDACTED]             │
│  SSRF: 阻止私有网络、回环、云元数据 IP             │
├──────────────────────────────────────────────┤
│  第 6 层：跨会话隔离 + 输入净化                   │
│  会话间数据不可访问，Cron 路径遍历保护            │
├──────────────────────────────────────────────┤
│  第 7 层：Tirith 预执行扫描                      │
│  同形异义词 URL 检测、管道注入检测               │
└──────────────────────────────────────────────┘
```

### 4.2 第 1 层：危险命令审批

**三种模式：**

| 模式 | 行为 |
|------|------|
| `manual`（默认）| 始终提示用户确认 |
| `smart` | 辅助 LLM 评估风险：低风险自动批准，明确危险自动拒绝，不确定的升级 |
| `off` | 禁用所有检查（等同 `--yolo`）|

**硬阻止列表（始终强制执行）：**
- `rm -rf /`
- 分叉炸弹
- 针对已挂载根设备的 `mkfs.*`
- `dd if=/dev/zero of=/dev/sd*`
- 在文件系统根目录将不信任 URL 通过管道传给 `sh`

**容器绕过：** 在 Docker、Singularity、Modal、Daytona 后端运行时，危险命令检查被跳过（容器本身就是安全边界）。

### 4.3 第 2 层：Gateway 用户授权

- **DM 配对码系统**：8 字符加密随机码（32 字符无歧义字母表，排除 0/O/1/I）
- 代码 1 小时后过期，速率限制：每用户每 10 分钟 1 次请求
- 5 次失败尝试后锁定 1 小时
- 配对数据文件权限 `chmod 0600`
- 代码永不记录到标准输出

### 4.4 第 3 层：容器隔离

```bash
# Docker 安全强化
--cap-drop ALL
--cap-add DAC_OVERRIDE,CHOWN,FOWNER
--security-opt no-new-privileges
--pids-limit 256
# tmpfs 挂载（noexec/nosuid）
```

可配置资源限制：CPU 核心数、内存（默认 5GB）、磁盘（默认 50GB）。

### 4.5 第 4 层：环境变量过滤

`execute_code` 和 `terminal` 从子进程中剥离敏感环境变量：
- 技能范围：SKILL.md 中声明的变量自动注册
- 基于配置：添加到 `terminal.env_passthrough`
- 过滤规则：阻止包含 KEY/TOKEN/SECRET/PASSWORD/CREDENTIAL/PASSWD/AUTH 的变量
- Docker 默认不传递任何环境变量

### 4.6 第 5 层：MCP 凭证 + SSRF

**凭证脱敏**：工具错误消息中的 `ghp_...`、`sk-...`、bearer token、`token=`、`key=`、`API_KEY=`、`password=`、`secret=` 等模式替换为 `[REDACTED]`。

**SSRF 防护**：
- 阻止私有网络（RFC 1918）、回环地址
- 阻止链路本地地址（含 `169.254.169.254`）
- 阻止 CGNAT（`100.64.0.0/10`）
- DNS 失败视为阻止（失败关闭）
- 重定向链每一跳重新验证
- 可配置白名单：`security.allow_private_urls: true`

### 4.7 第 6-7 层：会话隔离 + Tirith 扫描

**跨会话隔离**：会话无法访问彼此数据，Cron 任务路径经过路径遍历保护。

**上下文文件注入保护**：AGENTS.md、.cursorrules、SOUL.md 加载前扫描：
- 忽略/覆盖先前指令的指示
- 隐藏 HTML 注释中的可疑关键词
- 读取机密文件的尝试
- 通过 curl 泄露凭据
- 不可见 Unicode 字符

**Tirith 集成**：同形异义词 URL 欺骗检测、管道到解释器模式检测、终端注入检测。

### 4.8 终端后端安全对比

| 后端 | 隔离级别 | 危险命令检查 | 适用场景 |
|------|---------|-------------|---------|
| local | 无（主机）| 是 | 开发、可信用户 |
| ssh | 远程机器 | 是 | 分离服务器 |
| docker | 容器 | 跳过 | 生产 Gateway |
| singularity | 容器 | 跳过 | HPC 环境 |
| modal | 云沙箱 | 跳过 | 可扩展云隔离 |
| daytona | 云沙箱 | 跳过 | 持久云工作区 |

### 4.9 生产部署清单

1. 设置显式 allowlist（生产环境禁 `GATEWAY_ALLOW_ALL_USERS=true`）
2. 使用容器后端（`terminal.backend: docker`）
3. 限制资源上限（CPU、内存、磁盘）
4. 安全存储密钥（`~/.hermes/.env`，正确文件权限）
5. 启用 DM 配对
6. 审计命令允许列表
7. 设置 `MESSAGING_CWD` 防止在敏感目录操作
8. 以非 root 用户运行 Gateway
9. 监控日志 `~/.hermes/logs/`
10. 定期 `hermes update`

---

## 关键数据速查

| 指标 | 数值 |
|------|------|
| Agent 循环上限 | 90 次迭代（子 Agent 50 次）|
| 压缩阈值 | 50% 上下文窗口 |
| 尾部保护预算 | 20K tokens（≥20 条保底）|
| 摘要输出上限 | min(窗口×5%, 12000) |
| Tool worker 上限 | 8 |
| MEMORY.md 限制 | 2,200 字符 |
| USER.md 限制 | 1,375 字符 |
| WAL checkpoint | 每 50 次写入 |
| 写冲突退避 | 20-150ms 随机 |
| DM 配对码过期 | 1 小时 |
| 容器内存限制 | 默认 5GB |
| 容器磁盘限制 | 默认 50GB |
