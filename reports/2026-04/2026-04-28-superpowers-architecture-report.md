# Superpowers 技术架构与源码研读报告

> **项目**: obra/superpowers  
> **版本**: v5.0.7 (2026-03-31)  
> **Stars**: 167.6k+  
> **分析日期**: 2026-04-28  
> **分析师**: OpenClaw Daily Code Analysis

---

## 一、项目概述

Superpowers 是一个 **Agentic Skills Framework & Software Development Methodology**，由 Jesse Vincent 创建。它不是传统意义上的代码库，而是一个**面向AI编程代理的行为塑造系统**——通过可组合的"技能"（Skills）和初始指令，确保编码代理在软件开发全流程中遵循经过验证的最佳实践。

核心定位：**AI代理的软件工程方法论基础设施**

---

## 二、架构全景：四层抽象模型

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 4: 应用层 (IDE/CLI)                   │
│   Claude Code │ Codex CLI │ Cursor │ OpenCode │ Copilot CLI │
├─────────────────────────────────────────────────────────────┤
│                    Layer 3: 插件适配层                          │
│   .claude-plugin/ │ .codex-plugin/ │ .cursor-plugin/         │
│   .opencode/      │ gemini-extension.json                      │
├─────────────────────────────────────────────────────────────┤
│                    Layer 2: 技能编排引擎                          │
│   Skills Library (14+ skills)                                │
│   ├── 设计阶段: brainstorming, writing-plans                  │
│   ├── 执行阶段: subagent-driven-development, executing-plans   │
│   ├── 质量阶段: test-driven-development, systematic-debugging │
│   └── 协作阶段: requesting-code-review, using-git-worktrees  │
├─────────────────────────────────────────────────────────────┤
│                    Layer 1: 基础设施层                          │
│   Hooks系统 │ 视觉化Brainstorm Server │ 测试框架 │ 文档体系   │
└─────────────────────────────────────────────────────────────┘
```

### 架构哲学

> **"Skills are not prose — they are code that shapes agent behavior."**

这是整个项目最核心的架构洞察。Superpowers 将**自然语言文档视为可执行的行为代码**——SKILL.md 文件不是给人读的参考文档，而是给AI代理执行的"行为字节码"。

---

## 三、核心设计理念深度解读

### 3.1 行为即代码 (Behavior-as-Code)

传统框架提供API和工具函数。Superpowers 提供**认知模式和工作流**。

每个 SKILL.md 都遵循严格的结构：

```yaml
---
name: skill-name-with-hyphens
description: Use when [具体触发条件，而非功能描述]
---

# Skill Name
## Overview
1-2句话的核心原则

## When to Use
触发条件列表 + 反模式警示

## Core Pattern
Before/After 对比

## Red Flags - STOP and Start Over
代理最常见的合理化借口及其反驳
```

**关键设计决策**：`description` 字段**只描述触发条件，绝不描述流程**。测试发现，当 description 总结工作流程时，Claude 会走捷径只读 description 而不读完整技能内容，导致行为偏差。

### 3.2 反理性化设计 (Anti-Rationalization Engineering)

这是 Superpowers 最具创新性的架构特征。代理（尤其是LLM）在压力下会找到规则漏洞。

每个技能都包含：
- **合理化表格** (Rationalization Table): 列出代理可能找的借口和反驳
- **红旗列表** (Red Flags): 明确的停止信号
- **铁律声明** (Iron Law): 不可违反的绝对规则

例如 TDD 技能中的：

```markdown
| Excuse | Reality |
|--------|---------|
| "Too simple to test" | Simple code breaks. Test takes 30 seconds. |
| "I'll test after" | Tests passing immediately prove nothing. |
| "Tests after achieve same goals" | Tests-after = "what does this do?" Tests-first = "what should this do?" |
```

### 3.3 TDD 应用于文档 (TDD for Documentation)

`writing-skills` 技能揭示了元级架构：创建技能本身就是TDD。

| TDD 概念 | 技能创建映射 |
|----------|-------------|
| 测试用例 | 压力场景 + 子代理 |
| 生产代码 | SKILL.md 文档 |
| 测试失败(RED) | 代理在没有技能时违反规则 |
| 测试通过(GREEN) | 代理在有技能时遵守规则 |
| 重构 | 关闭漏洞同时保持合规 |

**铁律：没有失败的测试，就没有技能。**

---

## 四、插件系统架构：多平台适配层

Superpowers 支持 7+ 个不同的AI编码平台，每个都有独特的插件格式和运行时模型。

### 4.1 平台矩阵

| 平台 | 插件格式 | 技能发现 | 启动注入 | 工具映射 |
|------|---------|---------|---------|---------|
| Claude Code | `plugin.json` + 市场 | 自动 | `hooks/session-start` | 原生 |
| Codex CLI | `plugin.json` (Codex标准) | `skills` 字段 | `additionalContext` | `codex-tools.md` |
| Cursor | `plugin.json` + `hooks-cursor.json` | `skills`/`agents`/`commands` | `sessionStart` hook | 原生 |
| OpenCode | `superpowers.js` (ESM) | `config.skills.paths` | `messages.transform` | `todowrite`, `@mention` |
| Copilot CLI | 市场 + 上下文注入 | `additionalContext` | 环境变量检测 | `copilot-tools.md` |
| Gemini CLI | `gemini-extension.json` | 扩展安装 | `GEMINI.md` | 原生 |
| GitHub Copilot Chat | 市场 | 插件市场 | 自动 | 原生 |

### 4.2 适配器模式深度分析

#### OpenCode 适配器 (最复杂)

```javascript
// .opencode/plugins/superpowers.js
export const SuperpowersPlugin = async ({ client, directory }) => {
  return {
    // 1. 配置注入：自动注册 skills 目录
    config: async (config) => {
      config.skills = config.skills || {};
      config.skills.paths = config.skills.paths || [];
      if (!config.skills.paths.includes(superpowersSkillsDir)) {
        config.skills.paths.push(superpowersSkillsDir);
      }
    },
    
    // 2. 消息转换：将 bootstrap 注入第一个用户消息
    // （避免系统消息重复导致的token膨胀）
    'experimental.chat.messages.transform': async (_input, output) => {
      const bootstrap = getBootstrapContent();
      // ... 注入逻辑
    }
  };
};
```

**关键架构决策**：
- 使用 `messages.transform` 而非 `system.transform`，避免每轮重复系统消息导致的token膨胀
- 前置 matter 解析器内联实现，避免外部依赖（零依赖设计）

#### 跨平台启动注入

所有平台共享同一个 `hooks/session-start` 脚本，但通过环境变量检测平台：

```bash
# 伪代码逻辑
if [ -n "$COPILOT_CLI" ]; then
    # Copilot CLI: 输出 JSON 格式 additionalContext
    echo '{"additionalContext": "..."}'
elif [ -n "$CURSOR_PLUGIN_ROOT" ]; then
    # Cursor: camelCase 格式
    ...
else
    # Claude Code: 标准格式
    ...
fi
```

### 4.3 零依赖哲学

> "Superpowers is a zero-dependency plugin by design."

这是一个**有意的架构约束**：
- 没有 npm 依赖（`package.json` 只有元数据）
- 没有外部服务调用
- 所有工具自包含
- 所有脚本使用 POSIX shell + Node.js 内置API

**收益**：
- 在任何环境中都能工作
- 不会因为依赖更新而破坏
- 安装就是 clone，无需 `npm install`
- 审计面为零

---

## 五、Skills 系统架构

### 5.1 目录结构

```
skills/
├── brainstorming/                    # 设计阶段
│   ├── SKILL.md
│   ├── visual-companion.md           # 浏览器可视化伴侣
│   ├── spec-document-reviewer-prompt.md
│   └── scripts/                      # Brainstorm 服务器
│       ├── server.cjs               # WebSocket + HTTP 服务器
│       ├── frame-template.html      # 视觉展示框架
│       └── helper.js                # 客户端交互逻辑
│
├── subagent-driven-development/      # 核心执行引擎
│   ├── SKILL.md
│   ├── implementer-prompt.md        # 实现者子代理提示
│   ├── spec-reviewer-prompt.md      # 规格审查提示
│   └── code-quality-reviewer-prompt.md  # 代码质量审查提示
│
├── test-driven-development/          # 质量保障
│   ├── SKILL.md
│   └── testing-anti-patterns.md     # 测试反模式参考
│
├── systematic-debugging/             # 调试方法论
│   ├── SKILL.md
│   ├── root-cause-tracing.md        # 根因追溯技术
│   ├── defense-in-depth.md          # 纵深防御
│   └── condition-based-waiting.md # 条件轮询等待
│
├── writing-plans/                    # 计划编制
│   ├── SKILL.md
│   └── plan-document-reviewer-prompt.md
│
├── writing-skills/                   # 元技能：创建技能
│   ├── SKILL.md
│   ├── anthropic-best-practices.md  # Anthropic 官方最佳实践
│   ├── persuasion-principles.md     # 心理学原理（Cialdini等）
│   └── testing-skills-with-subagents.md
│
└── [其他技能...]
```

### 5.2 技能类型学

Superpowers 识别四种技能类型，每种需要不同的测试策略：

| 类型 | 示例 | 测试策略 | 成功标准 |
|------|------|---------|---------|
| **纪律执行型** | TDD, verification-before-completion | 学术问题 + 压力场景 | 代理在最大压力下遵守规则 |
| **技术指导型** | condition-based-waiting, root-cause-tracing | 应用场景 + 变体场景 | 代理成功应用技术到新场景 |
| **思维模式型** | reducing-complexity, information-hiding | 识别场景 + 反例 | 代理正确识别何时/如何应用 |
| **参考文档型** | API docs, command references | 检索场景 + 应用场景 | 代理找到并正确应用信息 |

### 5.3 CSO (Claude Search Optimization)

这是一个**发现优化系统**，确保未来代理能找到正确技能：

1. **丰富的 Description 字段** — 回答"我现在应该读这个技能吗？"
2. **关键词覆盖** — 包含错误消息、症状、同义词、工具名
3. **描述性命名** — 动词优先：`creating-skills` 而非 `skill-creation`
4. **Token 效率** — 常用技能 <150 词，其他 <500 词

---

## 六、工作流编排引擎

### 6.1 核心工作流（8步闭环）

```
brainstorming → using-git-worktrees → writing-plans → 
subagent-driven-development/executing-plans → test-driven-development → 
requesting-code-review → finishing-a-development-branch
```

### 6.2 关键架构：HARD-GATE 模式

在 `brainstorming` 技能中：

```markdown
<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project, 
or take any implementation action until you have presented a design and 
the user has approved it. This applies to EVERY project regardless of perceived simplicity.
</HARD-GATE>
```

这不是建议，这是**架构级别的强制门**——使用XML标签和全大写来突破LLM的"建议-遵循"梯度，达到"规则-必须"的心理效果。

### 6.3 Subagent-Driven Development (SDD) 引擎

这是 Superpowers 最核心的执行架构：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Controller    │────▶│  Implementer    │────▶│  Spec Reviewer  │
│   (当前会话)     │     │  (子代理)        │     │  (子代理)        │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                       │                       │
         │◀──────────────────────│◀──────────────────────│
         │    传递完整任务文本      │    审查规格合规性       │
         │                       │                       │
         │              ┌─────────────────┐     ┌─────────────────┐
         │              │  Code Quality   │     │   TodoWrite     │
         │              │   Reviewer      │     │   状态跟踪       │
         │              │   (子代理)       │     │                 │
         │              └─────────────────┘     └─────────────────┘
```

**关键设计决策**：

1. **每任务新鲜子代理** — 隔离上下文，避免污染
2. **控制器提取所有任务** — 子代理不读文件，减少文件IO开销
3. **两阶段审查** — 先规格合规（防止过度/不足构建），再代码质量
4. **模型分层** — 机械任务用便宜模型，架构判断用最强模型

### 6.4 审查循环的成本优化演进

从 v5.0.4 到 v5.0.6 的关键架构演进：

| 版本 | 审查方式 | 平均耗时 | 质量分数 |
|------|---------|---------|---------|
| v5.0.3 | 子代理审查循环 (5轮max) | ~25分钟 | 基准 |
| v5.0.4 | 单轮全计划审查 (3轮max) | ~15分钟 | 相同 |
| v5.0.6 | **行内自审查** (检查清单) | ~30秒 | 相同 |

**架构洞察**：通过回归测试（5版本 x 5次试验）证明，子代理审查循环没有可测量的质量提升，但成本极高。替换为**结构化自审查检查清单**后，每次运行捕获 3-5 个真实bug，速度提升 50倍。

---

## 七、视觉化协作系统：Brainstorm Server

### 7.1 架构

```
skills/brainstorming/scripts/
├── server.cjs          # Node.js HTTP + WebSocket 服务器
├── frame-template.html # 视觉展示HTML框架
├── helper.js           # 客户端交互脚本
├── start-server.sh     # 启动脚本（PID生命周期管理）
└── stop-server.sh      # 停止脚本（SIGTERM + SIGKILL回退）
```

### 7.2 零依赖服务器设计

```javascript
// server.cjs 核心架构
const http = require('http');   // Node.js 内置
const fs = require('fs');       // Node.js 内置
const path = require('path');   // Node.js 内置

// 双目录结构（安全隔离）
// content/  — 通过HTTP服务（HTML可视化内容）
// state/    — 不通过HTTP服务（事件、日志、PID）
```

**安全设计**：将服务内容和服务器状态分离，防止状态数据通过HTTP泄露。

### 7.3 生命周期管理

```bash
# start-server.sh 架构
1. 检测平台 (WSL, Windows, *nix)
2. 生成 session 目录 (content/ + state/)
3. 启动 Node.js 服务器
4. PID 监控（所有者进程存活检查）
5. 30分钟空闲超时自动关闭
```

**关键修复**（v5.0.7）：
- EPERM 错误处理（跨用户PID如Tailscale SSH）
- WSL 祖父PID解析问题
- Windows/MSYS2 PID命名空间不可见问题

---

## 八、测试与质量保证体系

### 8.1 测试架构

```
tests/
├── brainstorm-server/          # 服务器功能测试
│   ├── server.test.js           # HTTP/WS 端点测试
│   └── ws-protocol.test.js      # WebSocket 协议测试
│
├── claude-code/                 # Claude Code 集成测试
│   ├── test-subagent-driven-development.sh
│   ├── test-document-review-system.sh
│   └── run-skill-tests.sh       # 技能触发测试
│
├── explicit-skill-requests/     # 显式技能请求测试
│   ├── prompts/                 # 各种用户提示场景
│   └── run-all.sh              # 全量回归
│
├── skill-triggering/            # 技能自动触发测试
│   ├── prompts/                 # 触发条件测试用例
│   └── run-test.sh             # 触发验证
│
├── subagent-driven-dev/         # SDD 端到端测试
│   ├── go-fractals/             # Go语言项目测试
│   └── svelte-todo/             # Svelte项目测试
│
└── opencode/                    # OpenCode 平台测试
    ├── test-plugin-loading.sh
    └── test-tools.sh
```

### 8.2 压力测试方法论

`systematic-debugging` 技能中的测试文件展示了压力测试架构：

```
systematic-debugging/
├── test-academic.md           # 学术问题：代理理解规则吗？
├── test-pressure-1.md         # 压力场景：时间压力
├── test-pressure-2.md         # 压力场景：沉没成本
└── test-pressure-3.md         # 压力场景：权威压力 + 疲惫
```

**架构洞察**：不仅测试代理"能否做"，更测试代理"在压力下是否还会做"——这是行为塑造系统的核心验证。

---

## 九、技术债务与架构演进

### 9.1 版本演进关键决策

| 版本 | 架构决策 | 理由 |
|------|---------|------|
| v5.0.3 | Cursor 支持 | 新增 hooks-cursor.json 适配 camelCase 格式 |
| v5.0.4 | 审查循环精简 | 从5轮降到3轮，从分块审查到全计划审查 |
| v5.0.5 | ESM → CJS 回退 | `server.js` → `server.cjs` 解决 Node.js 22 `type: module` 冲突 |
| v5.0.6 | **子代理审查 → 行内自审查** | 回归测试证明无质量提升但成本高50倍 |
| v5.0.7 | Copilot CLI 支持 | 新增 `additionalContext` 注入 |

### 9.2 当前架构张力

1. **多平台复杂性 vs 零依赖**：每新增一个平台，维护成本线性增长
2. **技能简洁性 vs 完整性**：Token 效率要求 vs 防止漏洞需要的详细程度
3. **自动化 vs 人类监督**：SDD 可在无人工干预下运行数小时，但关键决策仍需人类

### 9.3 94% PR 拒绝率的架构含义

CLAUDE.md 中声明：
> "This repo has a 94% PR rejection rate."

这不是技术问题，而是**架构原则**：
- 技能是行为代码，不是文档
- 修改技能需要评估证据（before/after eval results）
- 未经测试的技能变更等同于未经测试的生产代码变更
- "人类伙伴"（human partner）的参与是强制要求

---

## 十、源码研读启示

### 10.1 对 OpenClaw 生态的借鉴

| Superpowers 模式 | OpenClaw 对应 |
|-----------------|--------------|
| Skills 系统 | OpenClaw Skills 体系 |
| 插件适配层 | OpenClaw 的 Channel/Platform 抽象 |
| SDD 引擎 | OpenClaw 的 Subagent 系统 |
| TDD 技能 | 工具质量保障方法论 |
| 反理性化设计 | Prompt 工程中的"防止越狱" |

### 10.2 架构设计模式总结

1. **文档即代码** — 将自然语言约束视为可执行规范
2. **压力测试哲学** — 在极端条件下验证行为，非理想条件下
3. **元工作流** — 技能的创建过程本身遵循与被创建技能相同的原则
4. **渐进披露** — Description → Overview → Detail 的三层信息架构
5. **零依赖边界** — 用架构约束换取部署确定性和维护简单性

### 10.3 关键代码片段赏析

**OpenCode 启动注入的精妙设计**（解决双重问题）：

```javascript
// 不使用 system message（避免每轮重复）
// 不使用 multiple system messages（避免 Qwen 崩溃）
// 而是 prepend 到第一个 user message
firstUser.parts.unshift({ 
  ...ref, 
  type: 'text', 
  text: bootstrap 
});
```

这是**平台约束驱动的架构创新**——不是理论优雅，而是实际问题的务实解决。

---

## 十一、结论

Superpowers 是一个**元级软件工程框架**——它不直接生成产品代码，而是塑造生成代码的代理的行为模式。其架构价值在于：

1. **方法论的产品化**：将 TDD、系统调试、代码审查等"软技能"编码为可执行规范
2. **行为工程**：将心理学原理（Cialdini 的说服力原则）应用于AI代理的行为塑造
3. **跨平台抽象**：在7+个异构AI平台间维持一致的用户体验
4. **实证优化**：每个架构决策（如子代理审查→行内自审查）都有回归测试支撑

**核心启示**：
> 当AI代理成为代码的生产者时，我们需要的不只是更好的API，而是**更好的行为塑造系统**。

Superpowers 正是这一范式的先驱实践。

---

*报告生成时间: 2026-04-28 03:00 CST*  
*分析仓库: https://github.com/obra/superpowers*  
*每日代码架构分析系列 | OpenClaw*
