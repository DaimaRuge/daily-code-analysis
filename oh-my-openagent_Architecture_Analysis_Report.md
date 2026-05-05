# code-yeongyu/oh-my-openagent 深度架构分析

> 分析日期：2026-04-30 | 仓库版本：v3.17.6 | ⭐55k | 原名：oh-my-opencode
>
> 项目定位：OMO — the best agent harness（最佳 Agent 马具）

---

## 一、项目概述

### 1.1 一句话定位

**oh-my-openagent**（简称 **OmO**）是运行在 OpenCode 之上的插件型 Agent 编排系统，通过多模型并行调度、专家 Agent 军团和深度调试工具链，将终端 Agent 的开发能力从"单模型助手"提升为"全栈 AI 开发团队"。

### 1.2 商业与社区背景

| 维度 | 内容 |
|------|------|
| 作者 | YeonGyu Kim（@code-yeongyu） |
| 所属实体 | [Sisyphus Labs](https://sisyphuslabs.ai) |
| 许可协议 | **SUL-1.0**（Sisyphus 用户许可协议，非 OSI 标准开源协议） |
| 构建方式 | Building in Public，使用 AI 助手 Jobdori（基于 OpenClaw 定制）实时开发 |
| 影响 | 385+ Issues, 291+ PRs, 社区极其活跃 |
| 安装量 | npm 月下载量可观，分 12 个平台二进制包 |

### 1.3 核心理念

> "这不是给一个模型打类固醇——我们在运营一个联合体。Claude、GPT、Kimi、Gemini——各司其职，并行运转，永不停歇。"

项目哲学：
1. **多模型不可替代性** — 没有单一模型能垄断所有能力，组合使用才是最优解
2. **Agent 组合 > 单 Agent 增强** — 专家 Agent 并行协作远胜于给单个 Agent 塞更多工具
3. **Harness 问题必须先解** — 编辑工具的不稳定性是 Agent 失败的 root cause，必须从架构层面解决
4. **全兼容** — 不锁定生态，完全兼容 Claude Code 的 Hooks、Skills、MCP 和插件

---

## 二、核心架构

### 2.1 总体架构图（概念）

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenCode CLI / SDK                         │
│  (AgentConfig, Hook 生命周期, Tool 执行引擎, MCP 协议)        │
├─────────────────────────────────────────────────────────────┤
│                  oh-my-openagent Plugin Layer                   │
│                                                               │
│  ┌──────────────────────┐   ┌────────────────────────────┐  │
│  │   Plugin Entry       │   │   Config System            │  │
│  │   (index.ts)         │──▶│   (Zod schema + config.ts)  │  │
│  └──────┬───────────────┘   └────────────────────────────┘  │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Agent Builder                            │    │
│  │  (Plugin 初始化 → Agent 注册 → Builtin Agents 创建)    │    │
│  └──────────────────────────────────────────────────────┘    │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Agent Registry                            │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │    │
│  │  │ Sisyphus │ │Hephaestus│ │  Oracle  │ │Librarian│ │    │
│  │  │(Orchestr)│ │(Worker)  │ │(Analyst) │ │(Researc)│ │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └─────────┘ │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │    │
│  │  │  Explore │ │ Metis    │ │Multimodal│ │  Atlas  │ │    │
│  │  │          │ │(Debug)   │ │(Visual)  │ │(Planner)│ │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └─────────┘ │    │
│  └──────────────────────────────────────────────────────┘    │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │             Hook Pipeline                              │    │
│  │  Session Hooks → Tool Guard Hooks → Transform Hooks   │    │
│  │  (IntentGate, RalphLoop, Safety, ModelFallback, ...)  │    │
│  └──────────────────────────────────────────────────────┘    │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │          Feature Modules                               │    │
│  │  BackgroundManager / Skill Loader / Prometheus        │    │
│  │  / Comment Checker / Tmux Integration                 │    │
│  └──────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 技术栈

| 层 | 技术 |
|----|------|
| 运行环境 | **Bun**（构建、测试、运行时） |
| 宿主平台 | **OpenCode CLI SDK** (`@opencode-ai/sdk` + `@opencode-ai/plugin`) |
| 类型系统 | TypeScript 5.7+, Zod v4（配置校验 + 类型推断） |
| 构建工具 | Bun build → dist/，tsc emitDeclarationOnly |
| 模型解析 | 内置模型能力台账（Model Capabilities） |
| 代码分析 | @ast-grep/napi（AST 模式匹配）, LSP（IDE 级工具） |
| 数据格式 | js-yaml, jsonc-parser, picocolors |
| 遥测 | PostHog（匿名，可选关闭） |
| 二进制分发 | 12 个平台优化包（darwin/linux/windows × arm64/x64/baseline/musl） |

### 2.3 插件架构

OmO 是 **OpenCode 的一个插件**，这一点决定了它的架构约束：

```
package.json 中的 exports:
  "." → dist/index.js           # 插件主入口
  "./schema.json" → JSON Schema  # 配置的 JSON Schema 自动生成

依赖关系:
  oh-my-openagent (插件)
      ↓ 依赖
  @opencode-ai/plugin ^1.4.0    # 插件 API
  @opencode-ai/sdk ^1.4.0       # AgentConfig, Hook 类型

运行时加载:
  OpenCode 读取 opencode.json
      → 发现 "oh-my-opencode" 在 plugins 数组中
      → 加载 dist/index.js 的 default export
      → 插件在 OpenCode 生命周期中注册 Agent 和 Hook
```

`packages/` 目录仅包含 12 个平台的原生二进制包，核心代码全部在 `src/` 中。

---

## 三、模块分析

### 3.1 `src/index.ts` — 插件入口点

插件导出的核心工厂函数：

```typescript
export default function ohMyOpenCode(
  pluginContext: PluginContext         // OpenCode 提供的上下文
): OhMyOpenCodePluginConfig {          // 插件返回的配置
}
```

**关键职责**：
1. **配置加载** — 解析 `oh-my-opencode.json(c)` 并通过 Zod Schema 校验
2. **Agent 创建** — 调用 `createBuiltinAgents()` 创建所有自带 Agent
3. **Hook 注册** — 创建 `createCoreHooks()` 注册生命周期 Hook 管线
4. **Plugin State** — 管理 `ModelCacheState`、`BackgroundManager`、`ModelFallbackController`

设计亮点：**配置热加载** — 配置变化后自动重建 Agent 和 Hook，无需重启 OpenCode。

### 3.2 `src/agents/` — 多 Agent 系统

#### 3.2.1 Agent 类型系统 (`types.ts`)

OmO 定义了一套完整的 Agent 类型体系：

```typescript
type BuiltinAgentName =
  | "sisyphus"           // 🏛️ 主编排器（Orchestrator）
  | "hephaestus"         // 🔨 深度工作者（Deep Worker）
  | "oracle"             // 🔮 代码分析师（Code Analyst）
  | "librarian"          // 📚 知识研究员（Knowledge Researcher）
  | "explore"            // 🔍 探索者（Code Explorer）
  | "multimodal-looker"  // 👁️ 视觉分析（Visual Analysis）
  | "metis"              // 🧠 调试推理（Debug/Reasoning）
  | "momus"              // 🎭 评论审查（Comment Reviewer）
  | "atlas"              // 🌐 战略规划师（Prometheus/Planner）
  | "sisyphus-junior"    // 👶 轻量编排（Lightweight Orchestrator）

type AgentMode = "primary" | "subagent" | "all"
type AgentCategory = "exploration" | "specialist" | "advisor" | "utility"
type AgentCost = "FREE" | "CHEAP" | "EXPENSIVE"
```

每个 Agent 通过 `AgentPromptMetadata` 接口暴露其元数据：

```typescript
interface AgentPromptMetadata {
  category: AgentCategory       // 分组（用于 Sisyphus 动态 Prompt 分组）
  cost: AgentCost               // 成本分级（用于 Sisyphus 工具选择表）
  triggers: DelegationTrigger[] // 委派触发条件（"什么情况交给这个 Agent"）
  useWhen?: string[]            // 何时使用
  avoidWhen?: string[]          // 何时避免
  dedicatedSection?: string     // 独立 Prompt 章节（特殊 Agent 用）
  promptAlias?: string          // 提示词中使用的别名
  keyTrigger?: string           // Phase 0 的关键触发词
}
```

**架构洞察**：OmO 实现了一种 **元编排（Meta-Orchestration）**，每个 Agent 不仅是执行单元，还通过元数据告诉编排器"我适合干什么，什么时候别找我"。

#### 3.2.2 Agent 工厂模式

```typescript
type AgentFactory = ((model: string) => AgentConfig) & { mode: AgentMode }
```

每个 Agent 是一个**函数 + 静态属性**的组合，接受模型字符串参数，返回 OpenCode 的 `AgentConfig`。`mode` 作为静态属性暴露，用于在实例化前就确定 Agent 的角色类型。

#### 3.2.3 内置 Agent 配置生成 (`builtin-agents.ts`)

`createBuiltinAgents()` 是调度系统的核心工厂函数，接收 20+ 参数：

```typescript
export async function createBuiltinAgents(
  disabledAgents: string[],      // 禁用的 Agent 列表
  agentOverrides: AgentOverrides, // 用户覆盖配置
  directory?: string,             // 工作目录
  systemDefaultModel?: string,    // 系统默认模型
  categories?: CategoriesConfig,  // 自定义类别
  gitMasterConfig?: GitMasterConfig,
  discoveredSkills: LoadedSkill[], // 发现的技能
  customAgentSummaries?: unknown,
  browserProvider?: BrowserAutomationProvider,
  uiSelectedModel?: string,       // 用户在 UI 中选择的模型
  disabledSkills?: Set<string>,
  useTaskSystem = false,
  disableOmoEnv = false
): Promise<Record<string, AgentConfig>>
```

创建流程：
1. 从缓存读取已连接的模型提供商和可用模型列表
2. 合并用户类别配置与默认类别
3. 收集所有 Agent 的元数据和配置
4. **关键顺序**：先创建 Sisyphus（编排器），再创建 Hephaestus（深度工作者），然后添加其他子 Agent，最后创建 Atlas（规划师）
5. 每个 Agent 都会获得 **可用模型列表** 的知识，确保分发时选择最合适的模型

注意：代码明确注释说**禁止在插件初始化期间调用 OpenCode client API**，否则会因为死锁挂起。

### 3.3 Sisyphus — 多模型编排器

Sisyphus 是 OmO 的灵魂，位于 `src/agents/sisyphus/` 目录：

```
src/agents/sisyphus/
├── index.ts              # 导出所有变体
├── default.ts            # Claude 和通用模型的基座 Prompt
├── claude-opus-4-7.ts    # Claude Opus 4.7 特化 Prompt
├── gemini.ts             # Gemini 矫正覆盖层
├── gpt-5-4.ts            # GPT-5.4 特化 Prompt
├── gpt-5-5.ts            # GPT-5.5（Codex 风格）特化 Prompt
├── kimi-k2-6.ts          # Kimi K2.6 特化 Prompt
└── ...                   # 更多模型变体
```

#### 3.3.1 多模型特异化设计

Sisyphus 的核心创新：**同一个编排器逻辑，针对不同底层模型生成不同 Prompt**。

| 模型 | Prompt 策略 | 原因 |
|------|------------|------|
| Claude Opus 4.7 | 原生高质量响应，literal-instruction 调优 | Claude 本身就是最好的 Agent 基座 |
| GPT-5.4 | 块状结构化指令（Block-structured） | GPT 需要更严格的格式约束 |
| GPT-5.5 | Codex 风格章节划分 | GPT-5.5 在 Codex 场景下最优 |
| Gemini | **矫正覆盖层** — 纠正过度激进等倾向 | Gemini 有独特的"急于行动"倾向 |
| Kimi K2.6 | 高效任务分割 | Kimi 速度优势，需要精细化任务拆分 |

这是**校准层（Calibration Layer）**模式：Prompt 不是写成一份然后测试所有模型，而是针对每个模型的"性格"写不同的 Prompt。

#### 3.3.2 动态 Prompt 构建 (`dynamic-agent-prompt-builder.ts`)

```typescript
export function buildDynamicAgentPrompt(
  agents: AvailableAgent[],        // 可用 Agent 列表
  categories: AvailableCategory[], // 可用类别
  skills: AvailableSkill[],        // 可用技能
  useTaskSystem: boolean
): string
```

Sisyphus 的 Prompt 是**动态生成的**，包含：
- **委派表（Delegation Table）**：当遇到什么工作时，交给哪个 Agent
- **工具选择表（Tool Selection）**：根据任务类型推荐工具
- **技能列表（Skills）**：按需激活的技能描述
- **关键触发器（Key Triggers）**：Phase 0 阶段的条件反射式行动建议

这意味着每次 Agent 运行时，其 Prompt 都是**根据当前环境、可用 Agent、用户配置**实时组装的，而非硬编码。

### 3.4 `src/hooks/` — Hook 管线系统

OmO 的 Hook 系统是其扩展性的第二支柱：

```
src/hooks/
├── model-fallback/        # 模型降级容错
│   ├── index.ts           # 导出 accessor
│   ├── controller.ts      # 回退控制器
│   └── controller-accessor.ts  # 对外的访问器
├── print-safety-check/    # 输出安全检查
├── insert-with-hash/      # Hashline 插入
├── write-file-tool/       # 文件写入工具重写
├── merge-strategy/        # 合并策略
├── sanity-check/          # 合理性检查
├── auto-response/         # 自动回应
├── deploy-fast-forward/   # 快速部署
└── ...
```

#### 3.4.1 Hook 管道

```
Session Hooks（会话生命周期）:
  onSessionStart → onToolStart → onToolEnd → onSessionEnd

Tool Guard Hooks（工具安全守卫）:
  onToolStart → 权限检查 → 参数验证 → 安全过滤

Transform Hooks（内容转换）:
  onSessionStart → Prompt 增强 → 响应后处理 → 结果格式化
```

核心 Hook 组件：
1. **IntentGate（意图门）** — 在行动前分析用户真正想要什么，而非字面意思
2. **Ralph Loop** — 自引用闭环，直到任务 100% 完成
3. **Hashline 编辑系统** — 基于内容哈希的行级编辑验证
4. **Model Fallback** — 模型失败时的自动降级链

### 3.5 `src/features/` — 特性模块

```
src/features/
├── background-agent/      # 后台 Agent 并行执行
├── opencode-skill-loader/ # 技能加载器
├── prometheus/            # 战略规划师
├── todo-enforcement/      # Todo 强制执行
└── ...
```

#### 3.5.1 后台 Agent 系统 (`features/background-agent/`)

```typescript
export { BackgroundManager, type SubagentSessionCreatedEvent }
export { waitForTaskSessionID }
```

**并行调度机制**：
1. Sisyphus 发出 `delegate_task` 命令
2. `BackgroundManager` 创建新的子会话（subagent session）
3. 子 Agent 在后台独立运行，与主会话隔离状态
4. 主会话可以同时发射 5+ 个后台 Agent
5. 通过 `waitForTaskSessionID` 等待结果

这种架构的关键优势：**上下文隔离**。每个子 Agent 专注于自己的任务，不会污染主 Agent 的上下文窗口。

#### 3.5.2 技能加载器 (`features/opencode-skill-loader/`)

技能不仅仅是 Prompt 模板，而是**包含独立 MCP 服务器的完整包**：

```typescript
{
  name: string              // 技能名称
  systemPrompt: string      // 系统指令
  mcpServers: MCPConfig[]   // 按需加载的 MCP 服务器
  constraints: string[]     // 能力边界约束
}
```

---

## 四、Harness 机制

### 4.1 Harness 问题是什么

项目 README 引用了 Can Bölük 的《The Harness Problem》：

> "目前所有工具都无法为模型提供一种稳定、可验证的行定位标识……它们全都依赖于模型去强行复写一遍自己刚才看到的原文。当模型一旦写错——而且这很常见——用户就会怪罪于大模型太蠢了。"

简单说：所有终端 Agent 编辑文件时，都是让 LLM **重写它看到的原文片段**。如果 LLM 记错或写错缩进，修改就失败了。这不是模型笨的问题，是工具设计的问题。

### 4.2 Hashline — 内容哈希锚点编辑

OmO 的解决方案：

```
11#VK| function hello() {
22#XJ|   return "world";
33#MB| }
```

**工作原理**：
1. Agent 读取文件时，每行末尾附加强绑定内容哈希
2. Agent 修改时必须引用 `LINE#HASH` 标识
3. 如果文件在此期间被修改过 → 哈希不匹配 → 修改被拒绝
4. 如果模型试图修改不存在的行 → 立即报错

**效果数据**：
- 在 Grok Code Fast 1 基准上，修改成功率从 **6.7% 提升到 68.3%**

这是 OmO 架构中最具工程价值的创新，直接解决了终端 Agent 最大的可靠性问题。

### 4.3 Model Fallback — 模型降级链

```typescript
// 每个 Agent 可以配置降级链
fallback_models: string | (string | FallbackModelObject)[]
```

当主模型失败时（超时、速率限制、API 错误），自动尝试降级链中的下一个模型。这种设计使得 Agent 系统在面对 API 不稳定时具有**弹性**。

### 4.4 Ralph Loop — 自引用闭环

用户输入 `ultrawork` 后：
1. Sisyphus 制定计划 → 分发任务
2. 子 Agent 执行 → 返回结果
3. Sisyphus 评估完成度 → 如果不满足 100% → 重新计划
4. 循环直到完成 → 才返回用户

这是**验收驱动的循环（Acceptance-Driven Loop）**，确保任务不会像传统 Agent 那样做到 80% 就停手。

---

## 五、竞品对比

| 维度 | oh-my-openagent | Claude Code | OpenCode | Codex (终端 Agent 模式) |
|------|----------------|-------------|----------|------------------------|
| **本质** | OpenCode 的编排插件 | 独立 CLI Agent | 开源 Agent 框架 | OpenAI 的 Agent 平台 |
| **模型策略** | 多模型编排（最佳匹配） | 仅 Claude | 可配置单模型 | 仅 OpenAI 模型 |
| **编辑可靠性** | Hashline（哈希锚点） | 传统行号编辑 | 传统编辑 | 传统编辑 |
| **并行 Agent** | 5+ 后台 Agent 并行 | 无（串行） | 有限 | 无 |
| **编辑工具** | LSP + AST-Grep + Tmux | 基础文件编辑 | 基础 + 插件 | 基础编辑 |
| **技能系统** | 自带 MCP 的按需技能 | CLAUDE.md + 工具 | 插件系统 | 无 |
| **Intent 分析** | IntentGate（内建） | 无（需手动提示） | 无 | 无 |
| **自修复循环** | Ralph Loop | 无 | 无 | 无 |
| **MCP 支持** | 内建 Exa/Context7/Grep.app | 支持 MCP | 插件级 | 不支持 |
| **生态兼容** | 兼容全部 Claude Code 生态 | 封闭 | 开放插件 | 封闭 |
| **许可** | SUL-1.0（受限） | 专有 | Apache 2.0 | 专有 |
| **商业模式** | Sisyphus Labs 产品化 | Anthropic 产品 | 纯开源 | OpenAI 产品 |

### 5.1 核心差异分析

**vs Claude Code**：
- Claude Code 是**单模型单 Agent**，强大但孤军奋战
- OmO 是**多模型多 Agent 军团**，优缺点互补
- Claude Code 的编辑工具是稳定的，但 OmO 的 Hashline 从根本上解决了编辑问题

**vs OpenCode**：
- OpenCode 是 OmO 的运行基座（依赖关系）
- OmO 是 OpenCode 上最复杂的插件，大幅扩展了 OpenCode 的能力边界
- OpenCode 提供 AgentConfig 和 Hook 接口，OmO 将这些接口用到了极致

**vs Codex (终端 Agent)**：
- Codex 更倾向于 Agent 平台/API 而不是本地 CLI 工具
- OmO 更关注本地开发体验（文件编辑、LSP、AST）
- Codex 有更强的后台推理能力（o-series），但缺乏多模型编排

---

## 六、洞察总结

### 6.1 架构亮点

1. **元数据驱动的动态编排** — Agent 通过 `AgentPromptMetadata` 告诉编排器自己的特长和成本，Sisyphus 据此动态生成 Prompt。这是**声明式 Agent 协作**的典型模式。

2. **Prompt 校准层** — 不为"所有模型写一份 Prompt"，而是为每个模型"性格"写特化 Prompt。在 LLM-as-Compiler 时代，这个模式未来会普及。

3. **Hashline 从根本上解决 Harness 问题** — 不是让模型"写对人"，而是让工具"拒绝错的"。工程思维改变 Agent 可靠性。

4. **按需 MCP 服务器** — 技能自带 MCP，用完即毁，解决了全局 MCP 消耗 Context 的问题。

5. **后台 Agent 上下文隔离** — 并行任务的独立状态管理，避免了传统 Agent 的"上下文污染"问题。

### 6.2 局限性

1. **强绑定 OpenCode** — 依赖 OpenCode SDK 的 API 和生命周期，不能独立运行
2. **SUL-1.0 许可** — 不是纯开源软件，商业化用途可能受限
3. **学习曲线陡峭** — 20+参数、10 个 Agent、复杂配置，新手上手困难
4. **维护者瓶颈** — 核心维护者只有一人（YeonGyu Kim），虽然社区活跃但发布节奏受个人影响
5. **遥测争议** — 默认开启匿名遥测，虽然可关闭但隐私敏感用户可能有顾虑

### 6.3 真实价值评估

| 价值点 | 评分 (1-10) | 说明 |
|--------|------------|------|
| 开发效率提升 | ★★★★★★★★★☆ 9 | 多 Agent 并行、自动完成循环、100% 验收驱动的效果显著 |
| 编辑可靠性 | ★★★★★★★★★★ 10 | Hashline 是当前最优的编辑可靠性方案 |
| 模型利用率 | ★★★★★★★★★☆ 9 | 让每个模型做最擅长的事，性价比最高 |
| 生态兼容性 | ★★★★★★★★☆☆ 8 | 兼容 Claude Code 生态，但不能脱离 OpenCode |
| 易用性 | ★★★★★★☆☆☆☆ 6 | ultrawork 一键启动很好，但深度配置需要学习 |
| 社区活跃度 | ★★★★★★★★★☆ 9 | 55k stars, 350+ issues, 290+ PRs, Discord 活跃 |
| 工程质量 | ★★★★★★★★★☆ 9 | 代码结构清晰，类型系统完善，边缘情况处理良好 |

### 6.4 对 AI Agent 开发的启示

1. **多模型编排不是锦上添花，而是必然选择** — 模型差异化只会加剧，能充分利用不同模型优势的系统会胜出。

2. **工具质量比模型质量更重要** — Hashline 从 6.7% 到 68.3% 的提升证明，工具设计对 Agent 的实际表现影响远超模型能力差距。

3. **Context 管理是 Agent 系统的核心瓶颈** — OmO 通过按需 MCP、后台 Agent 上下文隔离、动态 Prompt 生成三管齐下，值得所有 Agent 系统参考。

4. **Agent 需要"人格匹配"** — 不同模型有不同的行为倾向（Gemini 过于激进、Claude 偏向保守），Prompt 校准层是应对模型多样性的最佳实践。

---

> **结论**：oh-my-openagent 是当前终端 Agent 生态中最激进的工程实践。它不只是给 Claude Code 打类固醇——它重新想象了一个"Agent 驱动开发"的工作流应该长什么样。⭐⭐⭐
