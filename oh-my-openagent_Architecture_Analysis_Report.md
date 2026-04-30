# code-yeongyu/oh-my-openagent 深度架构分析

> 项目名：omo / oh-my-opencode（曾用名 oh-my-opencode）
> GitHub：https://github.com/code-yeongyu/oh-my-openagent
> Stars：55k+，版本 3.17.6，许可证 SUL-1.0（Sybil Unrestricted License）
> 官方定位：**The Best Agent Harness** —— 多模型编排的 OpenCode 插件

---

## 一、项目概述

### 1.1 核心定位

`oh-my-openagent`（简称 OmO，原名 oh-my-opencode）是一个**多模型 Agent 编排 Harness（工具架）**，专为 [OpenCode](https://opencode.ai)（开源 AI 编程终端）设计。它的核心使命是：**把一个孤单的 AI Agent 变成一支有纪律、能协作、真正交付代码的开发团队**。

> "我们最初把这叫做'给 Claude Code 打类固醇'。那是低估了它。"
> — 作者自述

### 1.2 核心哲学：不绑定任何模型供应商

项目鲜明地反对"锁死在一个模型/供应商上"的做法。文档中直接引用了 **Anthropic 曾因该项目屏蔽 OpenCode** 的事件，并将其描述为"牢笼"：

- **Claude / Kimi / GLM** → 负责编排（Orchestration）
- **GPT** → 负责深度推理（Reasoning）
- **Minimax** → 负责速度（Speed）
- **Gemini** → 负责创意/前端（Creativity / Frontend）

作者自称烧了 $24,000 的 LLM API 费用测试了市面上所有代码 Agent，最后得出结论：OpenCode + OmO 是最优解。Sisyphus Labs 正在产品化一个叫 **Sisyphus** 的版本。

### 1.3 关键指标

| 指标 | 数值 |
|------|------|
| GitHub Stars | 55k+ |
| 最新版本 | 3.17.6 |
| 贡献者 | 活跃（见 GitHub Insights） |
| npm 月下载 | 量级在万级（oh-my-opencode 包） |
| 支持平台 | Linux (x64/arm64/musl), macOS (darwin x64/arm64), Windows |
| 内置 Agent 数量 | **11 个**（Sisyphus、Hephaestus、Prometheus、Atlas、Oracle、Librarian、Explore、Metis、Momus、Multimodal-Looker、Sisyphus-Junior） |
| 内置 Skills | `playwright`、`git-master`、`frontend-ui-ux` |
| 内置 MCP | Exa（网页搜索）、Context7（官方文档）、Grep.app（GitHub 代码搜索） |

---

## 二、核心架构

### 2.1 整体架构：三层 Agent 编排体系

OmO 构建了一套**三层架构**，将"规划"和"执行"严格分离：

```
用户输入
    ↓
┌─────────────────────────────────────────────────┐
│         规划层（Planning Layer）                │
│  Prometheus（战略规划师）                        │
│  Metis（缺口分析师）                              │
│  Momus（计划审查员）                              │
└─────────────────────────────────────────────────┘
    ↓ 产出 .sisyphus/plans/*.md
┌─────────────────────────────────────────────────┐
│        执行层（Execution Layer）                 │
│  Atlas（指挥家/任务调度器）                      │
└─────────────────────────────────────────────────┘
    ↓ 分配任务类别
┌─────────────────────────────────────────────────┐
│        Worker 层（专业 Agent 群）                │
│  Sisyphus-Junior（深度任务执行）                │
│  Oracle（架构顾问，只读）                        │
│  Librarian（文档/OSS 搜索）                     │
│  Explore（代码库快速 grep）                      │
│  Hephaestus（GPT 原生的自主深度工作者）          │
└─────────────────────────────────────────────────┘
```

### 2.2 关键创新：类别驱动模型路由（Category-Based Routing）

当 Sisyphus 把任务分配给子 Agent 时，**选择的是"类别"而非"模型名"**。类别会自动映射到最优模型：

| 类别 | 默认模型 | 适用场景 |
|------|---------|---------|
| `visual-engineering` | Gemini | 前端、UI/UX、设计 |
| `deep` | GPT-5.4 | 自主调研 + 端到端执行 |
| `quick` | GPT-5.4 Mini | 单文件修改、快速任务 |
| `ultrabrain` | GPT-5.4 xhigh | 复杂硬核逻辑、架构决策 |
| `artistry` | Gemini | 创意 + 设计工作 |
| `writing` | prose 优化模型 | 文案写作 |

用户完全不碰模型选择，**系统自动路由**。这解决了"每次都要手动切换模型"的问题。

### 2.3 自律 Agent 军团（Discipline Agents）

**Sisyphus**（主编排器）
- 底层模型：Claude Opus 4.7 / Kimi K2.5 / GLM-5
- 职责：计划 → 委派 → 推动任务完成，激进的并行执行
- 特性：绝不停歇（名字来自希腊神话，推石头上山永不停）

**Hephaestus**（深度执行者，"正牌工匠"）
- 底层模型：GPT-5.4
- 名称的讽刺含义：Anthropic 因为该项目屏蔽了 OpenCode，所以团队构建了一个"GPT 原生"的自主 Agent
- 职责：给目标不给步骤；自动探索代码库模式，从头到尾独立执行

**Prometheus**（战略规划师）
- 底层模型：Claude Opus 4.7 / Kimi K2.5 / GLM-5
- 职责：**访谈模式**——在动一行代码之前，先向用户提问、深挖需求、识别歧义，构建经过验证的详细计划

**Atlas**（任务列表指挥家）
- 底层模型：Claude Sonnet 4.6 / Kimi K2.5 / GPT-5.4 / Minimax M2.7
- 职责：执行 Prometheus 计划，系统地管理 Todo 项，协调工作

### 2.4 工具权限隔离

| Agent | 工具限制 |
|-------|---------|
| Oracle | 只读（blocked: write, edit, task, call_omo_agent） |
| Librarian | 无法写/编辑/委派 |
| Explore | 无法写/编辑/委派 |
| Multimodal-Looker | 仅允许 read |
| Atlas | 无法委派（blocked: task, call_omo_agent） |
| Momus | 无法写/编辑/委派 |

---

## 三、模块分析

### 3.1 packages/ 目录结构（Monorepo）

从 package.json 的 `files` 和源码路径来看，项目采用**扁平化单仓结构**（非严格 monorepo）：

```
packages/              ← 可能为 monorepo 子包（占位，未来扩展）
src/
├── cli/              ← CLI 工具（doctor、run 等）
├── hooks/             ← OpenCode Hook 系统
│   └── hashline-read-enhancer/   ← Hashline 读取增强
├── plugin/            ← 插件主体
│   ├── hooks/         ← session hooks
│   └── handlers/      ← 插件处理器
├── shared/            ← 共享模块
│   ├── model-capabilities/        ← 模型能力映射
│   ├── model-capability-aliases/  ← 模型别名
│   ├── model-capability-guardrails/
│   └── model-requirements.ts      ← 模型需求配置
├── tools/             ← 核心工具
│   └── hashline-edit/  ← Hashline 编辑工具（核心）
│       ├── hash-computation.ts     ← 哈希计算
│       ├── edit-operation-primitives.ts
│       ├── edit-operations.ts
│       ├── file-text-canonicalization.ts
│       ├── validation.ts
│       └── tools.ts    ← hashline edit tool 实现
└── index.ts           ← 插件入口
```

### 3.2 核心依赖

```json
{
  "@opencode-ai/plugin": "^1.4.0",     // OpenCode 插件系统
  "@opencode-ai/sdk": "^1.4.0",        // OpenCode SDK
  "@modelcontextprotocol/sdk": "^1.25.2", // MCP 协议
  "@ast-grep/napi": "^0.41.1",         // AST 重写工具
  "@code-yeongyu/comment-checker": "^0.7.0", // AI 注释检查
  "@clack/prompts": "^0.11.0",        // 交互式 CLI
  "vscode-jsonrpc": "^8.2.0",          // LSP 通信
  "posthog-node": "^5.29.2"            // 匿名遥测
}
```

### 3.3 核心工具链

#### 3.3.1 Hashline（基于哈希的行编辑工具）

这是 OmO 最具创新性的模块，也是解决"Agent 编辑失败率高的根本原因"的核心方案。

**问题根源（来自 oh-my-pi 的洞察）：**
> "目前所有工具都无法为模型提供一种稳定、可验证的行定位标识……它们全都依赖于模型去强行复写一遍自己刚才看到的原文。当模型写错——且这很常见——用户就怪罪于大模型太蠢。"

**Hashline 解决方案：**
Agent 读取文件时，每一行末尾会打上**内容哈希标签**：
```
11#VK| function hello() {
22#XJ|   return "world";
33#MB| }
```

编辑时通过 `LINE#ID` 引用目标行。如果文件在此期间被修改，哈希验证失败，直接**在代码被污染前驳回编辑**。

**效果数据：**
在 Grok Code Fast 1 benchmark 上，修改成功率从 **6.7% 飙升至 68.3%**——仅通过更换编辑工具实现。

#### 3.3.2 LSP 工具

提供 IDE 级精度的代码操作：
- `lsp_rename` — 工作区级重命名
- `lsp_goto_definition` — 跳转定义
- `lsp_find_references` — 查找引用
- `lsp_diagnostics` — 构建前诊断

#### 3.3.3 AST-Grep

支持 25 种编程语言的**语法树感知**的代码搜索和重写，不是简单的文本匹配，而是基于 AST 结构。

#### 3.3.4 Tmux 集成

完整的交互式终端支持——REPL、调试器、TUI 工具，Agent 可以在持久会话中运行。

#### 3.3.5 IntentGate

在分类或行动之前先分析用户的真实意图，解决"字面意思误导 AI"的问题。数据来源：Terminal Bench（Factory.ai）。

### 3.4 Skill-Embedded MCPs

传统的 MCP 问题：全局 MCP 服务器消耗大量 Context 预算。

OmO 的解法：**技能自带专属 MCP 服务器**。按需启动，任务完成即销毁，Context 窗口始终干净。

内置 Skills：
- `playwright` — 浏览器自动化
- `git-master` — 原子提交 + rebase 手术
- `frontend-ui-ux` — 设计优先的 UI 实现

### 3.5 `/init-deep` 深度上下文初始化

在项目目录层级中自动生成树状 `AGENTS.md`：
```
project/
├── AGENTS.md              ← 全局级架构
├── src/
│   ├── AGENTS.md          ← src 级规范
│   └── components/
│       └── AGENTS.md      ← 组件级说明
```
Agent 自动顺藤摸瓜加载对应上下文，无需手动管理。

### 3.6 Ralph Loop / `/ulw-loop`

自我引用闭环——Agent 不断自我审视，达不到 100% 完成度绝不停止。

---

## 四、Harness 机制深度解析

### 4.1 什么是 Harness？

在 Agent 系统的语境中，**Harness（工具架/马具）** 指的是让 AI Agent 能够可靠地执行任务的一整套机制、工具和编排模式。核心挑战是：模型的推理能力再强，如果编辑工具不可靠、工具调用不稳定、任务委派机制混乱，整个系统就无法交付。

### 4.2 OmO 的 Harness 设计哲学

#### 问题一：工具不稳定 → Hashline
现有 Agent 框架的编辑工具依赖"模型复现原文"，当上下文变化或模型 token 截断时，复现失败导致编辑错位。OmO 用内容哈希替代了行号引用，让编辑目标有了一个**不可伪造的验证标签**。

#### 问题二：单模型瓶颈 → Category-Based Multi-Model Routing
不绑定单一最强模型，而是根据任务类型动态选择最优模型。`visual-engineering` 用 Gemini，`ultrabrain` 用 GPT-5.4 xhigh，`quick` 用 GPT-5.4 Mini。用户在配置文件中声明要什么，框架选择谁来做。

#### 问题三：上下文膨胀 → Skill-Embedded MCPs + `/init-deep`
MCP 服务器按需启动、任务级销毁；AGENTS.md 层级化按需加载。两者共同解决 Context 预算问题。

#### 问题四：Agent 摸鱼 → Todo Enforcer + Ralph Loop
Agent 想停止？系统直接揪回来。Ralph Loop 循环执行直到 100% 完成。

#### 问题五：规划与执行混在一起 → Prometheus + Atlas 分离
Prometheus 只读（只能修改 `.sisyphus/` 目录），Atlas 只执行（不能委派）。两者职责分离，避免"规划着规划着就开始写代码"的认知漂移。

### 4.3 编排流程详解

**ultrawork 模式（懒人模式）：**
```
用户输入 "ultrawork"
    ↓
Sisyphus 全权接管
    ↓
并行启动多个后台 Agent（Prometheus 规划、Hephaestus 执行、Explore 搜索...）
    ↓
Todo Enforcer 强制推动直到完成
    ↓
Ralph Loop 自审，输出最终报告
```

**Prometheus 模式（精确模式）：**
```
用户按 Tab 进入 Prometheus 模式
    ↓
Prometheus 访谈用户 → 收集需求 → 识别歧义
    ↓
Metis（缺口分析）预检计划
    ↓
Momus（严格审查）验证（高准确度模式）
    ↓
产出 .sisyphus/plans/{timestamp}.md
    ↓
用户确认 /start-work
    ↓
Atlas 读取计划 → 分配任务类别 → 分发给 Sisyphus-Junior 和 specialists
    ↓
每个任务完成后验证（lsp_diagnostics）
    ↓
Learnings 累积到 .sisyphus/notepads/{plan-name}/
    ↓
Atlas 汇总报告
```

### 4.4 模型回退链（Fallback Chains）

每个 Agent 都有一个按优先级排序的 Provider Chain：
```
Sisyphus: claude-opus-4-7 → kimi-k2.5 → glm-5
Hephaestus: gpt-5.4 → ...
Atlas: claude-sonnet-4-6 → kimi-k2.5 → gpt-5.4 → minimax-m2.7
```

运行时按顺序尝试 Provider，直到找到可用的模型。**即使首选 Provider 下线，工作仍继续**。

---

## 五、竞品对比

| 维度 | **OmO (oh-my-openagent)** | **LangGraph** | **Mastra** | **VoltAgent** | **OpenAI Agents SDK** |
|------|---------------------------|---------------|------------|--------------|---------------------|
| **架构模式** | Agent Harness（插件） | DAG 图编程框架 | Node.js 应用框架 | 轻量级 Agent 框架 | Python Agent 框架 |
| **模型编排** | 多模型自动路由（Category-based） | 手动配置 | 手动配置 | 手动配置 | 手动配置 |
| **Agent 数量** | 11 个内置 + 可扩展 | 自定义 | 自定义 | 自定义 | 自定义 |
| **编辑工具** | **Hashline**（基于内容哈希） | 通用 | 通用 | 通用 | 通用 |
| **规划执行分离** | ✅ Prometheus + Atlas | ❌ | ❌ | ❌ | ❌ |
| **Todo Enforcer** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Skill-Embedded MCP** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **内置 MCP** | Exa / Context7 / Grep.app | ❌ | ❌ | ❌ | ❌ |
| **LSP 集成** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **AST-Grep** | ✅（25 种语言） | ❌ | ❌ | ❌ | ❌ |
| **Tmux 集成** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Claude Code 兼容** | ✅ 插件兼容 | ❌ | ❌ | ❌ | ❌ |
| **匿名遥测** | PostHog（可选关闭） | ❌ | ❌ | ❌ | ❌ |
| **Provider 锁定** | **无锁定**（Claude/GPT/Kimi/Gemini/Minimax/GLM） | 取决于配置 | 取决于配置 | 取决于配置 | OpenAI 优先 |
| **复杂度定位** | 开箱即用（Batteries-included） | 中（需编写图逻辑） | 中（Node.js） | 轻量 | 轻量 |

### 5.1 各竞品简析

**LangGraph（LangChain 生态）**
- 优势：成熟的 DAG 编程模型，适合复杂多步骤工作流
- 劣势：需要手动定义节点和边，不负责模型路由，不解决编辑工具可靠性的根本问题

**Mastra**
- 优势：Node.js 开发者友好，内置工具链丰富
- 劣势：主要面向 JS/TS 生态，多模型编排能力有限

**VoltAgent**
- 优势：轻量、简单，适合快速原型
- 劣势：功能相对单薄，没有 OmO 的深度工具集成

**OpenAI Agents SDK**
- 优势：与 OpenAI 生态深度整合
- 劣势：**严重绑定 OpenAI**；模型编排能力有限；缺少 OmO 的 Hashline、Todo Enforcer 等核心机制

### 5.2 OmO 的差异化核心

1. **Hashline 是护城河**：解决了"模型改错行"的根本问题，将编辑成功率从 6.7% 提升到 68.3%。这是其他框架没有认真解决的问题。
2. **Category-Based Routing 是亮点**：用户说"做前端"，系统自动选 Gemini；用户说"写复杂算法"，系统自动选 GPT-5.4 xhigh。完全不需要手动模型切换。
3. **规划执行分离是工程化思维**：用 Prometheus 做访谈规划，用 Atlas 执行，两者职责清晰，避免 Agent 在规划阶段就开始写代码的常见问题。
4. **自律 Agent 概念**：Sisyphus 永不停止、Hephaestus 自主深度工作、Prometheus 强制规划——这套设计让 Agent 从"被动响应"变成"主动推进"。

---

## 六、洞察总结

### 6.1 这个项目解决的核心问题是什么？

**Agent 开发中最大的浪费不是模型不够聪明，而是工具不可靠、上下文管理混乱、任务委派没有章法。**

OmO 通过四个核心技术解决了这个问题：
1. **Hashline**：把编辑工具从"依赖模型复现原文"改成"基于不可伪造的内容哈希验证"，从根上解决了 stale-line 错误问题
2. **Category Routing**：把"选模型"变成"选任务类型"，让用户不需要懂模型，也让模型在它擅长的地方工作
3. **规划执行分离**：Prometheus 访谈规划 + Atlas 指挥执行，避免了 Agent 规划时就开始写代码的认知漂移
4. **Todo Enforcer + Ralph Loop**：从机制上保证任务完成，而不是依赖 Agent 的"自觉性"

### 6.2 创新价值评估

| 创新点 | 价值 | 可持续性 |
|--------|------|---------|
| Hashline 编辑系统 | **极高**：解决了行业普遍忽视的根本性问题，6.7%→68.3% 的提升数据很有说服力 | 高：依赖代码内容哈希，不依赖特定模型 |
| Category-Based 模型路由 | **高**：用户友好，抽象层次合理 | 高：模型映射表可配置，易扩展 |
| 自律 Agent 概念（Sisyphus/Hephaestus/Prometheus） | **中高**：概念清晰，但本质上是一种 prompt 优化策略 | 中：依赖模型能力，模型变强时优势可能缩小 |
| IntentGate | **中**：解决字面误解问题 | 中：依赖特定 benchmark 数据 |
| Skill-Embedded MCPs | **中**：解决了 MCP 膨胀问题 | 高：工程上合理的按需加载模式 |

### 6.3 用户真实反馈（来源：GitHub README）

- "因为它，我取消了 Cursor 的订阅。" — Arthur Guiot
- "如果人类需要 3 个月，Claude Code 需要 7 天，Sisyphus 只需要 1 小时。" — 量化研究员
- "一天之内解决了 8000 个 eslint 警告。" — Jacob Ferrari
- "用 Ohmyopencode 一晚上把 45k 行 Tauri 应用转成 SaaS Web 应用。" — James Hargis

### 6.4 局限性

1. **强依赖 OpenCode 生态**：OmO 是 OpenCode 的插件，脱离 OpenCode 无法独立运行
2. **多模型调优的维护成本**：随着支持的模型越来越多，模型映射表和 prompt 调优的维护成本指数增长
3. **匿名遥测（默认开启）**：尽管可禁用，但默认开启遥测可能让部分隐私敏感用户介意
4. **文档复杂度较高**："Skip This README"的反讽背后，实际上是因为功能太多导致文档体系庞大
5. **Sisyphus Labs 产品化风险**：如果 Sisyphus 商业版做得足够好，开源版 OmO 的维护动力可能受影响

### 6.5 结论

`oh-my-openagent` 是一款**工程化程度极高**的 Agent Harness。它的核心价值不在于"模型有多强"，而在于**把模型能做的事情用可靠的工具和机制包装起来，让输出稳定、可预期**。

Hashline 是它最具原创性的贡献，解决了行业长期忽视但影响巨大的"编辑工具可靠性"问题。Category-Based Routing 和规划执行分离则是工程化思维的体现——不是让模型更聪明，而是让系统更有纪律。

与 LangGraph、Mastra 等框架相比，OmO 更像一个**完整的解决方案**而非编程框架。它不要求你写图逻辑，但要求你接受它的 Opinionated 设计（类别路由、Todo Enforcer、Prometheus 模式）。如果你接受它的设计哲学，`ultrawork` 一键启动的体验确实领先于大多数竞品。

> "别再浪费时间去到处对比选哪个框架好了。我会去市面上调研，把最强的特性全偷过来，然后在这更新。"
> — 作者自述

这既是项目最大的优势（持续快速迭代），也是最大的风险（决策权集中在维护者一个人身上）。

---

*报告生成时间：2026-04-30 | 数据来源：GitHub README.md、docs/guide/、package.json、DeepWiki*