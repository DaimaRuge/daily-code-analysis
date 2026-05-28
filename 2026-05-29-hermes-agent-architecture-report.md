# Hermes Agent 技术架构与源码研读报告

> **项目**: [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)  
> **Stars**: 171,415 ⭐ | **Forks**: 28,733 | **Language**: Python / TypeScript  
> **License**: MIT  
> **分析日期**: 2026-05-29  
> **版本**: v0.15.0

---

## 一、执行摘要

Hermes Agent 是由 Nous Research 构建的**自进化型 AI Agent 框架**，其核心定位是"The agent that grows with you"——具备内置学习循环、自主技能创建、跨会话记忆持久化和用户建模能力。该框架支持在任何基础设施上运行（$5 VPS 到 GPU 集群），并通过统一网关对接 Telegram、Discord、Slack、WhatsApp 等 20+  messaging 平台。

**关键架构特征：**
- 模块化插件架构：支持模型提供者、记忆系统、平台适配器、工具集的热插拔
- 多模型兼容：30+ 提供商（OpenAI、Anthropic、DeepSeek、Kimi、MiniMax 等）零代码切换
- 三层终端后端：本地、Docker、SSH、Modal、Daytona、Singularity
- 自改进闭环：Agent 从经验创建技能、使用中自我改进、FTS5 会话搜索、Honcho 辩证用户建模
- ACP/MCP 双协议：同时支持 Agent-Computer Protocol 和 Model Context Protocol

---

## 二、系统架构总览

### 2.1 架构分层图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           入口层 (Entry Points)                          │
│                                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │
│  │ CLI TUI    │  │ Gateway    │  │ ACP Server │  │ MCP Server         │ │
│  │ (hermes)   │  │ (multi-pl) │  │ (editors)  │  │ (external tools)   │ │
│  │ Node TUI   │  │ Telegram   │  │ Zed/Vim    │  │ Claude Code        │ │
│  │ Web Dash   │  │ Discord    │  │ VS Code    │  │ Cursor             │ │
│  │ Batch Run  │  │ Slack...   │  │            │  │                    │ │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────────┬──────────┘ │
│        │               │               │                   │            │
└────────┼───────────────┼───────────────┼───────────────────┼────────────┘
         │               │               │                   │
         ▼               ▼               ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      AIAgent 核心运行时 (run_agent.py)                    │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Prompt       │  │ Provider     │  │ Tool         │  │ Session    │ │
│  │ Builder      │  │ Resolution   │  │ Dispatch     │  │ Storage    │ │
│  │ (prompt_     │  │ (runtime_    │  │ (model_      │  │ (SQLite    │ │
│  │  builder.py) │  │  provider.py)│  │  tools.py)   │  │  + FTS5)   │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                 │                 │                │        │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐              │
│  │ Compression  │  │ 3 API Modes  │  │ Tool Registry│              │
│  │ & Caching    │  │ chat_compl.  │  │ (registry.py)│              │
│  │              │  │ codex_resp.  │  │ 70+ tools    │              │
│  │              │  │ anthropic    │  │ 28 toolsets  │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└─────────┴─────────────────┴─────────────────┴────────────────────────┘
          │                                    │
          ▼                                    ▼
┌───────────────────┐              ┌──────────────────────────────┐
│  State Storage    │              │  Tool Backends                │
│  SQLite + FTS5    │              │  ├─ Terminal (6 backends)   │
│  hermes_state.py  │              │  ├─ Browser (5 backends)    │
│  gateway/session  │              │  ├─ Web Search (4 backends) │
│  kanban_db.py     │              │  ├─ MCP (dynamic)            │
│                   │              │  ├─ File, Vision, Code Exec  │
│                   │              │  └─ Subagent Delegation      │
└───────────────────┘              └──────────────────────────────┘
```

### 2.2 核心模块职责

| 模块 | 文件/目录 | 职责 | 规模 |
|------|----------|------|------|
| **AIAgent 核心** | `run_agent.py` + `agent/` | 对话循环、工具调度、模型调用 | ~58K 行 |
| **CLI 入口** | `hermes_cli/main.py` | 所有 `hermes` 子命令 | ~554K 行 |
| **网关** | `gateway/run.py` | 多平台消息收发 | ~889K 行 |
| **工具注册** | `tools/registry.py` | 70+ 工具的发现与调度 | - |
| **状态存储** | `hermes_state.py` | SQLite + FTS5 会话数据库 | - |
| **TUI 渲染** | `tui_gateway/server.py` | Node.js/React 终端 UI | ~254K 行 |
| **ACP 适配器** | `acp_adapter/` | 编辑器协议桥接 | - |
| **MCP 服务** | `mcp_serve.py` | 对外暴露 MCP 工具 | - |
| **看板** | `hermes_cli/kanban.py` | 任务管理与编排 | ~107K 行 |

---

## 三、核心子系统深度解析

### 3.1 AIAgent 运行时架构

Hermes 的核心是一个**事件驱动的对话状态机**，由 `AIAgent` 类（位于 `run_agent.py`）驱动。该文件虽然体量巨大，但已通过提取模式将子系统拆分到 `agent/` 目录下的独立模块：

#### 3.1.1 对话循环 (conversation_loop.py)

```python
# 核心流程伪代码
def run_conversation(agent, user_message):
    # 1. 消息预处理
    messages = prepare_messages(agent, user_message)
    
    # 2. 上下文压缩（默认引擎：lossy summarization）
    if token_count > threshold:
        messages = agent.context_compressor.compress(messages)
    
    # 3. 模型调用（支持三种 API 模式）
    response = call_model(agent, messages)  # chat_completion | codex_responses | anthropic
    
    # 4. 工具调用解析与调度
    if response.tool_calls:
        # 并行执行（最多 8 worker 线程）
        results = execute_tool_calls_concurrent(agent, response.tool_calls)
        messages.extend(results)
        # 递归：将结果送回模型继续对话
        return run_conversation(agent, None)
    
    # 5. 后处理：记忆审查、技能改进建议
    agent._maybe_trigger_background_review()
    return response.content
```

**关键设计决策：**
- **递归式工具循环**：不是一次调用就结束，而是将工具结果重新注入消息历史，让模型决定下一步
- **并发工具执行**：`execute_tool_calls_concurrent()` 使用线程池，最多 8 个并发 worker
- **安全护栏**：`tool_guardrails.py` 在工具调用前执行策略检查（destructive 命令检测、权限校验）
- **检查点机制**：文件修改前自动创建 Git 检查点，支持 `hermes undo`

#### 3.1.2 提示词构建系统 (prompt_builder.py)

```
System Prompt 组装流水线：
┌─────────────────┐
│ Base Prompt     │  ← 从 skills/SKILL.md 和 agent 指令构建
│ (system_prompt) │
└────────┬────────┘
         │
┌────────▼────────┐
│ Context Engine  │  ← 可插拔：local (默认) / honcho / hybrid
│ (context_engine)│    注入用户建模、跨会话记忆
└────────┬────────┘
         │
┌────────▼────────┐
│ Session Context │  ← gateway 来源信息（平台、聊天类型、用户）
│ (session_source)│
└────────┬────────┘
         │
┌────────▼────────┐
│ Active Skills   │  ← 当前启用的技能列表及其指令
│ (skill_bundles) │
└────────┬────────┘
         │
┌────────▼────────┐
│ Tool Schema     │  ← 当前可用工具的 JSON Schema
│ (tool_registry) │
└─────────────────┘
```

**Prompt Caching**：针对 Anthropic API 的 `apply_anthropic_cache_control()` 自动在消息列表中插入 `cache_control` 标记，降低 token 成本。

#### 3.1.3 模型适配层

Hermes 支持 **三种 API 调用模式**，由 `runtime_provider.py` 自动选择：

| 模式 | 适用场景 | 关键文件 |
|------|---------|---------|
| `chat_completion` | 标准 OpenAI 兼容 API | `agent/chat_completion_helpers.py` |
| `codex_responses` | OpenAI Codex / o1 系列 | `agent/codex_responses_adapter.py` |
| `anthropic` | Claude Messages API | `agent/anthropic_adapter.py` |

**30+ 模型提供者**：每个提供者通过 `plugins/model-providers/<name>/` 目录中的 `plugin.yaml` 注册，包含模型列表、认证方式、请求格式转换逻辑。

### 3.2 插件架构（Plugin System）

Hermes 的插件系统是其**最具扩展性的设计**之一。所有非核心功能都通过插件实现：

#### 3.2.1 插件目录结构

```
plugins/
├── model-providers/          # 30+ 模型提供商适配器
│   ├── anthropic/            # Claude
│   ├── openai/               # GPT-4
│   ├── deepseek/             # DeepSeek
│   ├── kimi-coding/          # Moonshot
│   └── ...
├── memory/                   # 记忆系统提供者（可插拔，一次只能激活一个）
│   ├── honcho/               # Honcho AI 辩证用户建模
│   ├── mem0/                 # Mem0 向量记忆
│   ├── holographic/          # 全息记忆
│   └── ...
├── browser/                  # 浏览器自动化后端
│   ├── firecrawl/            # Firecrawl 爬虫
│   └── browserbase/          # Browserbase 云浏览器
├── web/                      # 网页搜索后端
│   ├── firecrawl/            # Firecrawl 搜索
│   ├── searxng/              # SearXNG 私有搜索
│   └── tavily/               # Tavily AI 搜索
├── image_gen/                # 图像生成后端
├── video_gen/                # 视频生成后端
├── platforms/                # 消息平台网关适配器
│   ├── discord/              # Discord Bot
│   ├── telegram/             # Telegram Bot
│   ├── slack/                # Slack Bot
│   └── ...
└── context_engine/           # 上下文引擎扩展
```

#### 3.2.2 插件加载机制

```python
# hermes_cli/plugins.py 中的 PluginManager
class PluginManager:
    def discover_plugins(self, plugin_dirs):
        """扫描目录，加载所有 plugin.yaml 定义的插件"""
        
    def load_plugin(self, plugin_path):
        """动态导入插件模块，注册 hooks 和 tools"""
        
    def register_hooks(self, plugin):
        """注册生命周期 hooks：
            - on_agent_init      # Agent 初始化时
            - on_message         # 收到消息时
            - on_tool_call       # 工具调用前后
            - on_session_end     # 会话结束时
        """
```

**插件元数据** (`plugin.yaml`)：
```yaml
name: google_meet
version: "1.0.0"
author: Nous Research
description: Google Meet 集成 - 加入会议、管理音频
requires:
  - google-auth
  - google-api-python-client
hooks:
  - on_agent_init
  - on_tool_call
tools:
  - meet_join
  - meet_leave
  - meet_mute
```

### 3.3 工具与技能系统

#### 3.3.1 工具注册中心

```
tools/
├── registry.py               # 中央注册表：发现、加载、调度所有工具
├── terminal_tool.py          # 终端命令执行（6 种后端）
├── file_tools.py             # read_file, write_file, patch, search_files
├── web_tools.py              # web_search, web_extract
├── browser_tool.py             # 10 个浏览器自动化工具
├── code_execution_tool.py    # execute_code 沙箱
├── delegate_tool.py          # subagent 委派
├── mcp_tool.py               # MCP 客户端（大型文件）
├── cronjob_tools.py          # 定时任务管理
└── environments/             # 终端后端实现
    ├── local.py              # 本地终端
    ├── docker.py             # Docker 容器
    ├── ssh.py                # SSH 远程
    ├── modal.py              # Modal 云函数
    ├── daytona.py            # Daytona 开发环境
    └── singularity.py        # Singularity 容器
```

**工具调用流程：**
1. `model_tools.py` 收集当前上下文中可用工具的 JSON Schema
2. `prompt_builder.py` 将 schema 注入 system prompt
3. LLM 返回 `tool_calls` 数组
4. `tool_executor.py` 并行调度执行
5. 结果格式化后重新注入消息历史

#### 3.3.2 技能系统（Skills）

技能是 Hermes 的**高级抽象**——一组相关的工具、指令和上下文，打包成可复用的能力单元：

```
skills/                          # 内置技能（100+）
├── software-development/        # 软件开发技能
│   ├── spike/                   # 快速原型
│   ├── plan/                    # 计划制定
│   └── python-debugpy/          # Python 调试
├── devops/                      # DevOps 技能
│   ├── kanban-orchestrator/     # 看板编排
│   └── webhook-subscriptions/   # Webhook 管理
├── creative/                    # 创意技能
│   ├── architecture-diagram/    # 架构图生成
│   └── manim-video/             # Manim 视频
├── research/                    # 研究技能
│   └── llm-wiki/                # LLM 知识库
└── mcp/                         # MCP 相关技能
    └── native-mcp/              # 原生 MCP 集成

optional-skills/                 # 可选技能（需额外安装）
└── ...
```

**技能元数据** (`SKILL.md`)：
```markdown
# Skill Name

## Description
该技能的用途描述...

## Tools
- tool_name: 工具描述

## Instructions
给 LLM 的额外指令...

## Examples
使用示例...
```

**技能发现机制**：`agent/skill_utils.py` 使用 `rglob` 扫描所有 `SKILL.md` 文件，解析 YAML frontmatter 和 Markdown 内容，构建技能索引。

### 3.4 多平台网关架构（Gateway）

网关是 Hermes **最复杂的子系统**之一，负责将 Agent 连接到外部消息平台：

#### 3.4.1 网关核心组件

```
gateway/
├── run.py                  # GatewayRunner — 消息调度中枢 (~889K 行)
├── session.py              # SessionStore — 会话持久化
│   ├── SessionSource       # 消息来源描述（平台、聊天、用户）
│   ├── SessionContext      # 会话上下文（历史、状态）
│   └── SessionResetPolicy  # 自动重置策略
├── delivery.py             # DeliveryRouter —  outbound 路由
├── config.py               # GatewayConfig — 平台配置
├── pairing.py              # DM 配对授权
├── hooks.py                # Hook 发现与生命周期
├── stream_consumer.py      # 流式消息消费
└── platforms/              # 平台适配器
    ├── telegram/           # Telegram Bot API
    ├── discord/            # Discord.py
    ├── slack/              # Slack Bolt
    ├── whatsapp/           # WhatsApp Web.js
    ├── signal/             # Signal CLI
    ├── simplex/            # SimpleX Chat
    ├── mattermost/         # Mattermost
    ├── google_chat/        # Google Chat
    ├── line/               # LINE
    ├── irc/                # IRC
    └── qqbot/              # QQ Bot
```

#### 3.4.2 会话生命周期

```
用户消息到达
    │
    ▼
┌─────────────────┐
│ 平台适配器       │  ← 解析平台特定格式
│ (Platform Bot)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ SessionStore    │  ← 查找/创建会话
│                 │    持久化到 SQLite
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 动态上下文注入   │  ← 将 SessionSource 注入 system prompt
│ (Context Hook)  │    "You are talking to user_XXX on Telegram in a group"
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ AIAgent 处理     │  ← 调用 run_agent.AIAgent.run_conversation()
│                 │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ DeliveryRouter  │  ← 将响应路由回原平台
│                 │    支持 cron job 输出到任意平台
└─────────────────┘
```

**动态系统提示注入**是网关的关键创新：Agent 的 system prompt 会根据消息来源动态调整，使其知道"自己在和谁对话、在哪个平台上、是什么类型的聊天"。

### 3.5 记忆与状态管理

#### 3.5.1 三层记忆架构

```
┌─────────────────────────────────────────┐
│  Layer 3: 长期记忆 (Long-term)          │
│  ├─ Honcho AI 辩证用户建模              │
│  ├─ Mem0 向量记忆检索                    │
│  └─ 自定义记忆提供者 (plugin-based)      │
├─────────────────────────────────────────┤
│  Layer 2: 会话记忆 (Session)             │
│  ├─ SQLite + FTS5 全文搜索              │
│  ├─ 跨会话消息搜索 + LLM 摘要            │
│  └─ SessionStore 持久化                │
├─────────────────────────────────────────┤
│  Layer 1: 上下文记忆 (Context)            │
│  ├─ Conversation message history       │
│  ├─ ContextCompressor (有损压缩)        │
│  └─ 256K 上下文窗口管理                  │
└─────────────────────────────────────────┘
```

#### 3.5.2 记忆提供者插件

`plugins/memory/` 实现了**可插拔的记忆后端**架构：

```python
# plugins/memory/__init__.py

def discover_memory_providers():
    """扫描 bundled 和用户目录的记忆提供者"""
    
def load_memory_provider(name: str) -> MemoryProvider:
    """加载指定名称的记忆提供者"""
    # 优先级：bundled > user-installed
    # 一次只能激活一个提供者
```

**可用提供者**：
- **honcho**: Honcho AI 的辩证用户建模（默认推荐）
- **mem0**: Mem0 的向量记忆检索
- **holographic**: 全息记忆系统
- **hindsight**: 后见之明记忆
- **openviking**: OpenViking 记忆
- **supermemory**: Supermemory
- **retaindb**: RetainDB
- **byterover**: ByteRover

#### 3.5.3 状态存储 (hermes_state.py)

```python
# 核心数据结构
class SessionDB:
    """SQLite + FTS5 的会话数据库"""
    
    def save_message(self, session_id, message):
        """保存消息到 SQLite，同时更新 FTS5 索引"""
        
    def search_sessions(self, query):
        """FTS5 全文搜索历史会话"""
        
    def get_conversation_history(self, session_id, limit=100):
        """获取会话历史消息"""
```

**FTS5 索引**：支持自然语言搜索跨会话历史，配合 LLM 摘要实现"Agent 搜索自己的过去对话"。

### 3.6 看板任务系统（Kanban）

Hermes 内置了一个**完整的看板任务管理系统**，用于 Agent 自主任务编排：

#### 3.6.1 任务状态机

```
        ┌──────────┐
        │   todo   │  ← 新建任务
        └────┬─────┘
             │ ready
             ▼
        ┌──────────┐
        │  ready   │  ← 准备执行
        └────┬─────┘
             │ start
             ▼
        ┌──────────┐     ┌──────────┐
        │ running  │────▶│ blocked  │  ← 被阻塞
        └────┬─────┘     └──────────┘
             │ complete / fail
             ▼
        ┌──────────┐     ┌──────────┐
        │   done   │     │ scheduled│  ← 定时执行
        └──────────┘     └──────────┘
             │
             ▼
        ┌──────────┐
        │ archived │  ← 归档
        └──────────┘
```

#### 3.6.2 任务模型

```python
@dataclass
class Task:
    id: str
    title: str
    body: str
    assignee: str          # 执行者（可以是子 Agent）
    status: str            # todo|ready|running|blocked|done|scheduled|archived
    priority: int
    tenant: str            # 所属工作空间
    workspace_kind: str    # scratch|worktree|dir
    workspace_path: str
    branch_name: str       # Git 分支（用于代码任务）
    skills: List[str]      # 需要的技能
    max_retries: int
    session_id: str
    workflow_template_id: str
    current_step_key: str
```

#### 3.6.3 Swarm 模式

`kanban_swarm.py` 实现了**多 Agent 协作**的 Swarm 模式：
- 将大任务分解为子任务
- 每个子任务分配给一个子 Agent（通过 `delegate_tool`）
- 子 Agent 在隔离环境中并行执行
- 主 Agent 协调结果整合

### 3.7 Cron 调度系统

```
cron/
├── jobs.py           # 任务定义与管理
├── scheduler.py      # 调度引擎
└── delivery.py       # 结果投递
```

**功能特性**：
- 自然语言定时："every day at 9am", "every Monday"
- 多平台投递：cron 结果可发送到任意已配置平台
- 技能隔离：每个 cron job 可指定使用特定技能集
- 重复控制：支持最大重复次数和已完成计数

### 3.8 TUI 与 Web 界面

#### 3.8.1 三层 UI 架构

```
┌─────────────────────────────────────────────┐
│  Layer 1: Node.js TUI (Ink/React)           │
│  ├─ 文件: tui_gateway/server.py            │
│  ├─ 渲染: React Ink 组件                    │
│  ├─ 输入: multiline editing, slash commands │
│  ├─ 输出: streaming tool output             │
│  └─ 通信: JSON-RPC over stdio               │
├─────────────────────────────────────────────┤
│  Layer 2: Web Dashboard (Vite/React)        │
│  ├─ 文件: web/                              │
│  ├─ 框架: React + Vite + TypeScript         │
│  ├─ 功能: 聊天面板、设置界面、插件管理      │
│  └─ 通信: WebSocket                         │
├─────────────────────────────────────────────┤
│  Layer 3: Python CLI (hermes_cli/main.py)   │
│  ├─ 传统终端 UI（curses fallback）          │
│  ├─ 命令补全                              │
│  └─ 皮肤引擎 (skin_engine.py)               │
└─────────────────────────────────────────────┘
```

#### 3.8.2 TUI 网关通信协议

```python
# tui_gateway/server.py

def dispatch():
    """主事件循环：从 Node TUI 接收 JSON-RPC 请求"""
    # 1. 读取 stdio 上的 JSON-RPC 消息
    # 2. 路由到对应 handler
    # 3. 通过 TeeTransport 同时输出到：
    #    - stdio（回显给 TUI）
    #    - WebSocket sidecar（Dashboard 侧边栏）
```

---

## 四、数据流分析

### 4.1 单轮对话完整数据流

```
[User Input]
    │
    ▼
[Entry Layer] ──→ CLI / Gateway / ACP / MCP
    │
    ▼
[Message Preparation]
    ├─ 1. Session lookup (SQLite)
    ├─ 2. Context injection (平台信息、用户信息)
    ├─ 3. Memory retrieval (Honcho / Mem0 / FTS5)
    ├─ 4. Skill loading (解析 SKILL.md)
    ├─ 5. Tool schema collection (当前可用工具)
    ├─ 6. Prompt assembly (system + user + context)
    └─ 7. Token estimation + compression (如需)
    │
    ▼
[LLM API Call]
    ├─ Provider resolution (30+ providers)
    ├─ API mode selection (chat_completion / codex / anthropic)
    ├─ Credential resolution (credential_pool)
    ├─ Rate limit handling (nous_rate_guard)
    └─ Streaming response processing
    │
    ▼
[Response Processing]
    ├─ 如果是普通消息 → 直接返回
    └─ 如果有 tool_calls → Tool Dispatch
         │
         ▼
    [Tool Execution]
    ├─ Guardrail check (destructive? approved?)
    ├─ Concurrent execution (ThreadPool, max 8)
    ├─ Result formatting (text / multimodal)
    ├─ Checkpoint creation (Git-based)
    └─ Budget enforcement (turn budget)
         │
         ▼
    [Recursive Loop]
    └─ 工具结果重新注入消息历史
       → 再次调用 LLM（可能继续调用工具）
    │
    ▼
[Post-Turn]
    ├─ Background review trigger
    ├─ Memory persistence (session storage)
    ├─ Skill improvement suggestion
    ├─ Usage tracking (pricing estimation)
    └─ Trajectory saving (for training)
    │
    ▼
[Output Delivery]
    └─ 路由到原始平台 / CLI / Dashboard
```

### 4.2 网关消息流

```
[Platform Bot] (Telegram/Discord/Slack)
    │
    ▼
[Gateway.run.py] ──→ 解析 webhook / polling 消息
    │
    ▼
[SessionStore] ──→ lookup_or_create_session()
    │
    ▼
[Context Builder] ──→ build_session_context_prompt()
    │         注入："You are on Telegram, group chat, talking to @username"
    ▼
[AIAgent.run_conversation()] ──→ 处理消息
    │
    ▼
[DeliveryRouter] ──→ route_response()
    │         根据 SessionSource.platform 选择发送方式
    ▼
[Platform Adapter] ──→ 发送回复消息
```

---

## 五、部署与运行时架构

### 5.1 六种运行模式

| 模式 | 命令 | 适用场景 | 特点 |
|------|------|---------|------|
| **本地终端** | `hermes` | 开发、日常对话 | TUI + CLI，直接访问本地文件 |
| **Gateway** | `hermes gateway` | 生产、24/7 服务 | 守护进程，多平台接入 |
| **Docker** | `docker run hermes` | 隔离环境 | 完整容器化，s6-overlay 进程管理 |
| **SSH** | `hermes ssh <host>` | 远程服务器 | 通过 SSH 在远程运行 Agent |
| **Modal** | `hermes modal` | 弹性计算 | 空闲时休眠，按需唤醒，成本极低 |
| **Daytona** | `hermes daytona` | 开发环境 | 云端开发沙箱 |

### 5.2 Docker 架构

```
hermes-agent Docker Image
├─ s6-overlay 进程监督
│  ├─ main-hermes      # 主 Agent 进程
│  ├─ dashboard        # Web Dashboard 服务
│  └─ user             # 用户自定义服务
├─ cont-init.d         # 初始化脚本
│  ├─ 015-supervise-perms    # 权限设置
│  └─ 02-reconcile-profiles  # 配置文件同步
├─ SOUL.md             # 默认人格配置
└─ stage2-hook.sh      # 启动钩子
```

---

## 六、关键设计模式

### 6.1 提取模式（Extraction Pattern）

Hermes 使用一种独特的代码组织模式：将大文件中的方法提取到独立模块，但保留原文件中的薄包装器：

```python
# 原文件 run_agent.py (巨大)
class AIAgent:
    def __init__(self, ...):
        # ~1400 行初始化逻辑
        ...
    
    def run_conversation(self, ...):
        # ~3900 行对话循环
        ...

# 提取后
# agent/agent_init.py
def init_agent(agent, ...):
    # 所有初始化逻辑移到这里
    ...

# agent/conversation_loop.py  
def run_conversation(agent, ...):
    # 所有对话逻辑移到这里
    ...

# run_agent.py 保留薄包装器
class AIAgent:
    def __init__(self, ...):
        from agent.agent_init import init_agent
        init_agent(self, ...)
    
    def run_conversation(self, ...):
        from agent.conversation_loop import run_conversation
        return run_conversation(self, ...)
```

**好处**：
- 保持向后兼容（测试补丁仍然有效）
- 主文件可读性提升
- 子模块可独立测试
- 避免循环导入问题

### 6.2 插件化一切（Plugin Everything）

除了核心对话循环外，几乎所有功能都通过插件实现：
- 模型提供者 → 插件
- 记忆系统 → 插件
- 消息平台 → 插件
- 浏览器后端 → 插件
- 搜索后端 → 插件

### 6.3 延迟导入（Lazy Import）

大量模块使用延迟导入策略，减少启动时间和循环依赖：

```python
def _ra():
    """Lazy reference to run_agent for test patching compatibility."""
    import run_agent
    return run_agent

_yaml_load_fn = None
def yaml_load(content):
    global _yaml_load_fn
    if _yaml_load_fn is None:
        import yaml
        ...
```

### 6.4 注册表模式（Registry Pattern）

多个子系统使用注册表模式管理扩展点：
- `tools/registry.py` — 工具注册表
- `gateway/platform_registry.py` — 平台注册表
- `hermes_cli/commands.py` — 命令注册表 (`COMMAND_REGISTRY`)
- `agent/web_search_registry.py` — 搜索后端注册表
- `agent/image_gen_registry.py` — 图像生成注册表

---

## 七、代码质量与工程实践

### 7.1 测试架构

```
tests/
├── agent/              # Agent 核心测试
├── gateway/            # 网关测试
├── hermes_cli/         # CLI 测试
├── plugins/            # 插件测试
├── e2e/                # 端到端测试
├── integration/        # 集成测试
├── stress/             # 压力测试
└── fakes/              # 测试替身
```

### 7.2 CI/CD

```
.github/workflows/
├── tests.yml           # 单元测试
├── lint.yml            # 代码检查
├── release.yml         # 发布流程
└── ...
```

### 7.3 代码组织统计

| 指标 | 数值 |
|------|------|
| Python 文件数 | ~1,960 |
| TypeScript 文件数 | ~422 |
| `agent/` 目录总行数 | ~58,363 |
| `hermes_cli/main.py` | ~554,294 字节 |
| `gateway/run.py` | ~888,928 字节 |
| 模型提供者插件 | 30+ |
| 记忆提供者插件 | 8 |
| 消息平台适配器 | 20+ |
| 内置技能 | 100+ |
| 可选技能 | 50+ |
| 工具总数 | 70+ |
| 工具集 | 28 |

---

## 八、技术洞察与架构评价

### 8.1 架构优势

1. **极致的模块化**：几乎一切可插拔，从模型到平台到记忆系统
2. **多平台原生体验**：不是简单的 Webhook 桥接，而是深度集成每个平台的特性（Discord reactions、Telegram topics 等）
3. **自进化闭环**：真正的"会成长的 Agent"——技能自主创建、记忆主动整理、跨会话学习
4. **生产就绪**：s6-overlay 进程管理、检查点回滚、权限护栏、速率限制、故障转移
5. **开放协议**：同时支持 ACP（编辑器集成）和 MCP（工具生态）

### 8.2 架构挑战

1. **文件体积过大**：`run_agent.py`、`gateway/run.py`、`hermes_cli/main.py` 等核心文件超过 50万+ 字节，虽然使用了提取模式，但重构仍有提升空间
2. **模块间耦合**：尽管有提取模式，核心模块间仍存在较多属性级耦合（`agent.xxx` 直接访问）
3. **配置复杂度**：`hermes_cli/config.py` 处理了极大量的配置选项和迁移逻辑
4. **单进程架构**：Gateway 和 Agent 运行在同一进程中，高并发场景可能成为瓶颈

### 8.3 学习要点

1. **提取模式是治理大文件的有效策略**：不破坏现有接口的前提下逐步拆分
2. **插件化一切是扩展性的终极答案**：Hermes 证明了一个 Agent 框架可以通过纯插件化支持 30+ 模型和 20+ 平台
3. **动态上下文注入让 Agent 更有"场景感"**：知道自己在和谁对话、在哪里对话，显著提升了交互质量
4. **FTS5 + LLM 摘要 = 低成本记忆检索**：不需要昂贵的向量数据库就能实现有效的跨会话搜索
5. **检查点是生产级 Agent 的必需品**：任何能修改文件的 Agent 都应该有自动回滚能力

### 8.4 与 OpenClaw 的关系

Hermes 与 OpenClaw 有深厚的渊源：
- Hermes 的 topics 中明确标注了 `openclaw`
- 提供 `hermes claw migrate` 命令用于从 OpenClaw 迁移
- 兼容 `agentskills.io` 开放标准
- 技能目录结构与 OpenClaw 的 `skills/` 目录兼容

---

## 九、核心源码文件速查

| 文件 | 职责 | 推荐阅读 |
|------|------|---------|
| `agent/conversation_loop.py` | 核心对话循环 | ⭐⭐⭐ |
| `agent/prompt_builder.py` | 提示词组装 | ⭐⭐⭐ |
| `agent/tool_executor.py` | 工具调度执行 | ⭐⭐⭐ |
| `agent/context_compressor.py` | 上下文压缩 | ⭐⭐ |
| `gateway/session.py` | 会话管理 | ⭐⭐⭐ |
| `gateway/delivery.py` | 消息投递 | ⭐⭐ |
| `tools/registry.py` | 工具注册表 | ⭐⭐⭐ |
| `hermes_cli/plugins.py` | 插件管理 | ⭐⭐⭐ |
| `hermes_cli/kanban.py` | 看板系统 | ⭐⭐ |
| `mcp_serve.py` | MCP 服务 | ⭐⭐ |
| `acp_adapter/server.py` | ACP 适配 | ⭐⭐ |
| `hermes_state.py` | 状态存储 | ⭐⭐ |

---

## 十、结论

Hermes Agent 代表了 **2026 年 AI Agent 工程的最高水准**之一。其架构设计展现了从"实验室玩具"到"生产基础设施"的关键跨越所必需的全部要素：

- **模块化**到插件级别的一切
- **多平台**到 20+ messaging 平台
- **多模型**到 30+ 提供商
- **自进化**的闭环学习系统
- **生产级**的部署、监控、回滚能力
- **开放协议**兼容（ACP + MCP + agentskills.io）

对于构建类似系统的开发者，Hermes 的插件架构、提取模式、动态上下文注入和三层记忆模型都值得深入研究和借鉴。

---

*本报告由 OpenClaw 自动生成，基于 Hermes Agent v0.15.0 源码分析。*
