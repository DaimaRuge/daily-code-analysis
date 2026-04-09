# Hermes Agent 技术架构与源码研读报告

> **项目**: [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
> **分析日期**: 2026-04-10
> **GitHub Stars**: 43,619+（今日新增 6,788 stars，GitHub Trending 第一名）
> **技术栈**: Python 3.11+ / uv 包管理 / prompt_toolkit TUI / Rich 渲染 / SQLite(FTS5)
> **定位**: 自改进型 AI Agent（self-improving AI agent）——"与你共同成长的 AI 助手"
> **许可证**: MIT
> **Built by**: [Nous Research](https://nousresearch.com)

---

## 一、项目概述

Hermes Agent 是由 **Nous Research** 构建的自改进型 AI Agent，被称为"唯一内置学习循环的 Agent"。它能从经验中创建 Skills、在使用中自我改进、自主记忆知识、跨会话搜索历史，并通过 Honcho 实现用户画像建模。与 OpenClaw 高度竞争，拥有从 OpenClaw 一键迁移的能力。

**核心差异化定位**：
- **自改进学习循环**：唯一内置"闭合学习回路"的 Agent——自动创建 Skills、自动完善、记忆持久化
- **全平台统一入口**：CLI + Telegram + Discord + Slack + WhatsApp + Signal + Email + Home Assistant + DingTalk + Feishu + WeCom，从任意平台驱动同一 Agent
- **零成本待机**：Daytona/Modal 终端后端支持 Serverless 持久化，闲置时休眠、按需唤醒，$5 VPS 即可运行
- **任意模型切换**：`hermes model` 一键切换 OpenRouter（200+模型）、Nous Portal、MiniMax、Kimi、OpenAI、Anthropic 或自定义端点
- **ACP 协议集成**：原生支持 Agent Client Protocol，可接入 VS Code、Zed、JetBrains 等编辑器

**与 OpenClaw 的关系**：
- 功能高度重叠（CLI Agent、多平台消息、多工具集、本地记忆系统）
- Hermes 明确提供 `hermes claw migrate` 从 OpenClaw 迁移（SOUL.md、记忆、Skills、API Keys）
- 是 OpenClaw 当前最直接的商业竞品

---

## 二、架构设计总览

### 2.1 整体架构哲学

Hermes 遵循**平台解耦 + 协议驱动**的设计哲学：核心 Agent 逻辑与消息通道完全分离，Gateway 层负责所有外部平台的接入，核心只关心 API 调用和工具执行。这种设计使其能以相同的行为运行在 $5 VPS（SSH 后台）、服务器（Docker）、Serverless（Modal）和本地 CLI 等完全不同环境中。

### 2.2 模块层次结构

```
┌─────────────────────────────────────────────────────────┐
│                    用户交互层                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  CLI TUI │  │ Telegram │  │ Discord  │  │ Slack  │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬────┘ │
│       │             │             │            │        │
│  ┌────┴─────────────┴─────────────┴────────────┴────┐ │
│  │              Gateway Runner (gateway/run.py)        │ │
│  │   统一消息路由 / Slash 命令分发 / 会话管理 / 投递   │ │
│  └─────────────────────┬──────────────────────────────┘ │
└────────────────────────┼────────────────────────────────┘
                         │ run_conversation()
┌────────────────────────┼────────────────────────────────┐
│                 核心 Agent 层                            │
│  ┌────────────────────────────────────────────────────┐ │
│  │            run_agent.py (AIAgent, ~9722行)         │ │
│  │  对话循环 / 消息历史 / API 调用 / 工具调度 / 统计   │ │
│  └─────────────────────┬──────────────────────────────┘ │
│           ┌────────────┼────────────────┐               │
│  ┌────────┴───┐  ┌─────┴────┐  ┌──────┴──────┐        │
│  │model_tools │  │ toolsets  │  │  tools/registry│      │
│  │ (工具调度)  │  │ (工具集)   │  │  (中央注册表)  │      │
│  └────────────┘  └──────────┘  └──────────────┘        │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐│
│  │              agent/ (共 15551 行)                    ││
│  │  prompt_builder / memory_manager / context_compressor││
│  │  anthropic_adapter / auxiliary_client / error_classifier│
│  │  credential_pool / models_dev / display             ││
│  └─────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
                         │
┌────────────────────────┼────────────────────────────────┐
│                    工具层 (~38860行)                      │
│  ┌─────────────┐  ┌────────────┐  ┌─────────────────┐  │
│  │ terminal_   │  │ browser_   │  │   skills_tool   │  │
│  │ tool (1757) │  │ tool(2182) │  │     (1376)      │  │
│  ├─────────────┤  ├────────────┤  ├─────────────────┤  │
│  │  web_tools  │  │ mcp_tool   │  │ code_execution  │  │
│  │  (2101)     │  │ (2186)     │  │    _tool(1347)  │  │
│  ├─────────────┤  ├────────────┤  ├─────────────────┤  │
│  │delegate_tool│  │ skills_hub │  │ rl_training_tool│  │
│  │  (子Agent)  │  │ (2707)     │  │    (1396)       │  │
│  └─────────────┘  └────────────┘  └─────────────────┘  │
└──────────────────────────────────────────────────────────┘
                         │
┌────────────────────────┼────────────────────────────────┐
│               基础设施层                                  │
│  hermes_state.py  (SQLite/FTS5 会话持久化)              │
│  cron/scheduler.py  (定时任务调度)                       │
│  acp_adapter/        (VS Code/Zed/JetBrains 集成)         │
│  environments/       (RL 训练环境 / SWE-bench)            │
│  batch_runner.py     (批量轨迹生成)                       │
└──────────────────────────────────────────────────────────┘
```

---

## 三、核心模块深度分析

### 3.1 工具系统（Tool System）

#### 3.1.1 中央注册表 (`tools/registry.py`)

Hermes 的工具系统以**注册表模式**为核心，所有工具文件在模块导入时调用 `registry.register()` 向中央注册表声明自己。这一设计彻底解耦了工具定义与工具调用：

```python
# 工具注册入口（来自任意工具文件）
registry.register(
    name="example_tool",
    toolset="example",
    schema={"name": "example_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: example_tool(...),
    check_fn=check_requirements,      # 运行时可用性检查
    requires_env=["EXAMPLE_API_KEY"],
)
```

注册表收集所有 schema 后统一提供给 `model_tools.py` 生成工具定义，**无需维护平行数据结构**。动态 deregister 支持 MCP 的 `notifications/tools/list_changed` 事件——MCP 服务器工具变更时清空并重建对应工具。

#### 3.1.2 工具集（Toolsets）(`toolsets.py`)

`_HERMES_CORE_TOOLS` 是跨平台共享工具列表，Edit once 即全局生效：

```python
_HERMES_CORE_TOOLS = [
    "web_search", "web_extract",           # Web
    "terminal", "process",                  # 终端
    "read_file", "write_file", "patch", "search_files",  # 文件
    "vision_analyze", "image_generate",     # 视觉/图像生成
    "skills_list", "skill_view", "skill_manage",  # Skills
    "browser_navigate", "browser_snapshot",  # 浏览器
    "text_to_speech",                       # TTS
    "todo", "memory",                       # 规划/记忆
    "session_search",                       # 会话历史搜索
    "clarify",                              # 用户澄清
    "execute_code", "delegate_task",        # 代码执行/子Agent
    "cronjob",                              # 定时任务
    "send_message",                          # 跨平台消息
    "ha_*",                                 # HomeAssistant 控制
]
```

TOOLSETS 字典支持**组合式工具集**：一个 toolset 可包含直接工具列表（`tools`）和所依赖的其他工具集（`includes`），最终通过 `resolve_toolset()` 递归解析为完整工具列表。内置工具集包括：`web`、`search`、`vision`、`image_gen`、`terminal`、`moa`、`skills`、`browser`、`safe`、`debugging`、`full_stack` 等。

#### 3.1.3 工具调用分发 (`model_tools.py`)

```python
# 模型返回 tool_calls → handle_function_call()
response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
for tool_call in response.tool_calls:
    result = handle_function_call(tool_call.name, tool_call.args, task_id)
    messages.append(tool_result_message(result))
```

`_discover_tools()` 导入所有工具模块触发注册，然后查询注册表获取完整 schema 列表。`get_toolset_for_tool()` 支持按工具集过滤工具定义。

### 3.2 Agent 核心循环 (`run_agent.py`，~9722行)

`AIAgent` 是整个系统的核心 orchestrator，采用**同步轮询 + 线程池**架构：

**主循环**：
```python
while api_call_count < self.max_iterations:
    response = client.chat.completions.create(
        model=model, messages=messages, tools=tool_schemas
    )
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content  # 无工具调用，直接返回文本
```

**关键设计决策**：
- **消息格式**：统一 OpenAI 格式 `{"role": "system/user/assistant/tool", ...}`，reasoning content 存在 `assistant_msg["reasoning"]`
- **轮次预算**：`iteration_budget` 跟踪剩余迭代次数，防止无限循环
- **工具错误隔离**：每个工具调用错误被捕获并格式化为错误消息，不打断主循环
- **`_SafeWriter`**：透明包装 stdio，捕获 `OSError`/`ValueError`（pipe broken），确保 systemd/Docker 环境下不崩溃
- **`_isolate_hermes_home`** fixture：测试时重定向 `HERMES_HOME` 到临时目录，确保测试不污染用户数据

### 3.3 Agent 内核模块群 (`agent/` 目录，共 15551 行)

#### 3.3.1 提示词构建 (`prompt_builder.py`，990行)

系统提示词的组装工厂，职责包括：
- **身份定义**：`DEFAULT_AGENT_IDENTITY` 提供 Agent 人设
- **平台提示**：`PLATFORM_HINTS` 根据运行平台（CLI/Telegram/Discord）注入差异化指令
- **记忆块**：`build_memory_context_block()` 将 prefetch 的记忆内容包装在 `<memory-context>` 标签内，防止模型将其当作用户输入
- **上下文威胁扫描**：`_scan_context_content()` 在加载 `AGENTS.md`、`.cursorrules`、`SOUL.md` 前检测 prompt injection 特征（隐藏 Unicode、绕过指令、Secret exfil 模式等）
- **Skills 系统提示**：`build_skills_system_prompt()` 扫描 skills 目录，注入可用技能列表及触发条件

#### 3.3.2 上下文压缩 (`context_compressor.py`，745行)

自动上下文压缩机制，在上下文接近 token 上限时触发：
- 使用 LLM 将历史消息压缩为摘要
- 保留关键工具调用结果和决策点
- 支持多种压缩策略（最近优先、重要性优先）

#### 3.3.3 上下文缓存 (`prompt_caching.py`)

针对 Anthropic API 的 prompt caching 优化：
- `apply_anthropic_cache_control()` 标记可缓存的引用内容
- 跟踪缓存有效性，禁止在对话中途变更历史或重新加载记忆（防止 cache break）

#### 3.3.4 凭证池 (`credential_pool.py`，1211行)

多提供商凭证管理系统：
- 同时配置多个 provider 的 API keys
- `credential_pool.py` 负责动态路由时选择合适凭证
- 支持 API key 轮换、失败转移（failover）

#### 3.3.5 错误分类与故障转移 (`error_classifier.py`，792行)

```python
# classify_api_error() 解析 API 错误类型
# FailoverReason: rate_limit → 切换模型 / context_too_large → 压缩上下文 / auth_error → 换凭证
```

#### 3.3.6 智能模型路由 (`smart_model_routing.py`)

根据任务复杂度自动选择模型：
- 简单任务 → 小模型（低成本）
- 复杂任务 → 大模型（高质量）
- 失败历史 → 动态降级/升级策略

#### 3.3.7 Anthropic 适配器 (`anthropic_adapter.py`，1429行)

处理与 Anthropic API 的深度集成，包括：
- 缓存提示词块（Caching Prompt Blocks）支持
- 推理内容（Reasoning Content）提取
- 视觉/音频多模态消息处理
- 原生流式响应处理

### 3.4 记忆与学习系统

#### 3.4.1 记忆管理器 (`agent/memory_manager.py`)

核心抽象：**MemoryManager** 协调 BuiltinMemoryProvider 和至多一个外部内存 provider：

```python
class MemoryManager:
    def add_provider(provider)        # 注册 provider（builtin 优先，外置唯一）
    def build_system_prompt() -> str   # 收集所有 provider 的系统提示块
    def prefetch_all(query) -> str     # 收集所有 provider 的召回上下文
    def sync_all(user_msg, assistant)  # 工具调用后同步记忆
    def queue_prefetch_all(query)       # 队列预取下一轮所需记忆
```

**关键约束**：外部 provider 只能有一个，防止工具 schema 膨胀和记忆后端冲突。

#### 3.4.2 内置记忆 Provider (`agent/builtin_memory_provider.py`)

适配原有 Hermes 记忆系统（`MEMORY.md` + `USER.md`）：
- 系统提示块通过 `format_for_system_prompt()` 生成
- 记忆内容通过 `<memory-context>` 围栏隔离，防止被模型误认为用户输入
- 工具调用通过 `run_agent.py` 的 agent 级别拦截处理（不走标准注册表）

#### 3.4.3 Skills 自改进系统 (`agent/skill_utils.py`，442行)

Skills 是 Hermes 的**自改进机制核心**：
- 每个 Skill 是 Markdown 文件，含 YAML frontmatter（描述、条件、平台过滤）
- **自动创建**：复杂任务完成后，Agent 自动生成新 Skill 存入 `~/.hermes/skills/`
- **自完善**：Skill 在后续使用中被 Agent 主动改进（`skill_manage` 工具）
- **平台过滤**：`skill_matches_platform()` 根据 frontmatter 的 `platforms` 字段决定是否激活
- **条件匹配**：`extract_skill_conditions()` 支持 frontmatter 中的触发条件

#### 3.4.4 Skills Hub (`tools/skills_hub.py`，2707行)

社区 Skills 市场集成：
- 从 agentskills.io 索引社区 Skills
- 搜索、安装、更新、版本管理
- 与 OpenClaw ClawHub 功能完全对标

### 3.5 子 Agent 委托系统 (`tools/delegate_tool.py`)

并行多工作流的核心：

```python
# 委托给独立子 Agent（隔离上下文 + 独立终端会话）
spawn_child(
    goal="修复这个 bug",
    context=codebase_context,
    toolset=["terminal", "file"],  # 受限工具集
    max_iterations=50
)
```

**安全设计**：
- `DELEGATE_BLOCKED_TOOLS`：递归委托、用户交互、共享记忆写入、跨平台消息、代码执行脚本 — 永远禁止
- `_run_single_child()` 保存/恢复 `_last_resolved_tool_names` 全局状态，防止父 agent 上下文泄漏到子 agent
- **深度限制**：`MAX_DEPTH=2`（父→子→孙，拒绝第三代）
- **并发限制**：`MAX_CONCURRENT_CHILDREN=3`，超过则排队

### 3.6 Gateway 消息网关 (`gateway/`)

#### 3.6.1 架构设计

Gateway 是 Hermes 的**多平台消息接入层**，所有外部平台通过统一接口与核心 Agent 通信：

```
┌─────────────────────────────────────────┐
│  gateway/run.py (GatewayRunner)          │
│  统一消息路由 / Slash命令分发 / 会话管理   │
└──────┬──────────────────────────────────┘
       │
┌──────┼──────────────────────────────────┐
│  platform adapters (并发独立)            │
│  telegram.py / discord.py / slack.py    │
│  whatsapp.py / signal.py / email.py      │
│  homeassistant.py / dingtalk.py /        │
│  feishu.py / wecom.py / bluebubbles.py  │
└──────────────────────────────────────────┘
```

#### 3.6.2 会话持久化 (`hermes_state.py`)

- **SQLite + FTS5**：全文搜索会话历史，支持跨会话上下文召回
- **SessionDB**：会话存储抽象，按 session_id 隔离
- 会话可在任意平台恢复，保证跨平台连续性

#### 3.6.3 Slash 命令注册表

所有 slash 命令定义在 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY`，自动分发到：
- CLI TUI
- Gateway（hook 机制）
- Telegram Bot 菜单
- Slack 子命令路由
- 自动补全

### 3.7 ACP 协议适配器 (`acp_adapter/`)

**Agent Client Protocol** 是 VS Code、Zed、JetBrains 等编辑器的 Agent 集成标准：

```python
# ACP Server 暴露的能力
AgentCapabilities(
    streaming=True,
    notifications=True,          # 工具调用进度通知
    tools=True,
    sessions=True,               # 多会话管理
    fork=True,                  # 会话分叉
)
```

- `SessionManager`：管理多个并发的 ACP 会话
- `make_tool_progress_cb()`：实时推送工具执行进度
- `make_thinking_cb()`：推送推理过程（流式）
- `make_message_cb()`：消息事件回调
- 支持 VS Code 的 "Continue" 插件、Zed AI 面板、JetBrains Toolbox

### 3.8 Cron 调度系统 (`cron/`)

```python
# 定时任务支持任意平台投递
job = {
    "name": "daily-report",
    "schedule": "0 9 * * *",          # cron 表达式
    "deliver": "origin",               # 或 "telegram" / "discord" / "local"
    "agent_message": "生成今日工作报告",
    "origin": {"platform": "telegram", "chat_id": "12345"}
}
```

- **平台自动投递**：任务完成后自动推送到创建时所在平台
- **60 秒心跳检查**：后台线程每分钟扫描 `~/.hermes/cron/` 中的到期任务
- **文件锁防重**：`.tick.lock` 防止多进程重复执行
- **静默标记**：`[SILENT]` 前缀可抑制无新内容时的投递

---

## 四、关键工程亮点

### 4.1 Profile（多实例隔离）

Hermes 支持**完全隔离的多实例**，通过 `HERMES_HOME` 环境变量实现：

```bash
hermes -p coder      # 实例：~/.hermes/profiles/coder/
hermes -p research    # 实例：~/.hermes/profiles/research/
```

每个 profile 拥有独立的：config.yaml、.env（API Keys）、skills、memory、sessions、gateway 配置。

关键机制：`_apply_profile_override()` 在任何模块导入**之前**解析 `--profile/-p` 参数并设置 `HERMES_HOME` env var，确保所有模块级缓存的路径都指向正确实例。

### 4.2 Skin 主题引擎 (`hermes_cli/skin_engine.py`)

纯数据驱动的 CLI 视觉定制，无需修改代码：

```yaml
# ~/.hermes/skins/cyberpunk.yaml
colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
spinner:
  thinking_verbs: ["jacking in", "decrypting"]
  wings: [["⟨⚡", "⚡⟩"]]
branding:
  agent_name: "Cyber Agent"
```

内置 4 个主题：`default`（金色）、`ares`（猩红）、`mono`（灰阶）、`slate`（冷蓝）。

### 4.3 安全设计

**上下文注入防护**（`prompt_builder.py`）：
- 扫描 8 类 prompt injection 模式（忽略指令、隐藏 div、curl exfil、secret 读取等）
- 检测零宽 Unicode 字符（U+200B 等 10 种）
- 发现威胁时阻止文件加载并记录日志

**凭证安全**：
- API keys 存储在 `~/.hermes/.env`，从不提交到 Git
- 凭证池支持隔离存储和运行时注入

**MCP 动态工具清理**：
- MCP 服务器工具变更时自动 deregister 旧工具，防止 schema 陈旧

### 4.4 终端后端多样性

支持 6 种运行环境：

| 后端 | 用途 | 隔离级别 |
|------|------|---------|
| local | 本地执行 | 进程级 |
| Docker | 容器隔离 | 容器级 |
| SSH | 远程执行 | 主机级 |
| Daytona | 云端管理 | 沙箱 |
| Singularity | HPC 容器 | 容器级 |
| Modal | Serverless | 函数级冷启动 |

---

## 五、与 OpenClaw 的架构对比

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **核心语言** | Python 3.11 | Node.js |
| **包管理** | uv | npm |
| **CLI 框架** | prompt_toolkit + Rich | 内置 TUI |
| **工具注册** | 中央 Registry 单例 | Skill 插件系统 |
| **记忆系统** | MemoryManager + Honcho | MEMORY.md / USER.md |
| **多平台** | Gateway 统一接入 | channel plugin 架构 |
| **定时任务** | 内置 cron 模块 | cron job (Gateway) |
| **自改进** | 自动创建/改进 Skills | 无内置 |
| **编辑器集成** | ACP (VS Code/Zed/JetBrains) | ACP |
| **Serverless** | Daytona / Modal | 无 |
| **Profiles** | HERMES_HOME 隔离 | workspace 隔离 |
| **迁移路径** | OpenClaw → Hermes | N/A |

**核心差异**：Hermes 以 Python 生态深度和"自改进"能力为核心卖点；OpenClaw 以 Node.js 生态和插件 Skill 系统见长。两者均支持 ACP 协议和 CLI，表明行业在 Agent 架构上正在收敛。

---

## 六、源码研读洞见

### 6.1 值得学习的工程实践

1. **注册表模式的彻底应用**：工具、slash 命令、平台适配器均使用统一注册表，避免了条件分支膨胀
2. **`_SafeWriter` 模式**：在 daemon/systemd 场景下 stdio 可能随时断裂，主动捕获比被动崩溃更优雅
3. **Profile 隔离的 env var 前置拦截**：在所有模块导入前解析 `--profile`，从根本上保证隔离性
4. **上下文围栏（fenced context）**：`<memory-context>` 标签在语义层面隔离记忆内容和用户输入
5. **FTS5 会话搜索**：用 SQLite FTS5 实现跨会话历史搜索，避免引入重型搜索引擎

### 6.2 潜在改进点

1. **run_agent.py 达 9722 行**：即使拆分出 `agent/` 子模块，核心 orchestrator 仍然过大，可进一步按职责拆分（消息处理、预算管理、错误恢复各自独立）
2. **tools/ 目录 38860 行**：各工具文件大小不一（`terminal_tool.py` 1757行 vs `skills_hub.py` 2707行），可按领域进一步拆分
3. **多套配置加载**：CLI 用 `load_cli_config()`，Gateway 直接 YAML，工具用 `load_config()`——三套分离增加了维护成本
4. **honcho 外部依赖**：`honcho-ai` 引入外部记忆系统，增加了调试复杂度和依赖链风险

### 6.3 技术债警示

1. **测试在 `/tmp` 而非项目内**：`tests/` 分散，部分测试依赖 mock 而非真实场景
2. **MCP 工具清理逻辑**：当 MCP 服务器发 `notifications/tools/list_changed` 时清空旧工具，但如果同一工具名称被多个 MCP 服务器提供会存在竞态
3. **`simple_term_menu` vs `curses`**：已知 `simple_term_menu` 在 iTerm2/tmux 有渲染 bug，但仍有部分模块使用

---

## 七、总结

Hermes Agent 是当前 AI Agent 领域架构最完整、工程化程度最高的开源项目之一。其设计理念可以用一句话概括：**"让 AI 编程行为变得确定性"**——通过 YAML 工作流定义开发流程、通过工具注册表统一工具接口、通过 Profile 隔离实例状态、通过 Gateway 解耦消息平台、通过 Skills 实现自改进记忆。

对于 OpenClaw 而言，Hermes 的核心竞争启示在于：
1. **Skills 自改进机制**值得借鉴——让 Agent 从错误中生成新 Skills 比手动维护更高效
2. **Gateway 统一路由**的架构比分立 channel 插件更易维护
3. **Profile 隔离**的 env var 前置拦截模式是保证多租户安全的最简方案

---

*报告生成时间：2026-04-10 03:00 (Asia/Shanghai)*
*分析工具：静态源码解析 + 文档交叉验证*
*数据来源：github.com/NousResearch/hermes-agent (main branch, latest commit)*
