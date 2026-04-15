# 技术架构与源码研读报告
## Open Agents（vercel-labs/open-agents）

> 分析日期：2026-04-15
> 源码版本：latest（GitHub trending 热门项目）
> 项目链接：https://github.com/vercel-labs/open-agents

---

## 一、项目概述

**Open Agents** 是 Vercel Labs 开源的一个参考应用，用于在 Vercel 平台上构建和运行后台编码 Agent。它是一个完整的"云端 AI 编程助手"系统，包含 Web UI、Agent 运行时、沙箱编排和 GitHub 集成，可以从自然语言提示直接生成代码变更，全程无需本地设备参与。

### 核心定位

- **不是一个黑盒** —— 项目明确说明"有意 fork 和适配，而非当作黑盒使用"
- **面向开发者** —— 需要理解系统架构才能有效定制
- **生产级参考** —— 代码质量高，覆盖完整的 AI Coding 闭环

---

## 二、整体架构：三层分离设计

项目的核心架构哲学体现在这段精炼的描述中：

```
Web → Agent workflow → Sandbox VM
```

### 三层职责

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: Web App (Next.js App Router)                 │
│  - 身份认证（Vercel OAuth + GitHub OAuth）              │
│  - 会话管理 + 聊天流式响应                              │
│  - Sandbox 连接生命周期管理                             │
│  - GitHub App 集成 + PR 创建                            │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  Layer 2: Agent Runtime (Durable Workflow)            │
│  - 基于 Vercel Workflow SDK 的持久化执行                │
│  - 多步 Tool Calling 循环                               │
│  - 子 Agent（explorer/executor/design）分派             │
│  - 上下文压缩与消息管理                                  │
│  - 自动 commit / 自动 PR（可选）                        │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  Layer 3: Sandbox VM (Vercel Sandbox)                  │
│  - 隔离的文件系统 + Shell 环境                          │
│  - Git / 分支操作                                       │
│  - 开发服务器 + 预览端口暴露                             │
│  - 基于快照的 Hibernate / Resume                        │
└─────────────────────────────────────────────────────────┘
```

### 关键架构决策：Agent 不运行在 Sandbox 内部

这是整个项目最重要的设计决策：

> **The agent is not the sandbox.**

Agent 进程运行在 Vercel 的 Durable Object / Workflow 环境中，通过**工具调用**（文件读写、Shell 命令、Git 操作）与 Sandbox 交互，而非直接运行在 VM 内部。

这带来了以下架构优势：
1. Agent 执行不绑定到单一 HTTP 请求生命周期
2. Sandbox 可独立休眠/恢复
3. 模型提供商和沙箱实现可独立演进
4. VM 保持为纯执行环境，不会演变成控制平面

---

## 三、代码目录结构

```
open-agents/
├── apps/
│   └── web/                    # Next.js 主应用
│       ├── app/
│       │   ├── api/            # API 路由
│       │   │   ├── auth/       # 认证（Vercel/GitHub OAuth）
│       │   │   ├── chat/       # 聊天 API（核心）
│       │   │   ├── github/     # GitHub App Webhook + API
│       │   │   └── sandbox/     # Sandbox 生命周期管理
│       │   ├── workflows/      # Vercel Durable Workflows
│       │   │   ├── chat.ts      # 主工作流（核心编排逻辑）
│       │   │   ├── chat-post-finish.ts  # 工作流结束后的自动操作
│       │   │   └── sandbox-lifecycle.ts  # Sandbox 生命周期
│       │   └── lib/
│       │       ├── db/         # 数据库操作（Neon PostgreSQL）
│       │       ├── github/     # GitHub API 封装
│       │       ├── model-*     # 模型选择与访问控制
│       │       └── sandbox/     # Sandbox 配置与认证同步
│       └── package.json
├── packages/
│   ├── agent/                  # 核心 Agent 包（可独立引用）
│   │   ├── open-harness-agent.ts   # Agent 主入口（基于 ai SDK）
│   │   ├── models.ts           # AI Gateway 模型网关
│   │   ├── context-management/ # 上下文压缩策略
│   │   │   └── aggressive-compaction-helpers.ts
│   │   ├── skills/             # Skill 系统
│   │   │   ├── discovery.ts    # Skill 发现机制
│   │   │   ├── loader.ts       # Skill 加载与参数替换
│   │   │   └── types.ts        # Frontmatter Zod schema
│   │   ├── subagents/          # 子 Agent 系统
│   │   │   ├── registry.ts     # 子 Agent 注册表
│   │   │   ├── explorer.ts     # 只读代码探索子 Agent
│   │   │   ├── executor.ts     # 执行型子 Agent（写代码）
│   │   │   └── design.ts       # 前端设计子 Agent
│   │   └── tools/              # 工具集
│   │       ├── bash.ts         # Shell 执行（含危险命令审批）
│   │       ├── read.ts         # 文件读取
│   │       ├── write.ts        # 文件写入
│   │       ├── edit.ts         # 文件编辑（AST 级）
│   │       ├── glob.ts         # 文件模式匹配
│   │       ├── grep.ts         # 内容搜索
│   │       ├── task.ts         # 任务拆解工具
│   │       ├── skill.ts        # Skill 调用工具
│   │       └── todo.ts         # Todo 列表工具
│   ├── sandbox/               # Sandbox 抽象接口包
│   │   ├── interface.ts        # Sandbox 接口定义
│   │   ├── factory.ts          # Sandbox 工厂
│   │   └── vercel/             # Vercel Sandbox 实现
│   │       ├── sandbox.ts      # VercelSandboxSDK 封装
│   │       ├── connect.ts      # Sandbox 连接与恢复
│   │       ├── snapshot-refresh.ts  # 快照刷新
│   │       ├── state.ts        # 状态序列化
│   │       └── utils.ts        # 工具函数
│   └── shared/                # 跨包共享工具
│       ├── diff.ts             # Git diff 工具
│       ├── tool-state.ts       # 工具状态管理
│       └── paste-blocks.ts     # 粘贴块处理
└── docs/
    ├── agents/                # Agent 相关文档
    └── plans/                 # 设计文档
```

---

## 四、核心模块深度解析

### 4.1 Agent 层：`@open-harness/agent`

**核心文件：** `packages/agent/open-harness-agent.ts`

```typescript
export const openHarnessAgent = new ToolLoopAgent({
  model: defaultModel,  // anthropic/claude-opus-4.6
  instructions: buildSystemPrompt({}),
  tools: {
    todo_write, read, write, edit,
    grep, glob, bash, task,
    ask_user_question, skill, web_fetch
  },
  stopWhen: stepCountIs(1),  // 每步后主动让出控制权
  callOptionsSchema,          // 强类型 call options
  prepareStep: addCacheControl,  // 上下文压缩
  prepareCall: /* 注入 sandbox 上下文 */,
});
```

**关键设计点：**

1. **基于 `ai` SDK**：使用 Vercel 的 `ai` SDK（`ToolLoopAgent`），而非自研 Agent 循环
2. **`stopWhen: stepCountIs(1)`**：每完成一次 tool call 循环就主动停止，让 Workflow 层决定下一步
3. **强类型 `callOptionsSchema`**：通过 Zod schema 定义 sandbox、model、skills 等参数
4. **`prepareCall` 注入**：每次调用前动态构建 system prompt，注入当前分支、工作目录、环境变量等

#### 模型选择：`AI Gateway`

**文件：** `packages/agent/models.ts`

```typescript
import { gateway } from "ai/gateway"

export const defaultModelLabel = "anthropic/claude-opus-4.6"
export const defaultModel = gateway(defaultModelLabel)
```

模型路由通过 Vercel AI Gateway，支持任意模型提供商（Anthropic、OpenAI、Google 等），通过 `providerOptionsOverrides` 可以覆写特定模型的参数。

#### 上下文压缩：`Aggressive Compaction`

**文件：** `packages/agent/context-management/`

随着对话步数增加，历史消息和 tool results 会不断膨胀。系统实现了"激进的上下文压缩"策略：

- 识别并移除冗余的 tool result 内容
- 对长文本进行摘要化处理
- 保持关键代码片段完整

### 4.2 工具层：Sandbox 抽象

**核心接口：** `packages/sandbox/interface.ts`

```typescript
export interface Sandbox {
  readonly type: SandboxType          // "cloud"
  readonly workingDirectory: string
  readonly currentBranch?: string
  readonly hooks?: SandboxHooks       // 生命周期钩子

  // 文件系统
  readFile(path: string, encoding: "utf-8"): Promise<string>
  writeFile(path: string, content: string): Promise<void>
  mkdir(path: string, options?: { recursive?: boolean }): Promise<void>
  exec(command: string, cwd: string, timeoutMs: number): Promise<ExecResult>
  execDetached(command: string): Promise<{ commandId: string }>

  // 生命周期
  snapshot?(): Promise<SnapshotResult>
  extendTimeout?(additionalMs: number): Promise<{ expiresAt: number }>
  stop(): Promise<void>
  getState?(): unknown  // 持久化/恢复状态
}
```

**Vercel Sandbox 实现：** `packages/sandbox/vercel/sandbox.ts`

- 使用 `@vercel/sandbox` SDK
- 支持最多 5 小时超时（`MAX_SDK_TIMEOUT_MS = 18,000,000ms`）
- 支持 `beforeStop`/`afterStart`/`onTimeout` 生命周期钩子
- 支持端口暴露：`3000`（Next.js）、`5173`（Vite）、`4321`（Astro）、`8000`（通用）
- GitHub Token 网络策略（自动注入 `x-access-token` header）

### 4.3 工作流层：Vercel Durable Workflow

**核心文件：** `apps/web/app/workflows/chat.ts`

这是整个系统的编排核心。它使用了 Vercel Workflow SDK 的 `"use workflow"` 指令，使得函数可以在多次 HTTP 请求之间持久化状态。

```typescript
export async function runAgentWorkflow(options: Options) {
  "use workflow";

  for (let step = 0; step < options.maxSteps; step++) {
    // 1. 运行一个 Agent step（stream + tool calls）
    const result = await runAgentStep(...);

    // 2. 检查是否需要继续（是否有待处理的 tool calls）
    const shouldContinue = result.finishReason === "tool-calls"
      && !shouldPauseForToolInteraction(result.responseMessage.parts);

    if (!shouldContinue) break;
  }

  // 3. 工作流自然结束后，执行 auto-commit
  if (finishedNaturally && options.autoCommitEnabled) {
    await runAutoCommitStep(...);
    if (options.autoCreatePrEnabled) {
      await runAutoCreatePrStep(...);
    }
  }
}
```

**关键设计：**

1. **幂等式 stream 管理**：使用 `compareAndSetChatActiveStreamId` 原子操作防止并发启动多个 workflow
2. **优雅中断**：通过 `stopMonitor` 监听 workflow 取消信号，触发 `AbortController`
3. **Step timing 追踪**：每个 step 的耗时、finish reason、模型 usage 全部记录
4. **自动 commit/PR**：工作流自然结束后自动执行 Git 操作

#### Chat Post-Finish：`chat-post-finish.ts`

工作流结束后还有一套复杂的收尾流程：

```typescript
await runAutoCommitStep({      // 自动 commit
  sandboxState, repoOwner, repoName, ...
})

if (hasAutoCommitChanges) {
  await runAutoCreatePrStep({  // 自动创建 PR
    sandboxState, repoOwner, repoName, ...
  })
}

await refreshDiffCache(...)    // 刷新 diff 缓存
await persistSandboxState(...) // 持久化 sandbox 状态
await recordWorkflowUsage(...) // 记录用量
```

### 4.4 子 Agent 系统

**注册表：** `packages/agent/subagents/registry.ts`

```typescript
export const SUBAGENT_REGISTRY = {
  explorer: {
    shortDescription: "Use for read-only codebase exploration...",
    agent: explorerSubagent,
  },
  executor: {
    shortDescription: "Use for well-scoped implementation work...",
    agent: executorSubagent,
  },
  design: {
    shortDescription: "Use for creating distinctive, production-grade frontend...",
    agent: designSubagent,
  },
}
```

**三种子 Agent 类型：**

| 类型 | 用途 | 典型场景 |
|------|------|----------|
| `explorer` | 只读代码探索 | 查找代码片段、理解项目结构 |
| `executor` | 执行型编码 | 编写功能、重构、编辑文件 |
| `design` | 前端设计 | 生成高质量 UI 代码 |

子 Agent 通过 `task` 工具由主 Agent 分派，形成**层次化 Agent 架构**。

### 4.5 Skill 系统

Skill 系统允许将结构化的指示（包含参考文档和工具限制）注入 Agent 上下文中。

**文件结构：**
```
.agents/skills/<skill-name>/
├── SKILL.md              # 核心描述 + frontmatter
└── references/          # 参考文档
    ├── auth.md
    └── proxy-support.md
```

**Frontmatter Schema（`types.ts`）：**

```typescript
export const skillFrontmatterSchema = z.object({
  name: z.string(),           // 唯一名称
  description: z.string(),    // 短描述
  version: z.string().optional(),
  "disable-model-invocation": z.boolean().optional(),  // 禁止模型自动调用
  "user-invocable": z.boolean().optional(),           // 是否可通过 slash 调用
  "allowed-tools": z.string().optional(),             // 允许的工具白名单
  context: z.enum(["fork"]).optional(),               // 执行上下文
  agent: z.string().optional(),                       // 指定 agent 类型
})
```

**Skill 加载（`loader.ts`）：**
- 从 YAML frontmatter 提取 metadata
- 支持参数插值（`substituteArguments`）
- Skill 可在运行时动态加载到沙箱环境

---

## 五、认证与授权体系

项目支持两种 OAuth 认证方式：

### 5.1 Vercel OAuth

用户通过 Vercel 账号登录，用于管理 Sandbox 和项目。

### 5.2 GitHub OAuth + GitHub App

```
GitHub OAuth  →  用户身份验证（谁在操作）
GitHub App    →  仓库访问授权（能访问哪些仓库）
```

- **GitHub OAuth**：基础用户身份
- **GitHub App**：细粒度仓库权限，支持安装到特定 org/repo
- **Token brokering**：Sandbox 环境中自动注入 GitHub token，通过网络策略限制仅 `api.github.com` 可携带 token

---

## 六、数据流分析

### 一次完整对话的生命周期

```
1. 用户发送消息
   POST /api/chat → chat/route.ts

2. 验证与初始化
   ├─ requireAuthenticatedUser()     ← 检查 Vercel OAuth
   ├─ requireOwnedSessionChat()       ← 验证 session + sandbox 所有权
   └─ createChatRuntime()             ← 连接/恢复 Sandbox

3. 启动 Durable Workflow
   start(runAgentWorkflow, [...])     ← Vercel Workflow SDK
   ├─ 原子竞争：compareAndSetChatActiveStreamId
   └─ 返回 ReadableStream

4. Workflow 执行循环（每个 step）
   ├─ 转换消息格式（Web → Model）
   ├─ 调用 openHarnessAgent.stream()
   ├─ 处理 tool calls（通过 Sandbox 执行）
   ├─ 检查中断/取消信号
   └─ 决定是否继续

5. 结束流程
   ├─ persistAssistantMessage()       ← 持久化最后响应
   ├─ refreshDiffCache()             ← 刷新 diff
   ├─ runAutoCommitStep()            ← 可选：自动 commit
   └─ runAutoCreatePrStep()          ← 可选：自动 PR

6. Stream 结束
   返回 SSE 流 → 客户端渲染
```

---

## 七、技术选型分析

| 层次 | 技术选型 | 评价 |
|------|----------|------|
| **Web 框架** | Next.js App Router | 成熟，API Routes + Server Components |
| **AI SDK** | Vercel `ai` SDK | 抽象了多模型调用、tool calling、流式响应 |
| **状态持久化** | Neon PostgreSQL | Serverless Postgres，Vercel 生态契合 |
| **Workflow** | Vercel Workflow SDK | 终于有原生的"跨请求持久化"方案 |
| **Sandbox** | `@vercel/sandbox` | Vercel 自研，Hibernate/Resume 支持 |
| **KV 缓存** | Upstash KV / Redis | 高性能，TTL 支持 |
| **模型路由** | Vercel AI Gateway | 多提供商统一入口 |
| **语言** | TypeScript (严格模式) | 全部包均用 TS，无 JS 遗留 |

---

## 八、设计亮点与工程哲学

### 亮点

1. **Agent/Sandbox 分离**：这是最有价值的设计决策，使得两者的生命周期完全解耦
2. **Workflow 驱动的 Agent 循环**：用 Durable Execution 代替手写的状态机，避免了回调地狱
3. **Tool 层的危险命令审批**：`bash.ts` 中实现了 `commandNeedsApproval`，防止 rm -rf 等危险操作
4. **Skill 系统的 frontmatter 设计**：用 Zod schema 约束 YAML，前端 DX 友好
5. **原子性 stream 管理**：防止重复启动 workflow 的 race condition
6. **全面的可观测性**：每个 step 的 timing、usage、finish reason、request/response body 全部记录

### 可改进之处

1. **Agent package 依赖 `ai` SDK 的 `ToolLoopAgent`**：虽然保持了简洁，但深度定制需要理解 SDK 内部机制
2. **Sandbox 状态通过 JSON 序列化传递**：对于复杂状态（如 git repo），序列化开销较大
3. **子 Agent 系统目前是静态注册**：无法在运行时动态注册新的子 Agent 类型

---

## 九、部署架构

```
Vercel (托管运行时)
├── Next.js App (Edge/Railway)
│   ├─ API Routes (认证、Chat、GitHub)
│   └─ WebSocket/Stream 处理
├── Durable Workflows (Agent 执行)
│   ├─ 每个 Chat Session 一个 Workflow Run
│   └─ 状态持久化在 Vercel 平台
└── Vercel Sandbox (按需创建)
    ├─ 文件系统（/vercel/sandbox）
    ├─ 开发服务器端口
    └─ Git 操作
```

**环境变量要求：**

- `POSTGRES_URL` + `JWE_SECRET`：**最低运行要求**
- `VERCEL_APP_CLIENT_ID/SECRET`：**Vercel 登录**
- `GITHUB_*`：**GitHub 集成**（repo 访问、PR 创建）

---

## 十、总结

Open Agents 是一个**架构清晰、工程质量高**的 AI Coding Agent 参考实现。其核心贡献在于：

1. 证明了 **Workflow + Tool Calling** 的架构范式可以很好地支撑多步 AI 编程任务
2. 展示了 **Agent/Sandbox 分离** 的实用价值
3. 提供了一个**可直接 fork 定制**的生产级起点

对于想构建类似系统的团队，这个项目的参考价值极高 —— 不是因为它用了多少黑科技，而是因为它在复杂场景下依然保持了**架构的简洁性和可理解性**。

---

*报告由 OpenClaw 每日代码架构分析任务自动生成*
*分析时间：2026-04-15 19:00 UTC*
