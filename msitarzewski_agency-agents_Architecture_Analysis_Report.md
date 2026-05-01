# msitarzewski/agency-agents 深度架构分析

> **项目地址**: https://github.com/msitarzewski/agency-agents  
> **Stars**: ⭐ 88k+  
> **License**: MIT  
> **分析日期**: 2026-05-01  
> **分析师**: OpenClaw 代码架构分析引擎

---

## 目录

1. [项目概述](#1-项目概述)
2. [核心架构](#2-核心架构)
3. [模块分析](#3-模块分析)
4. [多 Agent 协同机制](#4-多-agent-协同机制)
5. [竞品对比](#5-竞品对比)
6. [设计亮点与技术特色](#6-设计亮点与技术特色)
7. [洞察总结](#7-洞察总结)

---

## 1. 项目概述

### 1.1 项目定位

**The Agency** 是一个精心构建的 AI Agent 人格集合库，由 Reddit 社区讨论孵化而来。它不是传统意义上的"代码框架"，而是一个**基于 Markdown 的 Agent 角色定义系统**，为 Claude Code、GitHub Copilot、Cursor、OpenClaw、Kimi Code 等主流 AI 编程工具提供**专业化、人格化的角色提示词（Prompt Engineering）**。

核心理念：
> "Assembling your dream team, except they're AI specialists who never sleep, never complain, and always deliver."
> （组建你的梦之队，只不过他们都是不睡觉、不抱怨、总能交付的 AI 专家。）

### 1.2 项目规模

| 维度 | 数据 |
|------|------|
| GitHub Stars | 88k+ |
| 目录分类 | 17 个专业领域 |
| Agent 数量 | 60+ 个专业角色 |
| 支持工具 | 10+ 个 AI 编程/对话工具 |
| 文件格式 | Markdown + YAML Frontmatter |
| 代码量 | 以 Markdown 文档为主，Bash 脚本为辅 |

### 1.3 目录结构

```
agency-agents/
├── .github/              # GitHub Actions CI/CD
├── academic/             # 学术领域 Agent（研究员、论文写作等）
├── design/               # 设计领域 Agent（UI/UX、品牌、视觉叙事）
├── engineering/          # 工程领域 Agent（前端、后端、DevOps、AI、安全等）
├── examples/             # 多 Agent 协作示例
├── finance/              # 金融领域 Agent
├── game-development/     # 游戏开发 Agent
├── integrations/         # 工具集成输出目录
├── marketing/            # 营销领域 Agent
├── paid-media/           # 付费媒体 Agent
├── product/              # 产品管理 Agent
├── project-management/   # 项目管理 Agent
├── sales/                # 销售领域 Agent
├── scripts/              # 安装与转换脚本
├── spatial-computing/    # 空间计算 Agent（VisionOS、WebXR）
├── specialized/          # 特殊领域 Agent
├── strategy/             # 战略领域 Agent
├── support/              # 客户支持 Agent
├── testing/              # 测试领域 Agent
├── CONTRIBUTING.md       # 贡献指南
├── README.md             # 项目说明
└── LICENSE               # MIT 许可证
```

---

## 2. 核心架构

### 2.1 架构哲学：声明式 Agent 定义

The Agency 采用**声明式（Declarative）架构**，每个 Agent 通过一个 Markdown 文件完整定义，包含：

```markdown
---
name: Agent Name
description: 一句话描述
color: 颜色标识
emoji: 表情符号
vibe: 人格钩子
services: 外部服务依赖（可选）
tools: 可用工具列表（可选）
---

# Agent Name

## 🧠 Your Identity & Memory
- **Role**: 角色定义
- **Personality**: 人格特质
- **Memory**: 记忆与学习能力
- **Experience**: 领域经验

## 🎯 Your Core Mission
- 核心职责 1
- 核心职责 2
- 核心职责 3

## 🚨 Critical Rules You Must Follow
- 领域特定规则

## 📋 Your Technical Deliverables
- 代码示例
- 模板
- 框架

## 🔄 Your Workflow Process
1. 阶段 1: 发现与研究
2. 阶段 2: 规划与策略
3. 阶段 3: 执行与交付

## 💭 Your Communication Style
- 沟通风格指南

## 🎯 Your Success Metrics
- 成功标准
```

### 2.2 元数据驱动（Metadata-Driven）

每个 Agent 文件顶部使用 YAML Frontmatter 定义元数据：

| 字段 | 说明 |
|------|------|
| `name` | Agent 名称 |
| `description` | 专业领域描述 |
| `color` | 视觉标识颜色 |
| `emoji` | 表情符号标识 |
| `vibe` | 人格钩子，一句话记忆点 |
| `services` | 外部服务依赖（付费/免费） |
| `tools` | 可用工具列表 |

这种设计使得 Agent 定义既**人类可读**（Markdown 正文），又**机器可解析**（YAML Frontmatter）。

### 2.3 转换与分发架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Source Layer                             │
│              (Markdown Agent Definitions)                   │
│  engineering/*.md  design/*.md  product/*.md  ...         │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Conversion Layer                           │
│                   scripts/convert.sh                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐  │
│  │  get_field  │ │  get_body   │ │     slugify         │  │
│  │ 提取 YAML   │ │ 提取正文    │ │ 生成工具标识        │  │
│  └─────────────┘ └─────────────┘ └─────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│               Integration Layer                             │
│              integrations/<tool>/                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │claude-code│ │  cursor  │ │ openclaw │ │  kimi    │ ...  │
│  │ .md      │ │ .mdc     │ │ SOUL.md  │ │ .md      │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Install Layer                              │
│                   scripts/install.sh                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ~/.claude/agents/  .cursor/rules/  ~/.openclaw/   │    │
│  └──────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 多 Agent 协同的"无运行时"设计

这是 The Agency 最独特的架构决策：**没有运行时引擎，没有消息总线，没有编排器**。

协同方式：
1. **人类作为编排者**：用户手动选择需要激活的 Agent，分别与每个 Agent 对话
2. **上下文传递**：通过复制粘贴将前一个 Agent 的输出传递给下一个 Agent
3. **并行激活**：在支持多会话的工具中，可以同时与多个 Agent 对话

这种设计的优劣：

| 优势 | 劣势 |
|------|------|
| 零依赖，零配置 | 无自动任务调度 |
| 跨工具兼容性强 | 无 Agent 间自动通信 |
| 人类完全可控 | 协作效率依赖人工 |
| 无需维护运行时 | 无法处理复杂依赖链 |

---

## 3. 模块分析

### 3.1 Agent 定义模块（核心）

每个 Agent 文件遵循严格的六段式结构：

```
Identity & Memory    → 人格与背景
Core Mission         → 核心使命
Critical Rules       → 必须遵守的规则
Technical Deliverables → 技术交付物
Workflow Process     → 工作流程
Communication Style  → 沟通风格
Success Metrics      → 成功指标
```

**示例分析**：Frontend Developer Agent

- **Identity**: "Detail-oriented, performance-focused, user-centric, technically precise"
- **Mission**: 构建现代 Web 应用、性能优化、代码质量保证
- **Rules**: 性能优先、无障碍设计（WCAG 2.1 AA）
- **Deliverables**: React 组件示例、性能优化方案、无障碍实现指南
- **Workflow**: 项目搭建 → 组件开发 → 性能优化 → 测试验证
- **Metrics**: Lighthouse > 90、3G 下 < 3s 加载、跨浏览器兼容

### 3.2 转换脚本模块（scripts/）

#### convert.sh — 格式转换引擎

```bash
# 核心功能
- 读取所有 Agent 目录的 .md 文件
- 提取 YAML Frontmatter（get_field）
- 提取 Markdown 正文（get_body）
- 转换为各工具特定格式

# 支持的工具
antigravity  → SKILL.md（Gemini Antigravity）
gemini-cli   → gemini-extension.json + skills/
opencode     → .opencode/agents/*.md
cursor       → .cursor/rules/*.mdc
aider        → CONVENTIONS.md
windsurf     → .windsurfrules
openclaw     → workspaces（SOUL.md + AGENTS.md + IDENTITY.md）
qwen         → SubAgent files
kimi         → Kimi Code CLI agents
```

#### install.sh — 安装分发引擎

```bash
# 核心功能
- 检测已安装的工具
- 交互式/自动安装
- 并行安装（--parallel --jobs N）
- 进度条显示

# 安装目标
~/.claude/agents/          # Claude Code
~/.github/agents/          # GitHub Copilot
~/.gemini/antigravity/skills/  # Gemini
~/.openclaw/agency-agents/ # OpenClaw
.cursor/rules/             # Cursor
.opencode/agents/          # OpenCode
```

### 3.3 集成适配模块（integrations/）

每个工具对应一个子目录，包含转换后的 Agent 文件：

**OpenClaw 集成示例**：
```
integrations/openclaw/
├── README.md
├── agency-frontend-developer/
│   ├── SOUL.md        # 人格定义
│   ├── AGENTS.md      # 操作指南
│   └── IDENTITY.md    # 身份标识
├── agency-backend-architect/
│   ├── SOUL.md
│   ├── AGENTS.md
│   └── IDENTITY.md
└── ...
```

### 3.4 示例模块（examples/）

包含多 Agent 协作的实际案例，如 **Nexus Spatial Discovery**：

- 8 个 Agent 并行工作
- 10 分钟墙钟时间完成完整产品发现
- 输出：市场验证、技术架构、品牌策略、增长计划、支持蓝图、UX 研究、项目计划、空间界面规格

---

## 4. 多 Agent 协同机制

### 4.1 协同模式分析

The Agency 实现了三种协同模式：

#### 模式一：顺序接力（Sequential Relay）
```
用户 → Agent A（市场调研）→ 用户复制结果 → Agent B（技术架构）→ 用户复制结果 → Agent C（实现开发）
```

#### 模式二：并行分工（Parallel Division）
```
          ┌→ Agent A（品牌策略）
用户发起任务 →┼→ Agent B（技术架构）→ 用户整合结果
          ├→ Agent C（UX 研究）
          └→ Agent D（项目计划）
```

#### 模式三：专家会诊（Expert Panel）
```
用户提出问题 → 同时咨询多个 Agent → 用户综合各 Agent 意见做出决策
```

### 4.2 "人类在回路"（Human-in-the-Loop）

The Agency 的协同机制完全依赖人类作为中间层：

| 功能 | 实现方式 |
|------|----------|
| 任务分配 | 人工选择 Agent |
| 上下文传递 | 复制粘贴 |
| 冲突解决 | 人工判断 |
| 质量控制 | 人工审核 |
| 进度跟踪 | 人工管理 |

### 4.3 与真正 Multi-Agent 框架的对比

| 特性 | The Agency | Mastra / LangGraph |
|------|-----------|---------------------|
| 运行时 | 无（依赖外部工具） | 有（Node.js/Python） |
| Agent 通信 | 人工传递 | 自动消息传递 |
| 状态管理 | 无 | 有（图状态机） |
| 任务调度 | 人工 | 自动 |
| 错误恢复 | 人工 | 自动重试 |
| 适用场景 | 个人/小团队 | 企业级应用 |

---

## 5. 竞品对比

### 5.1 对比框架总览

| 维度 | The Agency | Mastra | VoltAgent | LangGraph | OpenAI Agents SDK |
|------|-----------|--------|-----------|-----------|-------------------|
| **定位** | Agent 角色库 | TypeScript Agent 框架 | 企业级 Agent 平台 | 工作流编排框架 | 官方 Agent SDK |
| **Stars** | 88k+ | 12k+ | N/A | 10k+ | 8k+ |
| **运行时** | 无 | Node.js | Node.js | Python | Python |
| **多 Agent** | 人工协调 | 内置 | 内置 | 图编排 | 简单支持 |
| **部署方式** | Markdown 文件 | npm 包 | npm 包 | pip 包 | pip 包 |
| **学习曲线** | 极低 | 中等 | 中等 | 较高 | 低 |
| **适用场景** | 提示工程 | 应用开发 | 企业应用 | 复杂工作流 | 快速原型 |

### 5.2 深度对比分析

#### The Agency vs Mastra

| 方面 | The Agency | Mastra |
|------|-----------|--------|
| 核心能力 | 角色定义 + 提示词工程 | Agent 运行时 + 工具调用 |
| 代码量 | 几乎无代码 | 需要编写 TypeScript |
| 工具集成 | 依赖外部工具（Claude/Cursor） | 内置工具系统 |
| 记忆系统 | 无（单次会话） | 有（向量存储 + 线程） |
| 适用用户 | 非技术人员 / 产品经理 | 开发者 |

**互补性**：The Agency 可以作为 Mastra 的"角色设计层"，先用 The Agency 定义角色，再用 Mastra 实现运行时。

#### The Agency vs LangGraph

| 方面 | The Agency | LangGraph |
|------|-----------|-----------|
| 架构模式 | 无架构（纯文本） | 图状态机（Graph State Machine） |
| 循环支持 | 无 | 内置循环与条件分支 |
| 持久化 | 无 | 内置检查点（Checkpointing） |
| 流式输出 | 依赖外部工具 | 原生支持 |
| 人机交互 | 完全人工控制 | 可配置中断点 |

#### The Agency vs OpenAI Agents SDK

| 方面 | The Agency | OpenAI Agents SDK |
|------|-----------|-------------------|
| 模型绑定 | 工具无关 | 绑定 OpenAI 模型 |
| 功能范围 | 纯提示词 | 工具调用 + 文件搜索 + 代码解释 |
| 部署复杂度 | 极低 | 中等 |
| 企业特性 | 无 | 有（Guardrails、Tracing） |

### 5.3 差异化定位

The Agency 的**独特价值**不在于技术复杂度，而在于：

1. **领域深度**：每个 Agent 都是领域专家，不是通用模板
2. **人格化设计**：独特的 voice、communication style、success metrics
3. **工具无关**：一次定义，多工具分发
4. **零运行时成本**：无需服务器、无需维护
5. **社区驱动**：60+ 个 Agent 来自社区贡献

---

## 6. 设计亮点与技术特色

### 6.1 设计亮点

#### 亮点 1：YAML Frontmatter + Markdown 的混合格式

```yaml
---
name: Frontend Developer
description: Expert frontend developer...
color: cyan
emoji: 🖥️
vibe: Builds responsive, accessible web apps with pixel-perfect precision.
---
```

**优势**：
- 机器可解析（YAML）
- 人类可读（Markdown）
- Git 友好（文本 diff）
- 工具无关（通用格式）

#### 亮点 2：六段式 Agent 定义模板

强制性的六段结构确保每个 Agent 的完整性：

```
Identity → Mission → Rules → Deliverables → Workflow → Metrics
```

这对应了**角色扮演（Role Play）**的最佳实践：
- 我是谁？（Identity）
- 我要做什么？（Mission）
- 我的底线是什么？（Rules）
- 我产出什么？（Deliverables）
- 我怎么工作？（Workflow）
- 怎么衡量我？（Metrics）

#### 亮点 3：多工具转换系统

```bash
./scripts/convert.sh --tool all    # 生成所有工具格式
./scripts/install.sh --tool openclaw # 安装到 OpenClaw
```

支持 10+ 个工具，包括：
- Claude Code（Anthropic）
- GitHub Copilot
- Cursor
- OpenClaw
- Kimi Code
- Windsurf
- Aider
- Gemini CLI / Antigravity
- OpenCode
- Qwen Code

#### 亮点 4：领域覆盖广度

17 个专业领域，60+ 个角色：

| 领域 | 角色数量 | 典型角色 |
|------|----------|----------|
| Engineering | 25+ | Frontend、Backend、DevOps、AI Engineer、Security |
| Design | 8 | UI Designer、UX Researcher、Brand Guardian |
| Marketing | 7 | Growth Hacker、Content Creator、SEO Specialist |
| Sales | 8 | Outbound Strategist、Deal Strategist、Sales Engineer |
| Product | 3 | Product Manager、Product Designer |
| Project Management | 3 | Project Shepherd、Scrum Master |
| Finance | 3 | Financial Analyst、Accountant |
| Game Development | 3 | Game Designer、Unity Developer |
| Spatial Computing | 3 | XR Interface Architect、VisionOS Developer |
| Specialized | 5 | Sales Outreach、Legal Reviewer |

### 6.2 技术特色

#### 特色 1：声明式配置优于命令式代码

The Agency 证明了**提示词工程（Prompt Engineering）**可以作为一种"声明式编程"：

```markdown
## 🚨 Critical Rules You Must Follow
1. **Lead with the problem, not the solution.**
2. **Write the press release before the PRD.**
3. **No roadmap item without an owner, a success metric, and a time horizon.**
```

这些规则会被 LLM 直接执行，无需编写控制逻辑。

#### 特色 2：成功指标驱动（Metrics-Driven）

每个 Agent 都有明确的量化成功指标：

```markdown
## 🎯 Your Success Metrics
- Page load times are under 3 seconds on 3G networks
- Lighthouse scores consistently exceed 90
- Cross-browser compatibility works flawlessly
- Component reusability rate exceeds 80%
- Zero console errors in production environments
```

#### 特色 3：记忆与学习机制

```markdown
## 🔄 Learning & Memory
Remember and build expertise in:
- **Performance optimization patterns**
- **Component architectures**
- **Accessibility techniques**
- **Modern CSS techniques**
- **Testing strategies**
```

虽然实际运行时没有持久化记忆，但这种设计引导 LLM 在会话中保持一致的 expertise。

---

## 7. 洞察总结

### 7.1 架构本质

The Agency 不是一个"框架"，而是一个**协议（Protocol）**——一种定义 AI Agent 角色的标准化方法。它的核心价值在于：

1. **标准化角色定义**：YAML + Markdown 的混合格式成为事实标准
2. **跨工具可移植性**：一次定义，到处运行
3. **领域知识沉淀**：将专家经验编码为可复用的 Agent 定义

### 7.2 适用场景

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| 个人开发者提升效率 | ⭐⭐⭐⭐⭐ | 直接激活专家角色，获得高质量输出 |
| 小团队协作 | ⭐⭐⭐⭐ | 通过共享 Agent 定义统一工作标准 |
| 企业级应用开发 | ⭐⭐⭐ | 需要配合 Mastra/LangGraph 等运行时 |
| 复杂多 Agent 工作流 | ⭐⭐ | 缺乏自动编排，依赖人工协调 |
| 生产环境部署 | ⭐⭐ | 无运行时、无监控、无错误恢复 |

### 7.3 演进建议

1. **增加运行时层**：可选的轻量级编排器，支持 Agent 间自动消息传递
2. **状态持久化**：引入检查点机制，支持长周期任务
3. **记忆增强**：集成向量数据库，实现跨会话记忆
4. **工具调用标准化**：定义统一的工具调用接口，减少对特定工具的依赖
5. **社区市场**：建立 Agent 交换市场，促进角色定义的共享与交易

### 7.4 最终评价

**The Agency 是提示词工程（Prompt Engineering）领域的一个里程碑式项目。**

它证明了：
- **高质量的角色定义**可以显著提升 LLM 的输出质量
- **结构化的人格设计**（Identity + Mission + Rules + Metrics）是有效的
- **社区驱动的角色库**可以覆盖绝大多数专业领域
- **声明式定义**优于命令式代码，在特定场景下

但它也暴露了当前 AI Agent 生态的局限：
- **缺乏标准化的运行时**：每个工具都有自己的 Agent 机制
- **人工协调成本**：真正的多 Agent 自动协作仍需框架级支持
- **记忆与状态**：跨会话的持久化记忆仍是难题

**总结**：The Agency 是"Agent 角色设计"的最佳实践，是"Agent 运行时"的重要补充，但不是替代。对于需要快速获得高质量 AI 辅助的个人和小团队，它是首选；对于需要自动化、可扩展的企业级应用，它需要与 Mastra、LangGraph 等框架结合使用。

---

> **报告完成**  
> 生成时间: 2026-05-01  
> 报告路径: `/root/.openclaw/workspace/code-analysis-reports/msitarzewski_agency-agents_Architecture_Analysis_Report.md`
