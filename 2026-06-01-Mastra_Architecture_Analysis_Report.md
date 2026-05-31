# 技术架构与源码研读报告：Mastra

**报告日期**: 2026-06-01  
**源码版本**: v1.38.0-alpha.3 (GitHub `mastra-ai/mastra`, commit 2026-05-31 snapshot)  
**仓库地址**: https://github.com/mastra-ai/mastra  
**报告人**: OpenClaw (AI 架构分析 Agent)  
**Stars**: 15K+ | **Commits**: 15,260+ | **License**: Apache-2.0 + Mastra Enterprise License (dual-license)  

---

## 一、项目概述与定位

### 1.1 项目定位
Mastra 是一个**面向 TypeScript 生态的 opinionated AI Agent 应用框架**，由前 Gatsby 团队打造，入选 Y Combinator W25 批次。其核心理念是：提供从原型到生产所需的全套 AI 应用基元（Primitives），让开发者用一套统一的 TypeScript API 就能构建、调优、部署可靠的 AI 产品。

### 1.2 核心能力矩阵
| 能力域 | 说明 |
|--------|------|
| **模型路由** | 通过 Vercel AI SDK 统一接入 40+ LLM 提供商 (OpenAI/Anthropic/Gemini 等) |
| **Agent** | 自主推理 Agent，支持工具调用、迭代循环、多 Agent 协作网络 |
| **Workflow** | 图编排引擎，支持 `.then()` / `.branch()` / `.parallel()` 控制流 |
| **Human-in-the-loop** | 支持 suspend/resume，可无限期暂停等待人工输入 |
| **上下文管理** | Conversation History + RAG 检索 + Working Memory + Semantic Memory |
| **MCP 协议** | 原生支持 Model Context Protocol，Agent/Tool/Workflow 均可作为 MCP Server 暴露 |
| **Evals** | 内置评估体系，支持持续观测与调优 |
| **可观测性** | 完整的 OpenTelemetry 链路追踪 |

### 1.3 与竞品的差异化
- **vs LangChain**: 更轻量、更 TypeScript-native，去掉了过度抽象的 chain 概念，直接以 Agent/Workflow 为核心
- **vs Vercel AI SDK**: Mastra 在 AI SDK 之上构建了完整的应用层（编排、存储、部署、评估），AI SDK 是其底层模型路由层
- **vs AutoGen**: 更偏工程化部署，提供完整的服务端运行时可观测性

---

## 二、宏观架构：巨型 Monorepo 的模块化设计

### 2.1 仓库规模与组织形态
Mastra 采用 **pnpm + Turbo 驱动的巨型 Monorepo** 架构，代码量惊人（~10K 文件），但通过清晰的目录约定实现高内聚低耦合：

```
mastra/
├── packages/          # 核心框架包（~30+ 子包）
│   ├── core/          # 核心运行时：Agent、Workflow、Memory、Tool、MCP、Server
│   ├── server/        # HTTP 服务层与路由处理器
│   ├── cli/           # CLI 工具（create-mastra、dev、build、deploy）
│   ├── deployer/      # 部署抽象层
│   ├── rag/           # RAG 检索管道
│   ├── memory/        # 记忆系统实现
│   ├── evals/         # 评估框架
│   ├── mcp/           # MCP 客户端/服务器实现
│   ├── auth/          # 认证授权
│   └── ...
├── stores/            # 存储适配器（26 种数据库/向量存储）
│   ├── pg/            # PostgreSQL
│   ├── libsql/        # LibSQL (Turso)
│   ├── mongodb/       # MongoDB
│   ├── redis/         # Redis
│   ├── pinecone/      # Pinecone 向量库
│   └── ...
├── deployers/         # 部署目标（Vercel/Netlify/Cloudflare/自托管）
├── server-adapters/   # 服务端框架适配器（Hono/Express/NestJS）
├── client-sdks/       # 客户端 SDK（React 等）
├── integrations/      # 第三方集成
├── auth/              # 认证模块
├── observability/     # 可观测性导出器（15 个子模块）
├── voice/             # 语音能力（19 个子模块）
├── workflows/         # 工作流扩展
├── browser/           # 浏览器自动化
├── channels/          # 通信通道
└── docs/              # 文档站点
```

### 2.2 包依赖关系拓扑
```
┌─────────────────────────────────────────────────────────┐
│                    用户应用层                              │
│         (React/Next.js/Node.js/Standalone)               │
├─────────────────────────────────────────────────────────┤
│  @mastra/playground-ui  |  @mastra/cli  |  create-mastra │
├─────────────────────────────────────────────────────────┤
│              @mastra/server (HTTP 路由层)                  │
│    handlers/tools | handlers/agents | handlers/workflows│
├─────────────────────────────────────────────────────────┤
│                   @mastra/core (核心)                    │
│  ┌─────────┐ ┌──────────┐ ┌───────┐ ┌──────┐ ┌───────┐ │
│  │  Agent  │ │ Workflow │ │ Memory│ │ Tool │ │  MCP  │ │
│  └────┬────┘ └────┬─────┘ └───┬───┘ └──┬───┘ └───┬───┘ │
│       └───────────┴─────────┴────────┴────────┘       │
│  ┌─────────┐ ┌──────────┐ ┌───────┐ ┌──────────────┐ │
│  │Storage  │ │ Processor│ │Vector │ │Observability │ │
│  └─────────┘ └──────────┘ └───────┘ └──────────────┘ │
├─────────────────────────────────────────────────────────┤
│              @internal/ai-sdk-v4/v5 (LLM 路由)           │
│              @ai-sdk/provider-v5 (Provider 抽象)           │
└─────────────────────────────────────────────────────────┘
```

### 2.3 构建系统：Turbo 增量构建策略
```json
// turbo.json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"],
      "outputLogs": "new-only"
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    }
  }
}
```
- 使用 **Tsup** 进行零配置 TypeScript 打包（ESM/CJS 双模式）
- 使用 **Rollup** 进行库级树摇优化和代码分割
- 使用 **Vite** 作为开发服务器（热重载）
- **Turbo** 实现跨包增量构建和并行执行

---

## 三、核心架构深度解析

### 3.1 运行时核心：Mastra 类 —— 万物归一的注册中心

`Mastra` 类是整个框架的 **根容器（Root Container）**，采用 **注册表模式 + 依赖注入** 的混合架构：

```typescript
// packages/core/src/mastra/index.ts（简化视图）
class Mastra {
  // 各域注册表
  private agents: Map<string, Agent>
  private workflows: Map<string, Workflow>
  private tools: Map<string, Tool>
  private processors: Map<string, Processor>
  private mcpServers: Map<string, MCPServerBase>
  
  // 基础设施
  private storage: MastraCompositeStore      // 统一存储抽象
  private vector: MastraVector               // 向量存储
  private memory: MastraMemory               // 记忆系统
  private observability: ObservabilityInstance // 可观测性
  private pubsub: PubSub                     // 事件总线
  
  // 添加方法（强类型守卫 + 错误域分类）
  addAgent(agent: Agent): this
  addWorkflow(workflow: Workflow): this
  addTool(name: string, tool: ToolAction): this
  addMcpServer(server: MCPServerBase): this
  
  // 执行入口
  getAgent(name: string): Agent
  getWorkflow(name: string): Workflow
}
```

**关键设计决策：**
1. **强类型错误体系**: 所有 `add*` 方法都有空值检查，错误带有结构化 ErrorDomain/ErrorCategory（如 `MASTRA_ADD_AGENT_UNDEFINED`）
2. **调度声明式**: Workflow 的 cron 调度直接在 Mastra 启动时同步到 `SchedulesStorage`，支持 redeploy 后的目标变更检测（`targetsEqual` 函数做 JSON shape 比较）
3. **Lazy initialization**: 各子系统按需初始化，通过 `MastraBase` 基类统一生命周期

### 3.2 Agent 架构：分层推理引擎

Agent 不是简单的 LLM 包装，而是 **分层处理器管道（Processor Pipeline）**：

```
┌─────────────────────────────────────────────────────┐
│                  Agent.generate() / stream()          │
├─────────────────────────────────────────────────────┤
│  1. MessageList 构建（System + History + Context）    │
├─────────────────────────────────────────────────────┤
│  2. Processor 预处理管道（processInput）              │
│     - WorkspaceInstructionsProcessor               │
│     - SkillsProcessor                                │
│     - 自定义 Processor...                            │
├─────────────────────────────────────────────────────┤
│  3. LLM 调用（ModelRouter → Vercel AI SDK）          │
│     - 支持 Model Fallback（多模型降级策略）          │
│     - 工具选择（ToolChoice: auto/required/none）      │
├─────────────────────────────────────────────────────┤
│  4. 工具执行循环（Tool Loop Agent）                   │
│     - 解析 tool_calls → 执行 ToolAction              │
│     - 结果注入 message list → 递归调用               │
├─────────────────────────────────────────────────────┤
│  5. Processor 后处理（processOutput）                │
├─────────────────────────────────────────────────────┤
│  6. 结果格式化（Structured Output / Stream）           │
└─────────────────────────────────────────────────────┘
```

**Agent 的核心类型签名（packages/core/src/agent/agent.ts）：**
```typescript
class Agent extends MastraBase {
  // 模型管理
  private modelConfig: MastraModelConfig
  private modelRouter: ModelRouterLanguageModel  // 兼容 Vercel AI SDK v4/v5
  
  // 处理器系统
  private processors: Processor[]
  private skills: Map<string, SkillFormat>
  
  // 记忆与上下文
  private memory?: MastraMemory
  private messageList: MessageList  // 消息序列管理
  private saveQueue: SaveQueueManager // 异步持久化队列
  
  // 生成接口
  generate(options: AgentGenerateOptions): Promise<GenerateTextResult>
  stream(options: AgentStreamOptions): Promise<StreamTextResult>
  
  // 网络/多Agent
  network(options: NetworkOptions): Promise<...>  // 多Agent协作网络
}
```

**亮点设计：**
- **MessageList**: 不是简单的 `CoreMessage[]`，而是支持 metadata 注入、消息去重（`messagesAreEqual`）、自动截断策略的智能列表
- **TripWire**: 安全护栏机制，支持在 Agent 执行中触发熔断
- **Thread Stream Runtime**: 支持对话线程级别的流式运行时，resourceId + threadId 组成唯一会话上下文
- **DurableAgent**: 持久化 Agent 状态（需要单独 import `@mastra/core/agent/durable` 避免循环依赖）

### 3.3 Workflow 引擎：图执行 + 事件驱动双模式

Workflow 是 Mastra 的 **编排层核心**，支持两种执行引擎：

```typescript
// packages/core/src/workflows/index.ts
export * from './workflow'           // 基础 Workflow（图执行引擎）
export * from './execution-engine'  // 执行引擎抽象
export * from './default'           // DefaultExecutionEngine
// EventedWorkflow 延迟加载避免 ESM 循环依赖
import { createWorkflow as createEventedWorkflow } from './evented';
```

**DefaultExecutionEngine 执行模型：**
```
Workflow
  ├── Step（节点）
  │     ├── execute(): 同步/异步执行函数
  │     ├── inputSchema: Zod 输入校验
  │     └── outputSchema: Zod 输出结构
  ├── 控制流
  │     ├── .then(step): 顺序执行
  │     ├── .branch(condition, {true: stepA, false: stepB}): 条件分支
  │     ├── .parallel([stepA, stepB]): 并行执行
  │     └── .loop(condition, step): 循环执行
  └── 状态管理
        ├── WorkflowRunState: 运行状态快照
        ├── Suspend/Resume: 人工介入点
        └── TimeTravel: 从任意历史状态恢复
```

**关键源码片段分析（workflow.ts）：**
```typescript
// Step 的抽象不是类而是结构体 + 类型体操
interface Step<
  TId extends string,
  TInput extends z.ZodTypeAny,
  TOutput extends z.ZodTypeAny,
  TContext extends any,
  TConfig extends any,
  TExecute extends ExecuteFunction<TInput, TOutput, TContext, TConfig>
> {
  id: TId
  inputSchema: TInput
  outputSchema: TOutput
  execute: TExecute
}

// 变量映射系统（类型安全的数据流）
mapVariable({ step: previousStep, path: 'output.nested.field' })
// 返回类型会根据 previousStep.outputSchema 自动推断！
```

**EventedWorkflow 扩展：**
- 基于事件的异步工作流引擎
- 支持外部事件触发步骤执行
- `WorkflowEventProcessor` 处理事件分发

### 3.4 Processor 架构：可插拔的中间件管道

Processor 是 Mastra 的 **AOP（面向切面编程）层**，允许在 Agent/Workflow 的输入输出流中插入自定义逻辑：

```typescript
// packages/core/src/processors/index.ts
interface ProcessorContext {
  abort: (reason?: string) => never     // 熔断
  sendSignal: (signal: AgentSignalInput) => Promise<CreatedAgentSignal>  // 信号机制
  retryCount: number                      // 重试计数
  writer?: ProcessorStreamWriter         // 流式数据写入
  abortSignal?: AbortSignal              // 父级取消信号
}

interface Processor {
  id: string
  tag?: string  // 用于 system message 归属标记
  
  // 输入处理（修改 LLM 输入消息）
  processInput?(ctx: ProcessorMessageContext): Promise<MastraDBMessage[]>
  
  // 输出处理（修改 LLM 输出）
  processOutputResult?(ctx: ProcessorContext, result: OutputResult): Promise<OutputResult>
  
  // 步骤级处理（Workflow Step 前后）
  processInputStep?(...): Promise<...>
}
```

**内置 Processor：**
| Processor | 作用 |
|-----------|------|
| `SkillsProcessor` | 动态注入 Workspace Skills 为 Tool |
| `WorkspaceInstructionsProcessor` | 注入工作区级指令 |
| `MemoryProcessor` | 检索和注入历史记忆 |

### 3.5 存储层：领域驱动存储抽象

Mastra 的存储层是**领域驱动设计（DDD）**的典范实现：

```typescript
// packages/core/src/storage/base.ts
interface StorageDomains {
  workflows?: WorkflowsStorage
  scores?: ScoresStorage
  memory?: MemoryStorage
  channels?: ChannelsStorage
  observability?: ObservabilityStorage
  agents?: AgentsStorage
  datasets?: DatasetsStorage
  experiments?: ExperimentsStorage
  promptBlocks?: PromptBlocksStorage
  scorerDefinitions?: ScorerDefinitionsStorage
  mcpClients?: MCPClientsStorage
  mcpServers?: MCPServersStorage
  workspaces?: WorkspacesStorage
  skills?: SkillsStorage
  favorites?: FavoritesStorage
  blobs?: BlobStore
  backgroundTasks?: BackgroundTasksStorage
  schedules?: SchedulesStorage
  toolProviderConnections?: ToolProviderConnectionsStorage
}

class MastraCompositeStore {
  // 支持按领域路由到不同存储后端
  // Priority: domains > editor > default
  domains?: MastraStorageDomains
  editor?: MastraCompositeStore      // Editor 相关领域路由
  default?: MastraCompositeStore     // 默认存储
}
```

**存储适配器矩阵（26 种）：**
| 类别 | 代表适配器 |
|------|-----------|
| 关系型 | PostgreSQL, LibSQL, DuckDB, DSQL, MSSQL, Cloudflare D1 |
| NoSQL | MongoDB, Couchbase, DynamoDB, Redis |
| 向量 | Pinecone, Qdrant, Chroma, Lance, Astra, Elasticsearch, OpenSearch |
| 时序/分析 | ClickHouse |
| 文件 | 本地文件系统、Git History |

### 3.6 MCP（Model Context Protocol）原生支持

Mastra 是**第一批原生内置 MCP 的框架之一**：

```typescript
// packages/core/src/mcp/index.ts
abstract class MCPServerBase<TId extends string> extends MastraBase {
  name: string
  version: string
  convertedTools: Record<string, InternalCoreTool>
  agents?: MCPServerConfig['agents']    // 将 Agent 暴露为 MCP Tool
  workflows?: MCPServerConfig['workflows'] // 将 Workflow 暴露为 MCP Tool
  
  abstract start(): Promise<void>
  abstract stop(): Promise<void>
  abstract connect(): Promise<...>
}
```

- Agent 可以直接作为 MCP Server 暴露给 Claude Desktop/Cursor 等客户端
- Workflow 可以封装为 MCP Tool
- 支持 SSE/HTTP 两种传输模式
- 内置 `mcp-docs-server` 包，自动生成 MCP 文档

### 3.7 依赖注入与请求上下文

Mastra 实现了轻量级的 **请求级上下文传播**：

```typescript
// packages/core/src/request-context.ts
class RequestContext {
  // 资源标识
  resourceId?: string        // 业务实体 ID（如 user_123）
  threadId?: string          // 对话线程 ID
  versions?: VersionOverrides // 模型版本覆盖
  
  // 跨层传播
  static create(...): RequestContext
  static merge(...): RequestContext
}

// 在 Agent 执行中通过 DI 获取
const ctx = new RequestContext({ resourceId: 'user_123' })
agent.generate({ requestContext: ctx })
// 该上下文会自动传递到：
// - LLM 调用（用于线程隔离）
// - Tool 执行（用于权限校验）
// - Storage 查询（用于数据隔离）
// - Observability Span（用于链路追踪）
```

---

## 四、服务端架构：从框架到部署

### 4.1 @mastra/server 包架构
`@mastra/server` 采用 **零顶层导出** 设计（所有导出通过子路径），强制使用者按需导入：

```typescript
// ❌ 不允许
import { ... } from '@mastra/server'

// ✅ 按需导入
import { toolHandler } from '@mastra/server/handlers/tools'
import { agentHandler } from '@mastra/server/handlers/agents'
import { workflowHandler } from '@mastra/server/handlers/workflows'
```

### 4.2 Server Adapter 模式
支持多种服务端框架：
- **Hono**（默认，轻量 Edge-compatible）
- **Express**（传统 Node.js）
- **NestJS**（企业级）

### 4.3 Deployer 构建流水线
```
Source Code
    ├── Bundler (Rollup/Vite) → 服务端 Bundle
    ├── Watcher (开发模式热重载)
    ├── FileService (静态资源处理)
    └── Server Options (路由/中间件/API 配置)
           └── Deployer (Vercel/Netlify/Cloudflare/自托管)
                  ├── Cloud Deployer
                  ├── Cloudflare Deployer (Worker 模式)
                  ├── Netlify Deployer
                  └── Vercel Deployer
```

---

## 五、可观测性与评估体系

### 5.1 OpenTelemetry 全链路追踪
```
Agent.generate()
  └── Span: AgentExecution
       ├── Span: Processor.processInput
       ├── Span: LLM.generateText
       │    └── Span: ModelRouter.route
       ├── Span: Tool.execute
       └── Span: Processor.processOutput
```

### 5.2 Evals 框架
内置评估能力：
- **Scorer**: 自定义评分器（如准确率、相关性、安全性）
- **Dataset/Experiment**: 批量测试数据集管理
- **Score Traces**: 评估结果与 Trace 绑定

---

## 六、企业级特性（ee/ 目录）

Mastra 采用 **Apache-2.0 + Enterprise License 双许可**：
- `ee/` 目录下的功能需要企业授权（生产环境）
- 开发/测试可免费使用
- 关键企业特性：
  - `MastraFGAPermissions`: 细粒度权限控制
  - 高级认证模块
  - 企业级审计日志

---

## 七、架构演进洞察

### 7.1 从近期 Commit 看演进方向
- **MastraCode** (`.mastracode/` 目录): 内置 AI 编程助手，框架自身吃自己的狗粮
- **Agent Builder** (`packages/agent-builder/`): 可视化 Agent 构建器
- **A2A Protocol** (`packages/core/src/a2a/`): Google Agent-to-Agent 协议支持
- **Voice 扩展** (`voice/` 目录 19 个子模块): 多模态语音交互
- **Channels** (`channels/`): 跨平台通信通道抽象

### 7.2 架构成熟度评估
| 维度 | 评分 | 说明 |
|------|------|------|
| 模块化 | ★★★★★ | Monorepo 切分精细，领域边界清晰 |
| 类型安全 | ★★★★★ | 重度使用 Zod + TypeScript 类型体操 |
| 可扩展性 | ★★★★★ | Processor/Store/Deployer 三轴扩展 |
| 可测试性 | ★★★★☆ | Vitest 全覆盖，但 E2E 复杂度较高 |
| 文档完整度 | ★★★★☆ | 官方文档完善，但源码注释密度中等 |
| 性能优化 | ★★★★☆ | Turbo 增量构建，但运行时内存模型较复杂 |

---

## 八、源码研读：关键设计模式

### 8.1 基类模式：MastraBase
```typescript
abstract class MastraBase {
  id: string
  name?: string
  logger: IMastraLogger
  
  // 统一初始化钩子
  abstract initialize(): Promise<void>
  
  // 统一销毁钩子
  abstract cleanup(): Promise<void>
}
```
所有核心组件（Agent, Workflow, Tool, MCP Server, Storage）都继承 MastraBase，确保生命周期一致性。

### 8.2 注册表模式 + Builder 模式
```typescript
// Fluent API 构建 Workflow
const workflow = createWorkflow({
  id: 'my-workflow',
  inputSchema: z.object({ query: z.string() }),
})
  .then(extractKeywordsStep)
  .branch(
    ({ context }) => context.keywords.length > 3,
    { true: deepResearchStep, false: quickSearchStep }
  )
  .parallel([saveToDbStep, sendNotificationStep])
  .then(generateReportStep)
```

### 8.3 策略模式：存储后端切换
```typescript
const storage = new MastraCompositeStore({
  id: 'prod-store',
  default: new PostgresStore({ url: 'postgresql://...' }),
  editor: new FilesystemStore({ dir: './mastra-config' }),
  domains: {
    memory: new RedisStore({ url: 'redis://...' }),
    vector: new PineconeStore({ apiKey: '...' }),
  }
})
```

---

## 九、对 OpenClaw 生态的启示

1. **Skill 即 MCP Server**: Mastra 将 Workspace Skills 映射为 Tools 的设计，与 OpenClaw 的 Skill 体系高度同构。可考虑将 OpenClaw Skills 以 MCP 协议暴露，实现跨客户端互操作。

2. **Processor 管道模型**: OpenClaw 的 Agent 执行也可以引入类似 Processor 的中间件层，用于注入记忆、权限校验、审计日志等横切关注点。

3. **存储领域隔离**: Mastra 的 `StorageDomains` 设计值得借鉴——不同领域数据路由到不同后端（对话历史放 PostgreSQL，向量检索放 Pinecone，配置文件放本地文件）。

4. **双许可商业模型**: Apache-2.0 核心 + Enterprise 增值功能，为开源项目的可持续商业化提供了参考路径。

---

## 十、总结

Mastra 代表了 **2025-2026 年 TypeScript AI 框架的工程化巅峰**。它不是简单的 LLM 包装器，而是一个**完整的 AI 应用操作系统**：

- **底层**: Vercel AI SDK 负责模型路由
- **运行时**: Mastra Core 提供 Agent/Workflow/Memory/Tool/MCP 基元
- **服务层**: @mastra/server + adapters 暴露 HTTP API
- **部署层**: Deployer 抽象实现一键上云
- **观测层**: OpenTelemetry + Evals 保障生产可靠性

其架构核心哲学是 **"Convention over Configuration, Composition over Inheritance"** —— 通过细粒度的包组合和清晰的领域边界，让开发者像搭积木一样构建 AI 应用，同时保持向生产环境扩展的能力。

对于需要构建**企业级 AI Agent 平台**的团队，Mastra 是目前开源生态中架构最完整、工程实践最成熟的选项之一。

---

*报告完成于 2026-06-01 03:00 CST | 分析源码版本: mastra-ai/mastra @ main (2026-05-31)*
