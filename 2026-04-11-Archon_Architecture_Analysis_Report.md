# Archon 技术架构与源码研读报告

> 分析日期：2026-04-11
> 项目：coleam00/Archon
> GitHub：https://github.com/coleam00/Archon
> Stars：15,325 | Forks：2,527 | 今日新增：756 stars
> 语言：TypeScript
> 许可证：MIT

---

## 一、项目概述

**Archon** 是全球首个开源的 AI 编程 Harness 构建器，其核心理念是：**让 AI 编程过程变得确定性和可重复**。

Archon 借鉴了三个历史先例：
- **Dockerfile** 对基础设施的标准化
- **GitHub Actions** 对 CI/CD 的标准化
- **Archon** 对 AI 编程工作流的标准化

本质上是将软件开发流程（规划→实现→验证→评审→PR）编码为 YAML 工作流，由 AI 填充每一步的智能，而流程结构本身是确定性的。

---

## 二、技术架构总览

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────┐
│           平台适配层（Platform Adapters）                    │
│   Web UI │ CLI │ Telegram │ Slack │ GitHub │ Discord       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Orchestrator                              │
│         消息路由 & 上下文管理 & 会话生命周期                  │
└────────────┬─────────────────────────────┬─────────────────┘
             │                             │
      ┌──────┴───────┐             ┌────────┴────────┐
      │ Command      │             │ AI Assistant   │
      │ Handler       │             │ Clients        │
      │ (斜杠命令)    │             │ (Claude/Codex)│
      └──────────────┘             └────────────────┘
             │                             │
      ┌──────┴─────────────────────────────┴──────┐
      │           Workflow Executor               │
      │           (YAML DAG 工作流执行引擎)       │
      └──────────────────┬──────────────────────┘
                          │
             ┌────────────┴────────────┐
             │   Isolation Providers    │
             │   (Git Worktree 等)      │
             └──────────────────────────┘
                          │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              SQLite / PostgreSQL (7 张表)                    │
│ Codebases │ Conversations │ Sessions │ Workflow Runs         │
│ Isolation Environments │ Messages │ Workflow Events          │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 包结构（Monorepo）

```
packages/
├── core/         # 核心业务逻辑（Orchestrator, DB, Clients, Handlers）
├── workflows/    # 工作流引擎（DAG 执行器, YAML 加载器, 路由）
├── isolation/    # 隔离环境（Git Worktree 提供者）
├── adapters/     # 平台适配器（Telegram, Slack, GitHub, Discord）
│   ├── chat/     # 聊天平台（Telegram, Discord, Slack, Web）
│   └── forge/    # 代码平台（GitHub）
├── git/          # Git 操作封装
├── paths/        # 路径管理工具
├── server/       # HTTP 服务器（Web UI + Webhook 端点）
├── web/          # 前端（Web UI）
├── cli/          # CLI 工具
└── docs-web/     # 文档站点
```

---

## 三、核心模块深度解析

### 3.1 工作流引擎（packages/workflows）

#### 3.1.1 DAG 节点模型

Archon 的工作流基于 **有向无环图（DAG）** 执行模型，每个节点可以是：

| 节点类型 | 描述 |
|---------|------|
| `prompt` | AI 提示节点，调用 LLM 生成内容 |
| `bash` | 确定性 bash 命令节点（不经过 AI）|
| `loop` | 循环节点，迭代直到 `until` 条件满足 |
| `interactive` | 人工审批门节点，暂停等待人类输入 |

YAML 示例：
```yaml
nodes:
  - id: plan
    prompt: "探索代码库并制定实现计划"
  - id: implement
    depends_on: [plan]
    loop:
      prompt: "阅读计划，实现下一个任务，运行验证"
      until: ALL_TASKS_COMPLETE
      fresh_context: true
  - id: run-tests
    depends_on: [implement]
    bash: "bun run validate"
```

#### 3.1.2 工作流执行器（executor.ts）

`executeWorkflow()` 是核心函数，流程如下：

1. **并发检测**：检查同一路径是否有运行中的工作流，防止冲突
2. **恢复检测**：查找之前的失败运行，尝试从断点恢复（`findResumableRun`）
3. **创建工作流运行记录**：写入 `workflow_runs` 表
4. **DAG 执行**：`executeDagWorkflow()` 遍历节点，按拓扑顺序执行
5. **结果持久化**：每次节点完成都记录 `workflow_events`
6. **错误处理**：三重错误分类（FATAL / TRANSIENT / UNKNOWN），连续 UNKNOWN 超过阈值则中止

#### 3.1.3 工作流加载器（loader.ts）

支持两种工作流来源：
- **Bundled（内置）**：`.archon/workflows/defaults/` 下的 17 个默认工作流
- **Project（项目级）**：项目本地 `.archon/workflows/` 下的自定义工作流

同名文件项目级覆盖内置级，支持团队协作时统一开发流程。

#### 3.1.4 工作流路由器（router.ts）

根据用户输入自动选择最合适的工作流：
- 解析用户意图（issue / PR / feature / review）
- 匹配工作流名称和描述
- 返回最佳匹配或让用户确认

### 3.2 编排器（packages/core/src/orchestrator）

#### 3.2.1 核心职责

`Orchestrator` 是系统的中枢，负责：
- 接收平台消息并路由
- 管理会话生命周期
- 触发工作流执行
- 流式响应回传平台

#### 3.2.2 会话状态机

Archon 使用**不可变会话**设计：
- 每次状态转换创建新会话，通过 `parent_session_id` 链接
- `transition_reason` 明确记录转换原因

转换触发类型：
- `first-message` — 首次消息
- `plan-to-execute` — 计划阶段完成，开始执行
- `isolation-changed` — 隔离环境变化
- `codebase-changed` — 代码库切换
- `reset-requested` — 重置请求

#### 3.2.3 后台工作流调度

Web 平台支持将工作流分发到后台 worker 会话执行：
1. 创建隐藏的 worker conversation
2. 每个 worker 有独立 worktree
3. 事件通过 SSE bridge 桥接回父会话
4. 执行结果汇总到父会话显示

### 3.3 隔离提供者（packages/isolation）

#### 3.3.1 接口设计

```typescript
interface IIsolationProvider {
  create(request: IsolationRequest): Promise<IsolatedEnvironment>;
  destroy(envId: string): Promise<DestroyResult>;
  get(envId: string): Promise<IsolatedEnvironment | null>;
  list(codebaseId: string): Promise<IsolatedEnvironment[]>;
  healthCheck(envId: string): Promise<boolean>;
}
```

#### 3.3.2 Git Worktree 实现

默认使用 Git Worktree 提供隔离：

**路径规范：**
- 主路径：`~/.archon/workspaces/<owner>/<repo>/worktrees/<branch>/`
- Docker 路径：`/.archon/workspaces/<owner>/<repo>/worktrees/<branch>/`

**分支命名规则：**

| 工作流类型 | 标识符 | 生成分支名 |
|-----------|--------|-----------|
| issue | "42" | issue-42 |
| PR（同仓库）| "123" | feature/auth（实际分支）|
| PR（Fork）| "123" | pr-123-review |
| task | "my-feature" | task-my-feature |

**复用机制：** WorktreeProvider 在创建前先检查是否存在可复用的现有 worktree（路径匹配或分支匹配），避免重复创建。

#### 3.3.3 隔离解析器（IsolationResolver）

智能决策是否需要创建新隔离环境：
- 已有有效 worktree → 复用
- 检测到 stale worktree → 清理后重建
- 仓库未注册 → 使用 cwd 无隔离
- 资源不足 → 自动清理后创建

### 3.4 AI 客户端（packages/core/src/clients）

#### 3.4.1 接口设计

```typescript
interface IAssistantClient {
  sendQuery(
    prompt: string,
    cwd: string,
    resumeSessionId?: string
  ): AsyncGenerator<MessageChunk>;
  getType(): string;
}

interface MessageChunk {
  type: 'assistant' | 'result' | 'system' | 'tool' | 'thinking';
  content?: string;
  sessionId?: string;
  toolName?: string;
  toolInput?: Record<string, unknown>;
}
```

#### 3.4.2 支持的 AI 后端

1. **Claude Client** — 使用 `@anthropic-ai/claude-agent-sdk`
   - 映射 SDK 事件到 `MessageChunk` 类型
   - 支持 `tool_use` 块解析
   - Session 持久化（通过 `resumeSessionId`）

2. **Codex Client** — 使用 `@openai/codex-sdk`
   - 事件类型映射：`(item.completed)` → `assistant/tool/thinking`
   - **关键**：检测 `turn.completed` 后立即退出循环，避免无限循环

#### 3.4.3 模型兼容校验

```typescript
isModelCompatible('claude', 'sonnet-4-20250514') // true
isModelCompatible('codex', 'claude-sonnet-4')   // false → 报错
```

工作流可覆盖配置的 provider 和 model，支持细粒度控制。

### 3.5 平台适配层（packages/adapters）

#### 3.5.1 接口设计

```typescript
interface IPlatformAdapter {
  sendMessage(
    conversationId: string,
    message: string,
    metadata?: MessageMetadata
  ): Promise<void>;
  ensureThread(
    originalConversationId: string,
    messageContext?: unknown
  ): Promise<string>;
  getStreamingMode(): 'stream' | 'batch';
  getPlatformType(): string;
  start(): Promise<void>;
  stop(): void;
  sendStructuredEvent?(conversationId: string, event: MessageChunk): Promise<void>;
}
```

#### 3.5.2 各平台适配模式

| 平台 | 通信模式 | 会话 ID 格式 |
|------|---------|-------------|
| Web UI | SSE 流式 + REST API | 用户提供字符串或 UUID |
| Telegram | Webhook/Polling | `chat_id` |
| GitHub | Webhook | `owner/repo#issue_number` |
| Slack | Webhook | `thread_ts` 或 `channel_id+thread_ts` |
| Discord | Webhook | 平台特定 |
| CLI | 同步 | `cli-{timestamp}-{random}` |

#### 3.5.3 Web UI SSE 机制

- 维护 `Map<conversationId, SSEWriter>` 流集合
- 断连时自动缓冲消息，重连后恢复
- 支持结构化事件（tool call 可视化、workflow 进度）

---

## 四、数据库设计

### 4.1 七张核心表

```
codebases                    # 代码库注册表
conversations               # 会话表
sessions                    # AI 会话表（可链接成链）
workflow_runs               # 工作流运行记录
isolation_environments      # 隔离环境
messages                    # 消息记录
workflow_events             # 工作流事件日志
```

### 4.2 关键关系

- `conversation` → `sessions`：一对多（会话链）
- `conversation` → `workflow_runs`：一对多
- `conversation` → `isolation_environments`：多对一（当前活跃隔离）
- `codebase` → `isolation_environments`：一对多
- `workflow_run` → `workflow_events`：一对多（执行轨迹）

### 4.3 支持双数据库

默认使用 **SQLite**，生产环境可切换 **PostgreSQL**（通过 `pg` 适配器）。

---

## 五、依赖与技术栈

### 5.1 核心依赖

| 包 | 版本 | 用途 |
|----|------|------|
| `@anthropic-ai/claude-agent-sdk` | ^0.2.89 | Claude Code 集成 |
| `@openai/codex-sdk` | ^0.116.0 | OpenAI Codex 集成 |
| `pg` | ^8.11.0 | PostgreSQL 驱动 |
| `zod` | ^3 | Schema 校验 |
| `@hono/zod-openapi` | ^0.19.6 | 工作流 Schema OpenAPI 文档 |

### 5.2 技术选型分析

- **Bun**: 运行时 + 包管理器，直接执行 TypeScript（无构建步骤）
- **TypeScript**: 全链路类型安全
- **Zod**: 运行时 Schema 校验（YAML 解析、配置校验）
- **Pino**: 结构化日志（JSON 格式，支持分级）
- **Async Generators**: AI 流式响应的核心抽象

---

## 六、关键设计模式

### 6.1 接口驱动架构

平台适配器和 AI 客户端均通过接口定义，核心编排器不依赖具体实现：
- 新增平台 → 实现 `IPlatformAdapter` 即可
- 新增 AI 后端 → 实现 `IAssistantClient` 即可

### 6.2 不可变会话链

每次状态转换生成新会话，老会话仅读不写，通过 `parent_session_id` 链成历史轨迹，天然支持回溯和审计。

### 6.3 DAG 执行 + 循环节点

工作流是 DAG，但支持 `loop` 节点在 DAG 内实现迭代：
- `until: ALL_TASKS_COMPLETE` — 所有子任务完成
- `until: APPROVED` — 人工审批通过
- `fresh_context: true` — 每次迭代使用新 AI session

### 6.4 错误分类与容错

三类错误（FATAL / TRANSIENT / UNKNOWN）差异化处理：
- FATAL：立即中止（认证/权限问题）
- TRANSIENT：压制并继续
- UNKNOWN：计数跟踪，超过阈值中止

### 6.5 工作流即代码

所有工作流定义在 `.archon/workflows/` 下，提交到 Git，实现：
- 版本控制
- 团队共享
- 代码审查流程

---

## 七、典型工作流解析：archon-idea-to-pr

```
用户输入：添加深色模式到设置页面
    │
    ▼
[规划] → 创建实现计划（4个子任务）
    │
    ▼
[实现循环] × N
  ├─ 迭代1：实现任务1/4
  ├─ 迭代1：测试失败 → 重试
  ├─ 迭代2：修复 + 任务2/4
  └─ 迭代2：测试通过 → 任务3/4 → 任务4/4 → 完成
    │
    ▼
[运行测试] bun run validate
    │
    ▼
[代码评审] 5个并行评审 agent
    │
    ▼
[人工审批门] 展示变更，等待确认
    │
    ▼
[创建PR] 推送代码，创建 Pull Request
```

---

## 八、源码研读要点

### 8.1 入口：packages/core/src/index.ts

所有对外 API 的统一导出点，外部通过 `@archon/core` 引入。

### 8.2 工作流执行入口

核心调用链：
```
orchestrator.ts: handleMessage()
  → command-handler.ts: 处理斜杠命令
    → workflows/executor.ts: executeWorkflow()
      → dag-executor.ts: executeDagWorkflow()
        → 各节点类型处理器（prompt/bash/loop/interactive）
```

### 8.3 配置文件加载

`loadRepoConfig()` 查找 `.archon/config.yaml` 或 `config.json`，支持覆盖全局配置。

### 8.4 环境变量管理

两层环境变量合并：
1. `~/.archon/.env`（全局）
2. 项目 `.archon/.env`（项目级）
3. 数据库 `codebase_env_vars`（加密存储）

---

## 九、设计亮点与工程价值

### 9.1 亮点

1. **Harness 理念创新**：不是又一个 AI Coding Agent，而是 AI Coding Agent 的"驾驶舱"，赋予人类对 AI 行为的结构化控制
2. **工作流可版本化**：`.archon/workflows/` 是 Git 管理的开发流程契约
3. **多 agent 并行**：PR review 工作流支持 5 个并行评审 agent
4. **智能隔离管理**：worktree 复用 + 自动清理 + 资源限制
5. **平台无关性**：接口抽象到位，从 CLI 到 Discord 无差异运行

### 9.2 工程价值

- **确定性**：相同的 issue + 相同的工作流 = 相同的输出质量
- **并发安全**：worktree 隔离支持 5 个 fix 并行无冲突
- **可观测性**：完整的 workflow_events 事件日志，可完整回放执行过程
- **容错健壮**：错误分类 + 恢复机制 + 幂等性设计

---

## 十、与 OpenClaw 的比较分析

| 维度 | Archon | OpenClaw |
|------|--------|----------|
| 核心定位 | AI 编程工作流引擎 | AI 个人助手 |
| 工作流模型 | YAML DAG + 循环节点 | 技能（Skill）系统 |
| 隔离模型 | Git Worktree | 进程/会话隔离 |
| 平台集成 | 多聊天平台 | 飞书等 |
| AI 后端 | Claude Code + Codex | 多模型 |
| 多 agent | 内置多 agent 评审 | 子 agent 编排 |

Archon 的 Harness 理念与 OpenClaw 的 Skill 系统在设计哲学上有共通之处——都将 AI 行为标准化为可复用、可版本化的单元。两者未来可在"工作流跨平台执行"方向进行互补探索。

---

## 附录：默认工作流清单

| 工作流 | 用途 |
|--------|------|
| archon-assist | 通用问答/调试/探索 |
| archon-fix-github-issue | Issue → PR 完整修复流程 |
| archon-idea-to-pr | 创意 → PR（含 5 并行评审）|
| archon-plan-to-pr | 计划 → PR |
| archon-issue-review-full | Issue 全流程评审 |
| archon-smart-pr-review | 智能 PR 复杂度分类评审 |
| archon-comprehensive-pr-review | 5 并行评审 + 自动修复 |
| archon-create-issue | 问题分类 + GitHub Issue 创建 |
| archon-validate-pr | 双分支 PR 验证 |
| archon-resolve-conflicts | Merge 冲突检测与解决 |
| archon-feature-development | 功能开发 → PR |
| archon-architect | 架构扫描与复杂度优化 |
| archon-refactor-safely | 安全重构（类型检查钩子）|
| archon-ralph-dag | PRD → 多 story 迭代 |
| archon-remotion-generate | Remotion 视频生成 |
| archon-test-loop-dag | 迭代测试工作流 |
| archon-piv-loop | 人工审查的 PIV 循环 |

---

*报告生成时间：2026-04-11 03:00 (Asia/Shanghai)*
*数据来源：GitHub trending + archon.diy 官方文档 + 源码分析*
