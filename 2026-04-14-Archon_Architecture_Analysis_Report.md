# Archon 技术架构与源码研读报告

**项目**: [coleam00/Archon](https://github.com/coleam00/Archon)  
**⭐ Stars**: 17,500+ | **语言**: TypeScript (Bun) | **License**: MIT  
**分析日期**: 2026-04-14  
**分析人**: OpenClaw 自动架构分析系统

---

## 一、项目概述

Archon 是"第一个开源的 AI 编码 Harness 构建器"，旨在让 AI 编码变得**确定性**和**可重复**。其核心理念类比：

- **Dockerfile** → 基础设施标准化
- **GitHub Actions** → CI/CD 标准化
- **Archon** → AI 编码工作流标准化

当用户对 AI Agent 说"修一下这个 bug"，模型每次执行可能不同：可能跳过规划、忘记跑测试、PR 描述不符合模板。Archon 的解决方案是**将开发流程编码为 YAML 工作流**，由 AI 填充每个步骤的智能，但整体结构是确定性的。

### 核心特性

| 特性 | 说明 |
|------|------|
| **工作流引擎** | YAML 定义的 DAG，支持顺序/循环/并行/人工审批门 |
| **Git Worktree 隔离** | 每个工作流运行在独立 worktree，5 个 fix 并行跑无冲突 |
| **Composable** | 确定性节点（bash 脚本、测试、git ops）+ AI 节点混合编排 |
| **多平台适配器** | Web UI / CLI / Telegram / Slack / Discord / GitHub |
| **17 个内置工作流** | 从 assist 到完整 PRD → PR 多 Agent 评审流程 |
| **对话持久化** | SQLite / PostgreSQL，跨会话保持上下文 |
| **零配置启动** | 无需 DATABASE_URL，默认 SQLite，开箱即用 |

---

## 二、系统架构总览

### 2.1 Monorepo 结构

```
Archon (Bun workspace, v0.3.6)
├── packages/
│   ├── cli/           # @archon/cli     — 命令行入口
│   ├── core/          # @archon/core    — 共享业务逻辑
│   ├── adapters/      # @archon/adapters — 平台适配器
│   ├── workflows/     # @archon/workflows — 工作流引擎
│   ├── isolation/     # @archon/isolation — Git Worktree 隔离
│   ├── git/           # @archon/git     — Git 操作封装
│   ├── paths/          # @archon/paths   — 路径/日志/版本管理
│   ├── server/        # @archon/server   — Web 服务器
│   └── web/           # @archon/web     — Web UI (SvelteKit)
├── .archon/           # 用户项目中的工作流定义
│   ├── workflows/defaults/  # 17 个内置 YAML 工作流
│   └── commands/defaults/   # 35 个 Markdown 命令文件
└── docs/, scripts/, tests/
```

**依赖关系**（关键）:
- `@archon/cli` 依赖 core / adapters / git / isolation / paths / workflows / server
- `@archon/server` 依赖 core / adapters / git / paths / workflows
- `@archon/adapters` 依赖 core / git / isolation / paths

### 2.2 核心数据流架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      Platform Adapters                            │
│  Web UI (SSE) │ CLI │ Telegram │ Slack │ Discord │ GitHub         │
└──────────────┬──────────────────────────────────────────────────┘
               │  handleMessage()
               ▼
┌─────────────────────────────────────────────────────────────────┐
│              @archon/core: Orchestrator Agent                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 1. 解析用户消息 → 检测 /invoke-workflow 或 /register-project│ │
│  │ 2. AI 路由：直接回答 or 触发工作流                           │ │
│  │ 3. 加载对应项目配置 (loadConfig)                            │ │
│  │ 4. 创建隔离环境 (worktree) 或复用                           │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────┬──────────────────────────────────────────────────┘
               │
       ┌───────┴────────┐
       ▼                ▼
┌─────────────────┐  ┌──────────────────────────────────────────┐
│ @archon/workflows│  │  @archon/core: AI Clients                │
│ DAG 执行引擎      │  │  ClaudeClient / CodexClient (工厂模式)  │
│ (executeWorkflow)│  └──────────────────────────────────────────┘
└────────┬────────┘
         │ node 执行
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ @archon/core: Command Handler                                    │
│ 解析 Markdown 命令文件 → 注入 prompt → 调用 AI 客户端            │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  @archon/core: Database (SQLite / PostgreSQL)                   │
│  7 张核心表: codebases / conversations / sessions /             │
│  workflow_runs / isolation_environments / messages /             │
│  workflow_events                                                │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 平台适配层架构

```
IPlatformAdapter (接口)
    ├── TelegramAdapter  (telegraf)
    ├── DiscordAdapter   (discord.js)
    ├── SlackAdapter     (@slack/bolt)
    ├── GitHubAdapter    (@octokit/rest)
    ├── GiteaAdapter    (社区)
    ├── GitLabAdapter   (社区)
    └── WebAdapter      (SSE + WebSocket)

IWebPlatformAdapter (扩展接口)
    └── WebAdapter (SSETransport + MessagePersistence)
```

---

## 三、核心子系统深度分析

### 3.1 工作流引擎 (@archon/workflows)

这是 Archon 的心脏，从 YAML 定义构建可执行的 DAG。

#### YAML 工作流 schema 关键字段

```yaml
nodes:
  - id: create-plan          # 唯一节点 ID
    command: archon-create-plan  # 引用的 Markdown 命令文件
    depends_on: []           # 依赖节点 ID（拓扑排序）
    context: fresh            # fresh | accumulate | none
    loop:                    # 循环配置（可选）
      until: ALL_TASKS_COMPLETE | APPROVED | 数字
      prompt: "..."
    interactive: true        # 人工审批门（暂停等待）
    trigger_rule: one_success  # 并行节点的完成规则
```

#### 关键设计：fresh context

```typescript
// packages/core/src/orchestrator/prompt-builder.ts
export function buildProjectScopedPrompt(
  context: 'fresh' | 'accumulate' | 'none',
  // fresh = 全新会话，无历史包袱
  // accumulate = 携带历史上下文
  // none = 仅注入系统级指令
)
```

#### archon-idea-to-pr 工作流深度解析（核心标杆）

这是最完整的端到端工作流，包含 **17 个节点，8 个阶段**：

```
Phase 0: CREATE PLAN
  └── create-plan (archon-create-plan, 22KB prompt 文件)

Phase 1: SETUP
  └── plan-setup (archon-plan-setup)

Phase 2: CONFIRM PLAN  
  └── confirm-plan (archon-confirm-plan)

Phase 3: IMPLEMENT
  └── implement-tasks (archon-implement-tasks, claude-opus-4-6)

Phase 4: VALIDATE
  └── validate (archon-validate)

Phase 5: FINALIZE PR
  └── finalize-pr (archon-finalize-pr)

Phase 6: CODE REVIEW (5 个并行 Agent)
  ├── review-scope → sync
  ├── code-review  → sync  (并行)
  ├── error-handling → sync
  ├── test-coverage → sync
  ├── comment-quality → sync
  └── docs-impact → sync
  → synthesize (等待所有并行节点，one_success 触发)

Phase 7: FIX REVIEW ISSUES
  └── implement-fixes (archon-implement-review-fixes)

Phase 8: FINAL SUMMARY
  └── workflow-summary (archon-workflow-summary, 10KB prompt)
```

**5 并行评审 Agent 设计亮点**：
- 每个评审 Agent 专注一个维度：代码质量、错误处理、测试覆盖、注释质量、文档影响
- `trigger_rule: one_success` — 任意一个完成即可触发综合
- 最终 synthesizer 把所有评审结果合并成结构化报告

### 3.2 Orchestrator Agent (@archon/core)

**文件**: `packages/core/src/orchestrator/orchestrator-agent.ts`

这是**单入口**，所有平台消息都经过它。核心职责：

1. **命令解析** — 扫描 AI 响应中的 `/invoke-workflow` 和 `/register-project` 指令
2. **智能路由** — 直接回答 vs 触发工作流
3. **上下文管理** — fresh / accumulate / none 三种上下文模式
4. **工作流调度** — 发现、选择、执行工作流

```typescript
// 解析 AI 响应中的特殊指令
export function parseOrchestratorCommands(
  response: string,
  codebases: readonly Codebase[],
  workflows: readonly WorkflowDefinition[]
): OrchestratorCommands {
  // 扫描 /invoke-workflow <workflow> @ <project>
  // 扫描 /register-project <path>
}
```

**会话状态机** (`packages/core/src/state/session-transitions.ts`)：
- `shouldCreateNewSession` — 何时创建新会话
- `shouldDeactivateSession` — 何时停用
- `detectPlanToExecuteTransition` — 规划→执行转换
- `getTriggerForCommand` — 命令触发的状态转换

### 3.3 数据库层 (@archon/core/db)

**自动适配**: `DATABASE_URL` 存在 → PostgreSQL，否则 → SQLite（默认 ~/.archon/archon.db）

```
7 张核心表:
  codebases           — 项目注册表（名称、路径、Git URL）
  conversations        — 对话记录
  sessions             — 会话状态（活跃/停用）
  workflow_runs        — 工作流执行记录
  isolation_environments — Git Worktree 隔离环境
  messages             — 消息历史
  workflow_events      — 工作流节点事件流
```

**PostgreSQL 支持**: 通过 Docker Compose 一键部署（`--profile with-db`），生产环境推荐。

### 3.4 AI 客户端工厂 (@archon/core/clients)

```typescript
// 工厂模式，按环境变量或配置自动选择
export { getAssistantClient } from './factory';
// 支持: Claude (官方 API / Code OAuth) + Codex
```

### 3.5 隔离环境 (@archon/isolation)

Git Worktree 是 Archon 并行能力的基础：

```bash
# 每个工作流运行在独立 worktree
git worktree add ../archon-worktree-issue-42 issue-42
# 5 个 fix 并行 → 5 个独立 worktree → 零冲突
```

清理策略：按天数清理（默认 7 天）或清理已合并分支的 worktree。

### 3.6 Web 服务器 (@archon/server)

基于 **Hono** + **Zod** + **OpenAPI**，提供 REST API：

```typescript
// 路由模块
src/routes/api.workflows.ts      // 工作流 CRUD
src/routes/api.conversations.ts  // 对话管理
src/routes/api.codebases.ts      // 项目管理
src/routes/api.messages.ts       // 消息收发
src/routes/api.workflow-runs.ts  // 工作流执行状态
src/routes/api.health.ts         // 健康检查
```

**Web 传输层**（SSE）：
- `SSETransport` — 服务器推送实时流
- `MessagePersistence` — 消息持久化
- `WorkflowEventBridge` — 工作流事件桥接

**环境安全设计**（亮点）：
```typescript
// @archon/paths/strip-cwd-env-boot
// Bun 会自动加载 CWD/.env，但用户项目可能有敏感信息
// 启动时先 strip 掉，再加载 ~/.archon/.env
```

### 3.7 CLI 入口 (@archon/cli)

```bash
archon workflow list              # 列出可用工作流
archon workflow run <name> [msg] # 运行工作流
archon chat <message>           # 发送消息给编排器
archon setup                     # 交互式安装向导
archon serve                     # 启动 Web UI
archon isolation list/cleanup    # worktree 管理
```

---

## 四、安全设计亮点

### 4.1 环境变量隔离

**问题**: Bun 会自动从 CWD 加载 `.env`，用户项目可能包含 `GITHUB_TOKEN` 等敏感信息。  
**方案**: `@archon/paths/strip-cwd-env-boot` 在任何模块初始化前，从 `process.env` 中删除 CWD 的 `.env` 键。Archon 的配置从 `~/.archon/.env` 读取，始终优先。

### 4.2 凭证清理

```typescript
// sanitizeCredentials() — 日志输出前清理敏感信息
// sanitizeError() — 错误消息中的凭证脱敏
// EnvLeakScanner — 扫描路径下是否存在敏感 .env 泄露
```

### 4.3 智能鉴权

```typescript
// CLAUDE_USE_GLOBAL_AUTH: 使用 `claude /login` 的内置 OAuth
// 无需显式 API Key，开箱即用
if (!process.env.CLAUDE_API_KEY && !process.env.CLAUDE_CODE_OAUTH_TOKEN) {
  process.env.CLAUDE_USE_GLOBAL_AUTH = 'true';
}
```

---

## 五、命令 / 工作流内容系统

### 5.1 命令文件（Markdown）

35 个 Markdown 命令文件，每个包含：
- `---` frontmatter: description / argument-hint / requires-config
- 角色定义（system prompt）
- 任务描述
- 工具能力说明

**示例** `archon-assist.md`（最简单，22 行）：
```markdown
---
description: General assistance - questions, debugging, one-off tasks
argument-hint: <any request>
---
# Assist Mode
**Request**: $ARGUMENTS
...
```

**最复杂的命令** `archon-create-plan.md`（22KB）：
- 详细的代码库分析流程
- 分阶段实现计划模板
- 验证和风险管理

### 5.2 工作流选择路由

```typescript
// @archon/workflows/router.ts
findWorkflow(workflows, userMessage): WorkflowDefinition
// 基于工作流名称或描述关键词匹配
// 支持模糊匹配和同义词
```

---

## 六、工程实践亮点

| 实践 | 实现 |
|------|------|
| **Monorepo** | Bun workspace，7 个内部包 + Web |
| **类型安全** | 全链路 TypeScript + Zod schema 验证 |
| **零配置** | SQLite 默认，无 DATABASE_URL 也能跑 |
| **环保变量安全** | CWD .env strip 机制 |
| **Fail Fast** | 未处理 Promise rejection 直接 exit(1) |
| **日志分层** | pino，lazy init 支持测试 mock |
| **版本自检** | CLI 启动时检查是否有更新 |
| **Lint/CI** | ESLint + Prettier + Type check + Husky |

---

## 七、技术选型分析

### 为什么选 Bun 而不是 Node.js？

- 更快的启动和包安装速度
- 内置 `.env` 自动加载（同时也是安全问题的根源，见 4.1）
- 原生 TypeScript 支持（无额外 transpile）
- 单二进制分发（`bun build` → 可执行文件）

### 为什么选 Hono 而不是 Express？

- 更轻量（比 Express 轻 10x）
- 与 Bun 原生兼容
- 内置 OpenAPI/Zod 集成（`@hono/zod-openapi`）
- 类型推断完整

### 为什么自研 DAG 而不是用现成工作流引擎？

- 需要与 AI 编码深度集成（fresh context、human gate）
- 需要 Git Worktree 隔离编排
- 需要与消息平台无缝衔接
- 现有引擎（n8n、Prefect）面向通用场景，不适合 AI 编码特定需求

---

## 八、与 OpenClaw 的对比启示

| 维度 | OpenClaw | Archon |
|------|----------|--------|
| 核心抽象 | Agent + Skill | Workflow (YAML DAG) |
| 执行模型 | Agent 循环 + 工具分发 | 工作流节点 + AI 命令 |
| 持久化 | SQLite (MEMORY.md) | SQLite/PostgreSQL |
| 隔离 | Session 隔离 | Git Worktree 隔离 |
| 多平台 | 飞书为主 | CLI/Web/IM 全覆盖 |
| 可视化 | Canvas | Web UI Dashboard |
| 人工审批 | 内置对话 | Workflow interactive gate |

**Archon 的最大启发**：将开发流程编码为确定性工作流，比让 Agent 自由发挥更可靠。OpenClaw 的 Skill 系统可以做类似的事情——定义 Skill 之间的依赖关系和执行顺序。

---

## 九、源码研读建议

### 推荐阅读顺序

1. **`packages/cli/src/cli.ts`** — 入口，理解命令注册
2. **`packages/workflows/src/schemas/workflow.ts`** — YAML schema，理解工作流定义
3. **`packages/workflows/src/executor.ts`** — 工作流执行引擎
4. **`packages/core/src/orchestrator/orchestrator-agent.ts`** — 消息路由核心
5. **`packages/server/src/index.ts`** — Web 服务器初始化
6. **`packages/core/src/db/connection.ts`** — 数据库适配层
7. **`.archon/workflows/defaults/archon-idea-to-pr.yaml`** — 最完整工作流实例

### 关键文件速查

| 功能 | 文件路径 |
|------|---------|
| 工作流 YAML schema | `packages/workflows/src/schemas/workflow.ts` |
| 工作流执行器 | `packages/workflows/src/executor.ts` |
| 工作流路由/选择 | `packages/workflows/src/router.ts` |
| Orchestrator 入口 | `packages/core/src/orchestrator/orchestrator-agent.ts` |
| 提示词构建 | `packages/core/src/orchestrator/prompt-builder.ts` |
| 会话状态机 | `packages/core/src/state/session-transitions.ts` |
| 命令解析 | `packages/core/src/handlers/command-handler.ts` |
| DB 连接/适配 | `packages/core/src/db/connection.ts` |
| Claude 客户端 | `packages/core/src/clients/claude.ts` |
| Worktree 隔离 | `packages/isolation/src/index.ts` |
| Git 操作 | `packages/git/src/index.ts` |
| 平台适配器基类 | `packages/core/src/types.ts` (IPlatformAdapter) |
| 环境安全 strip | `packages/paths/src/strip-cwd-env-boot.ts` |

---

## 十、总结

Archon 是一个**高度工程化**的 AI 编码工作流引擎。它的核心价值不是 AI 本身，而是**将 AI 能力封装进确定性流程**的能力。其架构设计有以下几个亮点值得学习：

1. **分层解耦清晰**：CLI / Server / Core / Adapters / Workflows / Isolation / Git 每层职责明确
2. **环境安全设计用心**：CWD .env strip + 全局 .env 优先 + 凭证清理
3. **零配置体验**：SQLite 默认，无须任何环境变量即可运行
4. **工作流可组合性**：YAML DAG + Markdown 命令 + 上下文模式，灵活度高
5. **多平台一致性**：统一的消息模型通过 IPlatformAdapter 适配不同协议

其工程复杂度主要在**工作流执行引擎**和**跨平台消息路由**两部分，是研究 AI Agent 工程化的优秀参考项目。

---

*报告由 OpenClaw 自动生成 | 数据来源: GitHub API + 源码 | 2026-04-14*
