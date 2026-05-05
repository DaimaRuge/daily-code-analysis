# OpenAI Agents SDK 技术架构与源码研读报告

**项目**: [openai/openai-agents-python](https://github.com/openai/openai-agents-python)  
** stars**: 官方多智能体开发框架 | **语言**: Python 3.10+ | **License**: MIT  
**版本**: 0.14.1 | **分析日期**: 2026-04-17  
**分析人**: OpenClaw 自动架构分析系统

---

## 一、项目概述

OpenAI Agents SDK 是 OpenAI 官方开源的多智能体（Multi-Agent）工作流框架，采用轻量级 + 强能力结合的设计思路，定位是让开发者能快速构建、调试和部署多 Agent 协作系统。项目最大的差异化特色是**provider-agnostic（模型无关）**——不仅支持 OpenAI 全系列模型（Responses API / Chat Completions），还通过 LiteLLM、any-llm 等适配层支持 **100+ 大模型**，覆盖 Claude、Gemini、MiniMax、DeepSeek 等。

### 核心概念一览

| 概念 | 说明 |
|------|------|
| **Agent** | 配置了指令、工具、护栏和交接的 LLM 实例 |
| **Sandbox Agent** | 带容器执行环境的 Agent，可操作文件系统、运行命令、跨长周期任务保持状态 |
| **Handoffs** | Agent 之间的委托交接机制，一个 Agent 将任务转交给另一个专业 Agent |
| **Tools** | 函数工具 / MCP 工具 / hosted 工具，让 Agent 执行动作 |
| **Guardrails** | 输入/输出安全检查，可配置的内容验证 |
| **Human in the Loop** | 内置人工介入机制，关键步骤可暂停等待人工审批 |
| **Sessions** | 自动管理对话历史，支持多轮交互 |
| **Tracing** | 内置追踪系统，可观测 Agent 运行全流程 |
| **Realtime Agents** | 基于 gpt-realtime-1.5 的语音 Agent，支持实时对话 |

### 技术栈

- **核心依赖**: `openai>=2.26.0`, `pydantic>=2.12.2`, `griffelib>=2`, `websockets>=15`, `mcp>=1.19.0`
- **可选依赖**: voice（numpy/websockets）、litellm、any-llm、SQLAlchemy、Redis、Docker、E2B、Modal、Cloudflare Workers 等
- **开发工具**: `uv`（包管理）、`ruff`（代码规范）、`mypy`/`pyright`（类型检查）、`pytest`（测试）、`MkDocs`（文档）

---

## 二、系统架构总览

### 2.1 顶层目录结构

```
openai-agents-python/
├── src/agents/                  # 核心 SDK（主包）
│   ├── agent.py                 # Agent 定义（941 行）
│   ├── run.py                   # Runner 入口（1859 行）
│   ├── run_state.py             # 运行状态序列化/恢复（3304 行，最大文件）
│   ├── items.py                 # 对话项类型建模（829 行）
│   ├── tool.py                  # 工具定义（1938 行）
│   ├── result.py                # 运行结果建模（896 行）
│   ├── run_internal/             # 内部运行引擎（最核心）
│   │   ├── run_loop.py          # 主循环编排（1894 行）
│   │   ├── turn_resolution.py   # Turn 解析（1911 行）
│   │   ├── tool_execution.py     # 工具执行引擎（2329 行）
│   │   ├── tool_actions.py       # 工具动作（893 行）
│   │   ├── model_retry.py        # 模型重试逻辑（724 行）
│   │   └── ...
│   ├── models/                  # 模型层（支持多种 Provider）
│   │   ├── openai_responses.py   # OpenAI Responses API 实现
│   │   ├── openai_chatcompletions.py  # Chat Completions 实现
│   │   ├── multi_provider.py     # 多 Provider 聚合
│   │   └── interface.py          # Model 抽象接口
│   ├── handoffs/                # Agent 间交接机制
│   ├── memory/                   # 会话历史管理
│   │   └── session.py            # Session 协议/抽象基类
│   ├── guardrail.py              # 护栏机制
│   ├── tracing/                  # 可观测性
│   ├── realtime/                 # 实时语音 Agent
│   │   ├── openai_realtime.py    # GPT Realtime 1.5 实现
│   │   └── session.py
│   ├── sandbox/                  # 沙箱 Agent（v0.14.0 新增）
│   │   ├── runtime.py            # 沙箱运行时编排
│   │   ├── sandbox_agent.py      # 沙箱 Agent 定义
│   │   ├── session/              # 沙箱会话管理
│   │   ├── capabilities/         # 沙箱能力（工具/记忆/文件系统）
│   │   └── sandboxes/            # 多后端支持（UnixLocal/E2B/Modal/...）
│   ├── mcp/                      # MCP 协议支持
│   │   └── server.py             # MCP 服务器实现
│   └── extensions/               # 扩展（experimental/models/sandbox/memory）
├── examples/                     # 丰富示例
│   ├── basic/                    # 基础用法
│   ├── sandbox/                  # 沙箱示例
│   ├── memory/                   # 会话记忆示例
│   ├── realtime/                 # 实时语音示例
│   └── handoffs/                 # 多 Agent 交接示例
└── tests/                        # 完整测试套件
```

### 2.2 核心模块分层架构

```
┌─────────────────────────────────────────────────────────┐
│                      User Code                           │
│   Runner.run_sync() / Runner.run() / Agent.run()        │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                     run.py (Runner)                     │
│  ~1859 行：暴露给用户的入口，管理 run 生命周期           │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│              run_internal/run_loop.py                    │
│  ~1894 行：主循环编排器，协调所有组件                      │
│  核心：run_single_turn() / run_until_done()             │
└──────┬───────────────┬───────────────┬──────────────────┘
       │               │               │
       ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────────┐
│ Turn        │ │ Tool        │ │ Model Retry     │
│ Resolution  │ │ Execution   │ │ (model_retry)   │
│ ~1911 行    │ │ ~2329 行    │ │ ~724 行         │
│ 决定下一步   │ │ 工具调用/   │ │ 自动重试/      │
│ 做什么      │ │ 输出处理     │ │ 退避策略        │
└──────┬──────┘ └──────┬──────┘ └────────┬────────┘
       │               │               │
       ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────────┐
│ Agent       │ │ Model Layer │ │ Guardrails      │
│ ~941 行     │ │             │ │                 │
│ 指令/工具   │ │ openai_     │ │ Input/Output    │
│ /交接配置   │ │ responses   │ │ 验证            │
└─────────────┘ └─────────────┘ └─────────────────┘
```

---

## 三、核心模块深度分析

### 3.1 Agent 模型（agent.py, ~941 行）

`Agent` 是框架的核心抽象类，是一次 LLM 调用配置的载体：

```python
@dataclass
class Agent(Generic[TContext]):
    name: str
    instructions: str | Prompt | DynamicPromptFunction
    model: str | Model | None  # 模型标识或 Model 实例
    tools: list[Tool] = field(default_factory=list)
    input_guardrails: list[InputGuardrail[Any]] = field(default_factory=list)
    output_guardrails: list[OutputGuardrail[Any]] = field(default_factory=list)
    handoffs: list[Handoff[Any, Any]] = field(default_factory=list)
    tool_use_behavior: ToolUseBehavior | None = None
    output_type: type[BaseModel] | None = None
```

**设计亮点**：
- **泛型 TContext**：支持任意上下文类型，灵活注入自定义数据
- **DynamicPromptFunction**：指令可以是动态函数而非静态字符串
- **工具作为一等公民**：所有工具（FunctionTool/MCP/Hostd）统一建模
- **Handoff 作为特殊 Tool**：交接被建模为一种特殊的 Tool，模型主动决定何时交接

### 3.2 运行状态序列化（run_state.py, ~3304 行）

这是全 SDK 最大的文件，也是整个框架最精妙的设计之一——**完全可暂停、可恢复的运行状态**：

```python
@dataclass
class RunState:
    session: Session  # 对话历史
    run_context: RunContextWrapper[Any]
    items: deque[RunItem]  # 当前 Run 的所有 Item
    current_agent: Agent
    # ... 还有快照、usage 统计等
```

**核心价值**：支持 Human-in-the-Loop。当 Agent 执行到需要人工审批的节点时，整个 `RunState` 可以被序列化保存，暂停等待人工操作后再恢复——这对生产级工作流至关重要。

### 3.3 主循环引擎（run_internal/run_loop.py, ~1894 行）

主循环采用**事件驱动 + 阶段分离**的架构：

```
run_until_done()
  └─ run_single_turn()
       ├─ 1. 输入护栏检查 (input_guardrails)
       ├─ 2. 模型调用 (call_model)
       │    └─ 3. 流式输出处理 / 完整响应处理
       ├─ 4. 工具调用 + 执行
       ├─ 5. 输出护栏检查 (output_guardrails)
       └─ 6. Handoff 检测
```

**关键设计**：所有阶段都是**可插拔的**，通过 `RunHooks` 允许外部订阅每个阶段的事件。

### 3.4 工具执行引擎（run_internal/tool_execution.py, ~2329 行）

工具执行是最复杂的子系统之一：

```
ToolExecutionEngine
  ├─ resolve_tool()         # 根据 tool_call 找到对应 Tool
  ├─ prepare_tool_input()    # 解析参数（支持 strict/loose 模式）
  ├─ execute_tool()         # 实际执行
  │    ├─ FunctionTool      # Python 函数
  │    ├─ MCP Tool          # MCP 服务器工具
  │    ├─ ComputerTool      # AI 操控电脑（类似 Claude Computer）
  │    ├─ ApplyPatchTool    # 代码/文件补丁
  │    └─ LocalShellTool    # 本地 Shell
  ├─ handle_tool_error()    # 错误处理/重试
  └─ format_output()        # 标准化输出
```

**亮点**：支持**parallel tool execution**（并行工具调用），这对于需要同时查询多个数据源的场景非常重要。

### 3.5 模型层（models/）

模型层采用**适配器模式**，顶层是 `Model` 接口：

```python
class Model(Protocol):
    async def get_response(self, ...) -> Response: ...
    async def get_streaming_response(self, ...) -> AsyncIterator[ResponseStreamEvent]: ...
```

目前主要的实现：
- `OpenAIResponsesModel`：使用 OpenAI Responses API（官方推荐）
- `OpenAIChatCompletionsModel`：回退到 Chat Completions API
- `MultiProvider`：聚合多个 Provider，支持智能路由

这种设计使框架天然支持**多模型协作**——同一个工作流中可以切换不同模型。

### 3.6 Session 系统（memory/session.py）

Session 抽象了对话历史的存储：

```python
class Session(Protocol):
    session_id: str
    async def get_items(self, limit: int | None = None) -> list[TResponseInputItem]: ...
    async def add_items(self, items: list[TResponseInputItem]) -> None: ...
    async def pop_item(self) -> TResponseInputItem | None: ...
    async def clear_session(self) -> None: ...
```

**多种实现**：内置 SQLite、Redis、SQLAlchemy（多数据库）、File、Encrypted、OpenAI 原生会话等多种 Session 后端。这种灵活性使框架可以从小规模原型无缝迁移到生产级部署。

### 3.7 Sandbox Agent（v0.14.0 新增）

这是最新版本的核心亮点。Sandbox Agent 让 Agent 在一个隔离的容器环境中执行真实工作：

```python
agent = SandboxAgent(
    name="Workspace Assistant",
    instructions="Inspect the sandbox workspace before answering.",
    default_manifest=Manifest(
        entries={
            "repo": GitRepo(repo="openai/openai-agents-python", ref="main"),
        }
    ),
)
```

**支持的后端**：
- `UnixLocalSandboxClient`：本地 Unix 环境
- `E2B`：云端沙箱
- `Modal`：Modal Serverless
- `Daytona`、`Cloudflare Workers`、`Vercel`、`Blaxel` 等

Sandbox 的核心是**Manifest**机制——声明 Agent 可访问哪些文件系统、Git 仓库、环境变量等，运行时通过 `materialization` 按需加载。

### 3.8 Tracing 可观测性（tracing/）

内置完整的分布式追踪系统，基于 OpenTelemetry 语义：

```python
trace() → agent_span() → turn_span() → task_span() → generation_span()
                        → guardrail_span()
                        → handoff_span()
                        → mcp_tools_span()
```

所有 Span 数据（AgentSpanData、TurnSpanData、GenerationSpanData 等）统一建模，可接入任何 OpenTelemetry 兼容的追踪后端（Phospho、Braintrust 等）。

---

## 四、技术亮点与创新点

### 4.1 Provider-Agnostic 的模型抽象

整个框架围绕 `Model` 接口而非特定 SDK 构建，通过 `MultiProvider` 支持 100+ 模型。这使开发者不被单一模型绑定，可以根据任务特性选择最优模型（成本/速度/能力权衡）。

### 4.2 Handoff 作为工具的优雅设计

将 Agent 间的交接建模为一种**特殊 Tool**（而非内部跳转），模型可以在生成响应时**主动决定**何时交接，而不是由框架硬编码决策逻辑。这种设计让交接逻辑变成 LLM 的推理能力，而非工程代码。

### 4.3 完全可序列化的 RunState

通过 `run_state.py` 的精心设计，整个运行状态（包括对话历史、当前 Agent、上下文、Tool 调用结果）可以被完整序列化。这使得：
- Human-in-the-Loop 审批成为可能
- Agent 运行可以被中断、暂停、恢复
- 支持分布式/跨进程的执行

### 4.4 细粒度的 Lifecycle Hooks

`RunHooks` 和 `AgentHooks` 提供了每个阶段的订阅点：
- `on_agent_start / on_agent_end`
- `on_tool_start / on_tool_end`
- `on_turn_start / on_turn_end`
- `on_handoff`

这比单纯的"before/after"拦截器更细粒度，允许在工作流中插入任意自定义逻辑。

### 4.5 Sandbox Agent 的 Manifest 声明式设计

不直接给 Agent shell 访问权限，而是通过 `Manifest` 声明需要访问的 Git 仓库、本地文件、数据库等，运行时按需 materialization。这种**最小权限**设计比直接文件访问更安全可控。

---

## 五、源码研读要点

### 5.1 必读文件清单（按优先级）

| 优先级 | 文件 | 理由 |
|--------|------|------|
| ⭐⭐⭐ | `run.py` | 理解 Runner 入口和 run 生命周期 |
| ⭐⭐⭐ | `run_internal/run_loop.py` | 理解主循环编排的核心逻辑 |
| ⭐⭐⭐ | `agent.py` | 理解 Agent 抽象和配置 |
| ⭐⭐⭐ | `run_state.py` | 理解可恢复状态的设计 |
| ⭐⭐ | `tool.py` | 理解工具建模和执行 |
| ⭐⭐ | `run_internal/tool_execution.py` | 理解工具执行引擎 |
| ⭐⭐ | `models/openai_responses.py` | 理解 OpenAI API 集成 |
| ⭐ | `handoffs/__init__.py` | 理解交接机制 |
| ⭐ | `memory/session.py` | 理解会话历史管理 |
| ⭐ | `sandbox/runtime.py` | 理解沙箱执行编排 |

### 5.2 类型系统设计

框架大量使用 Python 3.12+ 的泛型和类型别名：

```python
TContext = TypeVar("TContext")
TResponseInputItem = TypeAlias = "ResponseInputItemParam"
MaybeAwaitable: TypeAlias = "T | Awaitable[T]"
```

Pydantic v2 的 `BaseModel` 被广泛用于配置和输出建模，确保类型安全。TypeAdapter 用于运行时类型验证。

### 5.3 错误处理模式

框架采用**分层错误处理**：
- `AgentsException` 基类
- `UserError`：用户输入错误
- `ModelBehaviorError`：模型行为异常
- `MaxTurnsExceeded`：超过最大轮次
- `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered`：护栏触发

通过 `RunErrorHandlers` 链式处理，可在任意错误发生时执行自定义恢复逻辑。

---

## 六、横向对比与启示

### 6.1 与 OpenClaw 的设计对比

| 维度 | OpenAI Agents SDK | OpenClaw |
|------|-------------------|----------|
| 核心抽象 | `Agent` + `Runner` | `Agent` + Session/Channel |
| 工具系统 | FunctionTool + MCP | FunctionTool + MCP + 原生工具 |
| 会话管理 | Session 协议（多种后端） | 直接依赖 Channel |
| 沙箱执行 | Sandbox Agent + Manifest | OpenClaw Browser / Node |
| 交接机制 | Handoff as Tool | Skill / Subagent |
| 可观测性 | 内置 Tracing Span | 通过 Gateway 插件 |
| 类型系统 | Pydantic v2 + 泛型 | Python 原生类型 |

**核心差异**：OpenAI Agents SDK 是**库（Library）模式**，适合嵌入到现有 Python 应用中；OpenClaw 是**应用/框架混合模式**，内置完整的运行时环境。

### 6.2 设计启示

1. **RunState 可序列化设计**值得借鉴：OpenClaw 的 Gateway 调度如果能引入类似的状态快照机制，可实现更灵活的任务中断/恢复。

2. **Handoff as Tool** 是一种聪明的抽象——把"多 Agent 协作决策"交给 LLM 推理，而非硬编码 switch/case。

3. **Manifest 声明式沙箱**比直接文件访问更安全，适合 OpenClaw 的 Skill 执行环境。

4. **MultiProvider 模型抽象**的 Provider-Agnostic 思路，可以指导 OpenClaw 更好地抽象模型层。

---

## 七、参考示例

```python
# 最简 Agent 示例
from agents import Agent, Runner

agent = Agent(
    name="Assistant",
    instructions="你是一个乐于助人的助手。",
)

result = Runner.run_sync(agent, "你好，请介绍一下自己")
print(result.final_output)

# 带工具的 Agent
from agents import Agent, Runner, FunctionTool

def get_weather(city: str) -> str:
    return f"{city} 今天晴天，25度"

weather_tool = FunctionTool.from_defaults(
    fn=get_weather,
    name="get_weather",
    description="获取城市天气"
)

agent = Agent(
    name="Weather Assistant",
    instructions="你是一个天气助手，用工具回答问题。",
    tools=[weather_tool],
)

result = Runner.run_sync(agent, "北京今天天气怎么样？")
print(result.final_output)

# 多 Agent 交接
from agents import Agent, Runner, Handoff

triage = Agent(name="Triage", instructions="分类用户问题...")
billing = Agent(name="Billing", instructions="处理账单问题...")
account = Agent(name="Account", instructions="处理账户问题...")

billing_handoff = Handoff(
    tool_name="transfer_to_billing",
    tool_description="转接到账单部门",
    on_invoke_handoff=lambda ctx, _: billing,
    agent_name="Billing",
)
# ...
```

---

*报告由 OpenClaw 自动生成 | 数据来源：GitHub trending 2026-04-17 | 模型：openai-agents-python v0.14.1*
