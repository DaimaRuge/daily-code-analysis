# Mastra 深度架构分析报告

> 2026-04-28 | TypeScript AI Agent 框架架构逆向工程
> 数据来源：mastra.ai 官方文档、GitHub 源码、npm 包分析
> 参考对比：VoltAgent 架构（同领域 TS Agent 框架）

---

## 一、项目概览

**Mastra** 是一个开源 TypeScript AI Agent 框架，由 Gatsby 团队创建。

| 维度 | 数据 |
|------|------|
| 团队 | Gatsby 创始团队 |
| 成立 | 2024年10月（Beta），v1.0 于 2026年1月20日 |
| 融资 | Y Combinator W25，$13M |
| GitHub Stars | 22k+ |
| npm 周下载量 | 300k+ |
| 许可证 | Apache 2.0 |
| 语言 | TypeScript（严格模式） |
| 构建工具 | pnpm workspace + Turbo (Turborepo) |
| 测试 | Vitest |
| 社区 | Discord、YouTube (97+ 视频) |

**核心理念**："Python trains, TypeScript ships." — 面向全栈 TypeScript 开发者。

---

## 二、Monorepo 架构

```
mastra/
├── packages/
│   ├── core/                    ← @mastra/core（核心运行时）
│   ├── cli/                     ← Mastra CLI
│   ├── memory/                  ← @mastra/memory（记忆系统）
│   ├── rag/                     ← @mastra/rag（文档处理+RAG）
│   ├── observability/           ← @mastra/observability（可观测性）
│   ├── mcpx/                    ← MCP 客户端工具封装
│   ├── playground/              ← 开发调试面板
│   └── ...更多
├── stores/                      ← 存储适配器
│   ├── libsql/                  ← LibSQL/SQLite
│   ├── pg/                      ← PostgreSQL (pgvector)
│   ├── cloudflare-d1/           ← Cloudflare D1
│   ├── upstash/                 ← Upstash Redis
│   └── ...更多
├── deployers/                   ← 部署器
│   ├── vercel/                  ← Vercel Edge
│   ├── cloudflare/              ← Cloudflare Workers
│   └── netlify/                 ← Netlify
├── integrations/                ← 第三方集成
├── voice/                       ← 语音支持
├── server-adapters/             ← 服务器适配器
├── browser/                     ← 浏览器自动化
├── client-sdks/                 ← 客户端 SDK
├── ee/                          ← 企业版功能
├── auth/                        ← 认证模块
├── pubsub/                      ← 消息队列
├── templates/                   ← 项目模板
└── examples/                    ← 示例项目（40+）
```

### 包对比（Mastra vs VoltAgent）

| 维度 | Mastra | VoltAgent |
|------|--------|-----------|
| Monorepo 工具 | Turbo | Lerna + Nx |
| 包管理器 | pnpm | pnpm |
| 核心包 | `@mastra/core` | `@voltagent/core` |
| 存储层 | `stores/`（独立顶级目录） | `packages/*`（适配器在核心旁） |
| 部署器 | `deployers/`（Vercel/CF/Netlify） | VoltOps 托管 |
| MCP 支持 | `mcpx/` 内置封装 | MCPConfiguration 内建 |
| 企业版 | `ee/`（部分闭源） | VoltOps Console |

---

## 三、核心运行时

### 3.1 Mastra 实例

```typescript
import { Mastra } from '@mastra/core'

export const mastra = new Mastra({
  agents: { testAgent, codeAgent },
  workflows: { testWorkflow },
  storage: new LibSQLStore({ url: 'file:./mastra.db' }),
  observability: new Observability({...}),
  server: { /* server config */ },
  logger: { level: 'info' },
  telemetry: { /* telemetry config */ },
})
```

**关键设计**：集中式 `Mastra` 实例管理一切 — agents、workflows、storage、observability、logger、server。所有 agent 通过 `mastra.getAgentById()` 获取，确保共享服务注入。

### 3.2 Agent 定义

```typescript
import { Agent } from '@mastra/core/agent'

const agent = new Agent({
  id: 'my-agent',                          // 唯一标识
  name: 'My Agent',                        // 显示名称
  instructions: 'You are a helpful...',    // 系统提示词
  model: 'openai/gpt-5.4',               // 模型路由字符串
  
  // 可选能力
  tools: {},                               // 工具集
  memory: new Memory({...}),              // 记忆配置
  voice: {},                               // 语音
  guardrails: {},                          // 护栏
  processors: {},                          // 消息处理器
  
  // 动态配置
  dynamicConfig: {
    defaultModel: (request) => request.model || defaultModel,
    defaultInstructions: (request) => {...},
    defaultTools: (request) => {...},
  },
})
```

### 3.3 Agent 调用方式

| 方法 | 用途 | 流式 |
|------|------|------|
| `agent.generate(prompt)` | 完整响应 | ❌ |
| `agent.stream(prompt)` | 实时流式 | ✅ |
| `agent.generateLegacy(prompt)` | 传统格式 | ❌ |
| `agent.streamLegacy(prompt)` | 传统流式 | ✅ |

**响应结构**：
```typescript
const response = await agent.generate('...')
console.log(response.text)        // 文本
console.log(response.toolCalls)   // 工具调用
console.log(response.toolResults) // 工具结果
console.log(response.steps)       // 步骤详情
console.log(response.usage)       // Token 用量
```

---

## 四、Workflow 引擎

### 4.1 设计哲学

Mastra 采用**图式 Workflow**：`createStep()` + `createWorkflow().then(step1).then(step2).commit()`。

与 VoltAgent 的声明式链式 API（`createWorkflowChain().andThen().andAgent()`）不同，Mastra 的 `createStep()` 和 `createWorkflow()` 是分离的概念。

```typescript
import { createWorkflow, createStep } from "@mastra/core/workflows";
import { z } from "zod";

const step1 = createStep({
  id: 'step-1',
  inputSchema: z.object({ message: z.string() }),
  outputSchema: z.object({ formatted: z.string() }),
  execute: async ({ inputData }) => ({
    formatted: inputData.message.toUpperCase()
  }),
});

export const workflow = createWorkflow({
  id: "test-workflow",
  inputSchema: z.object({ message: z.string() }),
  outputSchema: z.object({ output: z.string() })
})
  .then(step1)
  .commit();
```

### 4.2 控制流能力

| 方法 | 功能 |
|------|------|
| `.then(step)` | 顺序执行 |
| `.branch(condition, trueStep, falseStep)` | 条件分支 |
| `.parallel(steps)` | 并行执行 |
| `.suspend()` / `.resume()` | 暂停/恢复（Human-in-the-loop） |
| `cloneWorkflow()` | 克隆复用 |
| `workflow.step()` | 嵌套 Workflow |

### 4.3 Workflow State

```typescript
const step = createStep({
  id: 'step-counter',
  stateSchema: z.object({ counter: z.number() }),
  execute: async ({ state, setState }) => {
    setState({ ...state, counter: state.counter + 1 })
    return { /* output */ }
  },
})
```

### 4.4 Workflow 架构对比

| 特性 | Mastra | VoltAgent |
|------|--------|-----------|
| 步骤定义 | `createStep()` 分离 | `andThen()` 内联 |
| Agent 调用 | 在 step 内 `mastra.getAgentById()` | 专用 `andAgent()` 方法 |
| Schema 库 | Zod / Valibot / ArkType | Zod |
| 分支语法 | `.branch(cond, t, f)` | `.andWhen(cond)` + `.andBranch()` |
| 挂起/恢复 | ✅ | ✅ |
| 时间旅行调试 | ✅ Studio "Time Travel" | ❌ |
| 嵌套 Workflow | ✅ `cloneWorkflow()` | ❌ 手动组合 |

---

## 五、Memory 系统

### 5.1 四层记忆模型

```
┌─────────────────────────────────────────┐
│         Message History（消息历史）        │
│    基础对话 → 短轮次 → 直接注入上下文      │
├─────────────────────────────────────────┤
│  Observational Memory（观察记忆）🆕       │
│  长对话 → 后台 Agent 压缩 → 密集观察日志   │
├─────────────────────────────────────────┤
│  Working Memory（工作记忆）               │
│  持久结构化数据：姓名、偏好、目标           │
├─────────────────────────────────────────┤
│  Semantic Recall（语义召回）              │
│  按语义检索历史消息，非关键词匹配           │
└─────────────────────────────────────────┘
```

### 5.2 Observational Memory（核心竞争力）

**问题**：长对话消息历史撑满上下文窗口

**方案**：
- 后台 agent 将旧消息压缩为密集观察
- 类似人类将细节记忆浓缩为"要点"
- 观察日志替代原始消息历史，保持上下文窗口小而精

```typescript
memory: new Memory({
  options: {
    observationalMemory: true,  // 启用观察记忆
  }
})
```

### 5.3 多 Agent 记忆隔离

Supervisor → Subagent 委派时：
- 每次委派创建新 `threadId`
- `resourceId` = `{parentResourceId}-{agentName}`（确定性）
- Subagent 继承 supervisor 的 Memory 实例或使用自己的
- Supervisor 上下文通过 `messageFilter` 选择性传递给 Subagent

### 5.4 Memory 对比

| 特性 | Mastra | VoltAgent |
|------|--------|-----------|
| 消息历史 | resource+thread 模式 | userId+conversationId |
| 观察记忆 | ✅ 后台 Agent 压缩 | ❌ |
| 工作记忆 | ✅ structured data | ✅ Markdown/JSON/Zod |
| 语义召回 | ✅ | ✅ 可选向量嵌入 |
| 记忆处理器 | ✅ filter/trim/prioritize | ❌ |
| 自动标题 | ❌ | ✅ auto title generation |
| Memory 隔离 | 自动？委派级别 | 手动配置 |

---

## 六、RAG 系统

Mastra 采用**显式管道**模型（而非 VoltAgent 的声明式 retriever 模式）：

```typescript
// 1. 文档初始化
const doc = MDocument.fromText('text...')

// 2. 分块
const chunks = await doc.chunk({ strategy: 'recursive', size: 512, overlap: 50 })

// 3. 嵌入
const { embeddings } = await embedMany({
  values: chunks.map(c => c.text),
  model: new ModelRouterEmbeddingModel('openai/text-embedding-3-small'),
})

// 4. 存储
await pgVector.upsert({ indexName: 'embeddings', vectors: embeddings })

// 5. 查询
const results = await pgVector.query({
  indexName: 'embeddings',
  queryVector: queryEmbedding,
  topK: 3,
})
```

**对比**：

| 维度 | Mastra | VoltAgent |
|------|--------|-----------|
| 模式 | 显式 5 步管道 | 声明式 retriever |
| 分块器 | 内建策略（recursive/sliding） | 15+ 种 Chunkers |
| 矢量数据库 | pgvector/Pinecone/Qdrant/MongoDB | Chroma/Pinecone/Qdrant/Weaviate |
| 托管 RAG | ❌ | ✅ VoltAgent KB |
| Agent 集成 | 显式注入 | 自动注入（retriever 属性） |
| Chain-of-Thought RAG | ✅ 示例 | ❌ |

---

## 七、Observability（可观测性）

### 7.1 三信号模型

```
Tracing（追踪）
  ├── 层级 Span 时间线
  └── 输入/输出/Token/时序
        ↓ 自动派生
Metrics（指标）
  ├── 持续时间
  ├── Token 消耗
  └── 成本估算
        ↓ 自动关联
Logging（日志）
  ├── 结构化日志
  └── traceId/spanId 自动标记
```

### 7.2 存储架构

```typescript
import { MastraCompositeStore } from '@mastra/core/storage'

new Mastra({
  storage: new MastraCompositeStore({
    default: new LibSQLStore({...}),     // 对话/记忆
    domains: {
      observability: new DuckDBStore(),  // 指标聚合（OLAP）
    },
  }),
})
```

**关键设计**：Composite Store 通过 domain 路由不同数据到不同后端：
- LibSQL → 对话数据、记忆
- DuckDB/ClickHouse → 可观测性（需要 OLAP 聚合查询）

### 7.3 Mastra Studio

可视化管理面板：
- 🔍 实时查看 agent/workflow 执行
- 🕐 **时间旅行调试**：回放单步、重试
- 📊 指标仪表盘
- 🎯 Graph 视图：Workflow 步骤可视化
- ⏸️ 暂停/恢复控制

**对比**：

| 维度 | Mastra | VoltAgent (VoltOps) |
|------|--------|---------------------|
| 追踪协议 | OpenTelemetry | OpenTelemetry |
| 外部导出 | Langfuse/Datadog/任意 OTLP | VoltOps 专有 |
| 本地调试 | Studio + DuckDB | Console |
| 时间旅行 | ✅ | ❌ |
| 敏感数据过滤 | ✅ SensitiveDataFilter | ❌ |
| 指标聚合 | DuckDB/ClickHouse | VoltOps 托管 |
| 开源程度 | 全部开源（Apache 2.0） | 核心开源 + VoltOps 闭源 |

---

## 八、部署与集成

### 8.1 部署器支持

| 平台 | 状态 |
|------|------|
| Vercel (Next.js) | ✅ 官方 |
| Cloudflare Workers | ✅ 官方 |
| Netlify | ✅ 官方 |
| Standalone (Node/Bun) | ✅ |
| Docker | ✅ |

### 8.2 通道支持

内建 Bot 集成：Slack、Discord、Telegram

### 8.3 模型提供商

40+ 模型提供商通过统一 Model Router 接口（OpenAI、Anthropic、Gemini、Groq、Together AI、Fireworks 等）

---

## 九、Mastra vs VoltAgent 全景对比

| 维度 | Mastra | VoltAgent |
|------|--------|-----------|
| **定位** | TS 全栈 AI 框架（React/Next.js 优先） | Agent 工程平台（observability-first） |
| **成熟度** | v1.0（2026.01）/ 22k stars | v2.0（2026） |
| **社区规模** | 300k+ 周下载 / 40+ 示例 | 较小但增长中 |
| **许可证** | Apache 2.0（全开源） | MIT 核心 + 闭源 VoltOps |
| **Workflow** | 图式（createStep + createWorkflow） | 声明式链式（createWorkflowChain） |
| **Agent 模型** | 独立 Agent + Supervisor | 独立 Agent + Supervisor |
| **记忆** | 4 层模型（Observational Memory 创新） | 对话 + 工作记忆 + 语义搜索 |
| **RAG** | 显式管道 | 声明式 retriever |
| **可观测性** | Studio + 任意 OTLP 导出 | VoltOps 专有 |
| **时间旅行** | ✅ | ❌ |
| **部署** | Vercel/CF/Netlify + 独立 | VoltOps 托管 |
| **Schema** | Zod/Valibot/ArkType | Zod（为主） |
| **MCP 支持** | ✅ mcpx 客户端 | ✅ Client + Server |
| **A2A 协议** | ❌ | ✅ |
| **Guardrails** | ✅ guardrails + processors | ✅ 输入/输出护栏 |
| **Human-in-loop** | ✅ suspend/resume | ✅ suspend/resume |
| **企业版** | ee/ 部分功能 | VoltOps Console |
| **前端集成** | Next.js/React 优先 | 任意前端（API 优先） |
| **Python** | ❌ TS only | ❌ TS only |

---

## 十、架构洞察

### 10.1 Mastra 的设计亮点

1. **Observational Memory**：这是 Mastra 最独特的设计。通过后台 Agent 压缩对话历史，解决长对话的上下文窗口问题。类似 MemGPT 的自动记忆管理，但集成度更高。

2. **Composite Store**：通过 domain 路由将不同类型数据（对话 vs 可观测性）分发到不同存储后端（OLTP vs OLAP），在框架层面解决了生产环境中数据分层存储的复杂性。

3. **时间旅行调试**：Workflow 执行后可以回放单个 Step，检查和重试。这个功能对复杂 Agent 系统的调试价值巨大。

4. **"Python trains, TypeScript ships"**：精准的市场定位 — 研究在 Python，产品在 TypeScript。直接对标 Next.js 生态。

5. **Turbo 构建 + pnpm workspace**：比 Lerna+Nx 的 VoltAgent 更现代化，增量构建更快。

### 10.2 潜在挑战

| 挑战 | 说明 |
|------|------|
| **Schema 库支持过多** | 同时支持 Zod/Valibot/ArkType 增加维护成本 |
| **API 稳定性** | v1.0 刚发布，Legacy API 尚存 |
| **包碎片化** | stores/deployers/integrations 各成独立包，版本同步复杂 |
| **Observational Memory 开销** | 后台 Agent 在每次调用时运行，增加延迟和成本 |
| **Evals 不成熟** | 评估系统比 VoltAgent 弱 |

### 10.3 选型建议

| 场景 | 推荐 |
|------|------|
| Next.js/React 全栈项目 | **Mastra** ⭐ |
| 需要长对话记忆（如客服机器人） | **Mastra**（Observational Memory） |
| 需要完整可观测性平台 | **Mastra**（Studio + 开放 OTLP） |
| 需要 Agent-as-a-Service 暴露 | VoltAgent（A2A Server） |
| 需要托管 RAG 服务 | VoltAgent（VoltAgent KB） |
| 需要部署到多平台 | **Mastra**（Vercel/CF/Netlify deployers） |
| 轻量原型 | **Mastra**（CLI templates 更快上手） |
| MCP Server 暴露 | VoltAgent（MCP Server 角色） |

---

## 十一、总结

Mastra 和 VoltAgent 代表了 TypeScript Agent 框架的两种路线：

- **Mastra**：前端优先，"Python trains, TypeScript ships"。面向 Next.js/React 开发者，强调开发体验（Studio、时间旅行、templates），记忆系统的 Observational Memory 是独有创新。

- **VoltAgent**：平台优先，"Build agents with full code control"。强调生产运维（VoltOps、deployment、observability），Workflow DSL 和 MCP Server 是差异优势。

从社区规模和产品成熟度看，Mastra 目前领先（22k stars vs VoltAgent 的 ~2k），但 VoltAgent 的 MCP 双角色支持和 A2A 协议更前瞻。

两个框架的竞争本质是**前端集成深度 vs 平台运维深度**的路线之争。

*报告基于 mastra.ai 官方文档、GitHub 源码、npm 分析。2026-04-28 编制。*
