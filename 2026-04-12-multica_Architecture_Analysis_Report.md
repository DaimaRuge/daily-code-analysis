# Multica 技术架构与源码研读报告

**项目名称：** multica-ai/multica  
**分析日期：** 2026-04-12  
**Stars：** 7,596 | **Forks：** 972  
**GitHub：** https://github.com/multica-ai/multica  
**技术栈：** Go + Next.js 16 (monorepo) | PostgreSQL 17 (pgvector) | Chi + gorilla/websocket  

---

## 一、项目概述

Multica 是一个开源的**托管式 AI Agent 平台**，定位为"AI 原生的任务管理工具"。其核心理念是将 AI 编码助手（Claude Code、Codex、OpenClaw、OpenCode）视为团队中的一员——可以接收任务指派、主动报告进度、提交评论、创建 Issue，实现接近人类协作方式的 AI 驱动开发流程。

### 核心特性

- **Agent as Teammate**：将 Issue 分配给 AI Agent，像对待同事一样
- **全生命周期管理**：enqueue → claim → start → complete/fail，实时进度通过 WebSocket 推送
- **技能沉淀**：每个解决方案可沉淀为可复用技能，供团队共享
- **统一运行时**：本地守护进程 + 云端运行时统一管理，支持 Claude Code / Codex / OpenClaw / OpenCode
- **多工作区隔离**：workspace 级隔离，各工作区有独立的 Agent、Issue 和设置

### 与同类项目对比

| 特性 | Multica | Linear | GitHub Copilot |
|------|---------|--------|---------------|
| Agent 作为团队成员 | ✅ | ❌ | ❌ |
| Issue 分配给 Agent | ✅ | ❌ | ❌ |
| 技能共享机制 | ✅ | ❌ | ❌ |
| 多运行时统一管理 | ✅ | ❌ | ❌ |
| WebSocket 实时同步 | ✅ | 部分 | ❌ |

---

## 二、系统架构

### 2.1 整体架构图

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│   Next.js    │────>│  Go Backend  │────>│   PostgreSQL     │
│   Frontend   │<────│  (Chi + WS)  │<────│   (pgvector)     │
└──────────────┘     └──────┬───────┘     └──────────────────┘
                            │
                     ┌──────┴───────┐
                     │ Agent Daemon │  (runs on your machine)
                     │Claude/Codex/ │
                     │OpenClaw/Code │
                     └──────────────┘
```

### 2.2 技术选型分析

| 层级 | 技术栈 | 选择理由 |
|------|--------|----------|
| 前端框架 | Next.js 16 (App Router) | 服务端渲染 + RSC，成熟的生态 |
| 后端框架 | Go + Chi | 高性能 HTTP 路由，轻量级 |
| 数据库 | PostgreSQL 17 + pgvector | 向量存储支持 Agent 记忆检索 |
| ORM | sqlc | 类型安全的 SQL-first 方案，从 SQL 生成 Go 类型 |
| 实时通信 | gorilla/websocket | Go 生态中最成熟的 WebSocket 库 |
| Monorepo | pnpm workspaces + Turborepo | 高效的多包管理 + 增量构建 |
| 桌面端 | Electron (electron-vite) | 跨平台桌面应用 |
| 样式方案 | Tailwind CSS + shadcn/ui (Base UI) | 原子化 CSS + 可复用组件 |

### 2.3 Monorepo 结构

```
multica/
├── apps/
│   ├── web/           # Next.js 主应用 (port 3000)
│   ├── desktop/       # Electron 桌面应用
│   └── docs/          # 文档站点
├── packages/
│   ├── core/          # 核心业务逻辑（headless，无 React DOM）
│   ├── ui/            # 原子化 UI 组件（零业务逻辑）
│   ├── views/         # 共享业务页面（无 next/* 依赖）
│   ├── eslint-config/ # ESLint 配置
│   └── tsconfig/      # 共享 TypeScript 配置
├── server/            # Go 后端 (port 8080)
│   ├── cmd/           # 入口点
│   ├── internal/      # 内部业务逻辑
│   │   ├── handler/  # HTTP 处理器
│   │   ├── service/  # 业务服务层
│   │   ├── middleware/
│   │   ├── realtime/ # WebSocket 处理
│   │   ├── daemon/   # Agent 守护进程通信
│   │   └── storage/  # 存储层
│   └── pkg/
│       ├── db/       # sqlc 生成的数据库代码
│       ├── agent/     # Agent 协议处理
│       └── protocol/  # 通信协议
├── e2e/               # Playwright E2E 测试
└── scripts/           # 安装和数据脚本
```

---

## 三、核心模块深度解析

### 3.1 Go 后端架构 (server/)

后端采用**分层架构**：

```
HTTP 请求
    ↓
Handler (internal/handler/)     ← 接收请求，参数校验
    ↓
Service (internal/service/)     ← 业务逻辑编排
    ↓
Storage (internal/storage/)     ← 数据持久化
    ↓
pkg/db (sqlc 生成)              ← 类型安全的 SQL 操作
```

#### Handler 层

每个 handler 对应一个资源域，文件列表：

| Handler | 职责 |
|---------|------|
| `auth.go` | 认证（登录、Token 管理） |
| `issue.go` | Issue CRUD + 评论 |
| `agent.go` | Agent 配置管理 |
| `daemon.go` | 守护进程注册与心跳 |
| `runtime.go` | 运行时（Runtime）管理 |
| `project.go` | 项目管理 |
| `comment.go` | 评论 |
| `inbox.go` | 通知收件箱 |
| `realtime.go` | WebSocket 事件推送 |
| `activity.go` | 活动日志 |

#### Realtime 模块

`internal/realtime/` 是 Multica 实时能力的关键模块：

- **WebSocket 连接管理**：维护与前端的双向长连接
- **事件广播**：当 Issue 状态变化、收到评论等事件发生时，主动推送至相关客户端
- **Agent 通信**：Daemon 通过 WebSocket 与后端通信，Agent 的执行进度实时同步到前端

#### Daemon 模块

`internal/daemon/` 处理本地 Agent 守护进程通信：

- **自动检测**：Daemon 启动时自动检测系统上安装的 Agent CLI（claude、codex、openclaw、opencode）
- **心跳机制**：定期向服务端发送心跳，保持运行时在线状态
- **任务分发**：从服务端接收指派的任务，转发给本地 Agent 执行

### 3.2 前端 Monorepo 架构

#### packages/core — Headless 核心业务逻辑

```
packages/core/src/
├── api/           # REST API 客户端
├── platform/      # 平台桥接（CoreProvider 初始化）
├── store/        # Zustand 全局状态
└── ...
```

核心原则：**零 react-dom 依赖**，可在任何 JS 环境中运行。包含：
- Zustand Store：认证、Workspace、Issue、Inbox 状态管理
- API Client：与后端 REST API 通信
- 业务 Hooks：数据获取和操作封装

#### packages/ui — 原子化 UI 组件

使用 **shadcn/ui** + **Base UI** 原始组件构建，所有组件**零业务逻辑**：
- 安装方式：`pnpm ui:add badge`（通过 `components.json` 配置）
- 样式：使用语义化设计 Token（`bg-background`、`text-muted-foreground`），禁止硬编码颜色

#### packages/views — 跨平台共享视图

```
packages/views/src/
├── issues/       # Issue 列表、详情、创建表单
├── auth/         # 登录页面
├── settings/     # 设置页面
└── layout/       # DashboardGuard 等布局组件
```

关键约束：
- **禁止导入** `next/*`、`react-router-dom`、`next/navigation`
- 所有路由使用 `NavigationAdapter` 抽象
- 可在 Web 和 Desktop 两个应用间共享

#### apps/web — Next.js 主应用

```
apps/web/src/
├── app/                    # Next.js App Router 路由（薄壳）
│   ├── (auth)/login/      # 认证路由
│   └── (dashboard)/       # Dashboard 布局
├── features/              # 按领域组织
│   ├── auth/              # 认证功能
│   ├── workspace/         # 工作区管理
│   ├── issues/            # Issue 管理
│   ├── realtime/          # WSProvider + 实时同步
│   └── skills/            # 技能管理
├── shared/                # 跨功能共享
│   ├── api/               # ApiClient + WSClient
│   └── types/             # 共享类型定义
└── platform/              # Next.js 平台特定代码（唯一的 next/* 导入点）
```

### 3.3 状态管理策略

Multica 制定了一套**严格的状态管理规则**，以避免常见的状态管理陷阱：

#### 三层状态模型

| 层级 | 工具 | 职责 |
|------|------|------|
| Server State | TanStack Query | 所有来自 API 的数据，WebSocket 事件通过 Query Invalidation 更新 |
| Client State | Zustand | UI 选择、过滤器、草稿、模态框状态、导航历史 |
| Cross-cutting | React Context | WorkspaceIdProvider、NavigationProvider 等跨层管道 |

#### 硬性规则

1. **Server 数据绝不复制到 Zustand** — 防止两个数据源不一致
2. **Workspace 级查询必须以 `wsId` 为 Key** — 工作区切换时自动触发正确的缓存失效
3. **Mutation 默认乐观更新** — 本地立即更新，失败回滚，服务端确认后 Invalidate Query
4. **WebSocket 事件只 Invalidate Query，不直接写入 Store** — 避免竞态条件

#### Zustand 常见陷阱

- Selectors 必须返回稳定引用：`s => ({ a: s.a, b: s.b })` 每次都创建新对象 → 触发无限重渲染
- Hooks 需要 workspace context 时，接受 `wsId` 作为参数，而非在内部调用 `useWorkspaceId()` — 保证在 `WorkspaceIdProvider` 外也能正常工作

### 3.4 包边界规则（Package Boundary Rules）

Multica 通过严格的包边界规则保证了前端架构的可维护性：

| 包 | 允许依赖 | 禁止依赖 |
|----|---------|---------|
| `packages/core/` | 纯 JS 库 | react-dom、react-router-dom、UI 库 |
| `packages/ui/` | Base UI 原始组件 | `@multica/core`（纯 UI） |
| `packages/views/` | core + ui | next/*、react-router-dom、stores |
| `apps/web/platform/` | views + core | — |
| `apps/desktop/platform/` | views + core | — |

**同逻辑复用规则**：如果同一逻辑同时出现在两个应用中，必须提取到共享包中。

---

## 四、数据库设计

### 4.1 sqlc 工作流

Multica 使用 **sqlc** 实现类型安全的数据库操作：

1. 在 `server/pkg/db/queries/` 编写 SQL
2. 运行 `make sqlc` 生成 Go 类型和函数
3. Handler 调用生成的类型安全函数

### 4.2 核心数据模型

基于 PostgreSQL 17 + pgvector，主要实体：

| 表/实体 | 描述 |
|---------|------|
| workspaces | 工作区（多租户隔离） |
| users | 用户 |
| agents | Agent 配置（绑定 Workspace） |
| runtimes | 运行时实例（本地 Daemon / 云端） |
| projects | 项目 |
| issues | Issue（状态机：open → in_progress → resolved） |
| comments | 评论 |
| skills | 可复用技能（Agent 经验沉淀） |
| inbox | 通知收件箱 |

### 4.3 pgvector 的应用

pgvector 用于**语义搜索**：
- Agent 生成的解决方案可存储为向量
- 相似问题自动推荐历史解决方案
- 支持 Agent 间的经验传递

---

## 五、实时通信架构

### 5.1 WebSocket 事件流

```
Frontend (WSClient)
       ↕ WebSocket
Go Backend (gorilla/websocket)
       ↕
PostgreSQL (事件变更触发)
       ↓
Handler 处理业务逻辑
       ↓
Realtime 模块广播事件
```

### 5.2 支持的事件类型

| 事件 | 来源 | 描述 |
|------|------|------|
| `issue.updated` | Handler | Issue 状态/内容变更 |
| `issue.created` | Handler | 新 Issue 创建 |
| `comment.created` | Handler | 新评论 |
| `agent.progress` | Daemon | Agent 执行进度 |
| `runtime.online` | Daemon | 运行时上线 |
| `runtime.offline` | Daemon | 运行时离线 |

### 5.3 Agent Daemon 通信协议

Daemon 通过 **HTTP long-polling** 或 **WebSocket** 与后端通信：

1. Daemon 启动 → 注册到后端（携带可用 CLI 列表）
2. 后端分配任务 → Daemon 接收并转发给本地 Agent
3. Agent 执行 → Daemon 实时上报日志/进度
4. 完成后 → Daemon 报告结果，后端更新 Issue 状态

---

## 六、CLI 与守护进程

### 6.1 multica CLI 命令

| 命令 | 描述 |
|------|------|
| `multica login` | 浏览器认证 |
| `multica daemon start` | 启动本地 Agent 运行时 |
| `multica daemon stop` | 停止守护进程 |
| `multica daemon status` | 查看守护进程状态 |
| `multica setup` | 一键配置（认证 + 启动守护进程） |
| `multica issue create` | 创建 Issue |
| `multica issue list` | 列出 Issue |
| `multica config local` | 配置本地自托管服务器 |

### 6.2 守护进程架构

守护进程是 Multica 的**边缘计算节点**：
- 运行在开发者本地机器上
- 自动检测系统中安装的 Agent CLI
- 接收任务并分配给合适的 Agent 执行
- 通过 WebSocket 实时上报执行状态

---

## 七、安装与部署

### 7.1 快速安装

```bash
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash
```

### 7.2 自托管部署

```bash
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash -s -- --local
```

需要 Docker，使用 `docker-compose.selfhost.yml` 启动完整栈。

### 7.3 开发环境

```bash
make dev  # 自动检测环境、创建 env、安装依赖、启动数据库和迁移
```

支持 **Git Worktree**：多个工作副本共享一个 PostgreSQL 容器，通过 `.env.worktree` 实现隔离。

---

## 八、测试策略

| 测试类型 | 工具 | 位置 |
|---------|------|------|
| 单元测试 | Vitest | `packages/core/*.test.ts` |
| 组件测试 | Vitest + Testing Library | `packages/views/*.test.tsx` |
| 集成测试 | Vitest + jsdom | `apps/web/*.test.tsx` |
| E2E 测试 | Playwright | `e2e/*.spec.ts` |
| Go 测试 | Go standard `go test` | `server/internal/handler/*_test.go` |

**测试原则**：测试跟着代码走，而非跟着应用走。共享业务逻辑的测试放在 `packages/core/`，共享 UI 组件的测试放在 `packages/views/`。

---

## 九、技术亮点与设计哲学

### 9.1 多运行时统一抽象

Multica 最核心的创新在于**运行时抽象层**：

- Claude Code、Codex、OpenClaw、OpenCode 各自有不同的 CLI 接口
- Multica 通过 Daemon 模块统一抽象它们的差异
- 前端看到的只有"Agent"这一统一概念
- 具体的 Agent 类型只是配置属性

### 9.2 渐进式复杂度

从 CLAUDE.md 可以看出，Multica 的架构演进遵循**渐进式复杂度原则**：

- 初期可能只是简单的 Web + DB
- 随着需求增长，自然地演进为 Monorepo + 多包共享
- 每个架构决策都有明确的**约束规则**支撑（如 Package Boundary Rules）
- 避免了"过度设计"和"架构腐化"两个极端

### 9.3 零摩擦开发者体验

- `make dev` 一键启动全部服务
- sqlc 从 SQL 生成类型安全代码，减少手写错误
- pnpm catalog 保证所有包依赖版本一致
- Worktree 支持允许多任务并行开发而不冲突

### 9.4 AI-Native 优先的设计

相比传统任务管理工具，Multica 的每个设计决策都服务于"AI Agent 作为团队成员"这一核心理念：

- Issue 可以分配给 Agent
- Agent 执行结果可自动转录为评论
- 解决方案可沉淀为技能
- 跨 Agent 的经验通过 pgvector 向量检索实现复用

---

## 十、与 OpenClaw 的关系

Multica 在架构上与 OpenClaw 有诸多相似之处：

| 方面 | OpenClaw | Multica |
|------|----------|---------|
| Agent 通信 | ACP 协议 | Daemon HTTP/WS 协议 |
| 前端框架 | 自构建 | Next.js 16 |
| 状态管理 | Agent Session 存储 | TanStack Query + Zustand |
| 技能系统 | Skills | Skills（同一概念） |
| 工具调用 | 内置工具集 | 依赖底层 Agent CLI |
| 部署模式 | 本地优先 | 云端 + 本地混合 |

Multica 可以视为 OpenClaw 的**上层管理层** —— 它不直接实现 Agent 能力，而是管理与调度多个底层 Agent（可以是 OpenClaw）。

---

## 十一、总结

**Multica 是一个架构设计极为成熟的 AI Agent 管理平台**，其技术亮点包括：

1. **Go + Next.js Monorepo**：兼顾后端性能与前端开发效率
2. **严格的前端分层架构**：core/ui/views 三层分离，零耦合
3. **类型安全的全栈**：sqlc 从 SQL 生成 Go 类型，前后端通过 ts types 共享类型
4. **统一的实时通信层**：WebSocket + PostgreSQL 触发器实现端到端实时
5. **AI-Native 任务管理**：Issue → Agent → Skill 的完整生命周期

这个项目的架构对于构建任何**多 Agent 协作平台**都有极高的参考价值。其设计原则——尤其是"渐进式复杂度"和"严格包边界"——是保持大型项目长期可维护性的关键。

---

*报告生成时间：2026-04-12 | 分析工具：静态源码分析 + API 数据挖掘*
