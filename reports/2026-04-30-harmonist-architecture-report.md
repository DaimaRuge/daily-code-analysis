# 技术架构与源码研读报告

## GammaLabTechnologies/harmonist — 便携式AI Agent编排框架

**分析日期**: 2026-04-30  
**报告版本**: v1.0  
**分析者**: OpenClaw Daily Code Analysis  
**项目**: [GammaLabTechnologies/harmonist](https://github.com/GammaLabTechnologies/harmonist)  
**Stars**: 878 | **Agents**: 186 | **License**: MIT | **Dependencies**: stdlib only

---

## 一、项目定位与核心价值主张

Harmonist 是一个**便携式AI Agent编排框架**，专为 Cursor、Claude Code、Copilot、Windsurf、Aider 等AI编程助手设计。其核心差异化在于：**协议执行是机械门控，而非提示中的礼貌请求**。

当前AI编程框架的痛点：
- **薄框架**（LangChain, CrewAI, AutoGen等）提供编排原语，但将执行留给提示——模型可以随时覆盖自己的协议
- **重企业平台**通过独立运行时、数据库和供应商锁定来承诺治理——但需要基础设施安装，无法在单个开发者的笔记本上工作

Harmonist 选择了第三条路：**零运行时依赖、文件级可审计、机械强制执行的轻量级编排**。

---

## 二、架构全景图

```
┌─────────────────────────────────────────────────────────────┐
│                    IDE / AI Assistant                        │
│         (Cursor, Claude Code, Windsurf, Aider...)           │
└────────────────────────┬──────────────────────────────────────┘
                         │ hooks.json (5 lifecycle events)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Mechanical Protocol Enforcement                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ hook_runner │  │  session    │  │   state machine     │  │
│  │   .py       │  │    .json    │  │  (5 phases)         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│         │                                               │     │
│         ▼                                               ▼     │
│  ┌─────────────┐                               ┌─────────┐ │
│  │ telemetry/  │                               │incidents│ │
│  │agent-usage │                               │ .json   │ │
│  └─────────────┘                               └─────────┘ │
└─────────────────────────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐
│  agents/        │ │  memory/    │ │  playbooks/     │
│  186 agent defs │ │ structured  │ │ 6-phase project │
│  YAML frontmatter│ │ validated   │ │ lifecycle       │
│  + Markdown body │ │ markdown    │ │                 │
└─────────────────┘ └─────────────┘ └─────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Supply Chain Integrity Layer                    │
│         MANIFEST.sha256 — every pack file hashed             │
│    verify_integration.py / build_manifest.py / CI gates      │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、核心子系统深度分析

### 3.1 机械协议执行层 (Hooks)

**文件**: `hooks/scripts/hook_runner.py` (≈500行)  
**配套**: `hooks/hooks.json`, `hooks/scripts/*.sh` (POSIX fallback)

这是整个框架的灵魂。Hook Runner 实现了**跨平台的Python状态机**，替代了原来仅支持macOS/Linux的bash脚本，使Windows原生Cursor也能使用完整的执行层。

#### 3.1.1 五大生命周期阶段

| Phase | 触发时机 | 核心职责 |
|-------|---------|---------|
| `sessionStart` | 新会话开始 | 重置状态、检测协议跳过滥用、注入项目上下文、加载最近记忆 |
| `afterFileEdit` | 每次文件编辑后 | 记录写入路径、检测readonly agent违规写入、记录memory文件更新 |
| `subagentStart` | 子agent启动 | 解析 `AGENT: <slug>` 标记、记录调用到状态、递增遥测计数器 |
| `subagentStop` | 子agent完成 | 关闭最老的开放调用、如slug属于reviewer集合则标记为已见 |
| `stop` | 响应即将完成 | **核心门控** — 验证所有协议要求是否满足，否则返回followup_message阻止完成 |

#### 3.1.2 状态机设计

```python
state = {
    "session_id": "<unix-seconds><pid4>",      # 碰撞防护
    "task_seq": 0,                              # 任务序列号
    "active_correlation_id": "<sid>-<seq>",    # 当前任务关联ID
    "writes": [],                               # 本次任务的文件写入记录
    "subagent_calls": [],                       # 子agent调用记录
    "reviewers_seen": [],                       # 已执行的reviewer列表
    "memory_updates": [],                       # memory文件更新记录
    "enforcement_attempts": 0,                  # 强制执行尝试次数
    "protocol_skipped": False,                  # 是否使用PROTOCOL-SKIP
}
```

**设计亮点**:
- **Fail-Closed（故障关闭）**: `loop_limit` 默认3次，超过后标记为 `protocol-exhausted`，写入 `incidents.json`，下一会话启动时自动提醒用户
- **PROTOCOL-SKIP 滥用检测**: 遥测系统跟踪跳过比例，超过25%触发警告横幅
- **Readonly 违规检测**: 如 `qa-verifier` 等 `readonly=true` 的agent被检测到写入文件，直接阻止完成

#### 3.1.3 Stop Gate 验证逻辑（核心）

```
IF 无写入 → 允许完成 (no-writes)
IF PROTOCOL-SKIP → 允许完成，记录原因 (protocol-explicitly-skipped)
IF 全部trivial路径（.md, README等）→ 允许完成 (trivial-only)

ELSE:
  检查清单:
  ✗ readonly agent 写入？→ 阻止
  ✗ 无reviewer被调用？→ 阻止
  ✗ qa-verifier 未调用？→ 阻止（可配置）
  ✗ session-handoff.md 未更新？→ 阻止
  ✗ correlation_id 不匹配？→ 阻止
  ✗ memory文件schema验证失败？→ 阻止
  ✗ 回归测试未通过？→ 阻止（可配置）
  
  失败超过 loop_limit → EXHAUSTED，记录incident
```

---

### 3.2 结构化验证内存系统 (Memory)

**文件**: `memory/memory.py` (≈700行), `memory/validate.py` (≈300行)  
**Schema**: `memory/SCHEMA.md`

#### 3.2.1 设计哲学

为什么不用SQLite/JSON file？因为**这个框架的目的是嵌入IDE，零外部依赖**。Markdown作为存储介质的原因：
1. **人类可读**: 开发者可以直接打开 `.cursor/memory/session-handoff.md` 查看历史
2. **Git友好**: diff能清晰显示变化
3. **IDE原生支持**: 无需额外工具即可浏览
4. **标准格式**: 使用 `<!-- memory-entry:start/end -->` HTML注释作为结构标记，不干扰渲染

#### 3.2.2 Entry 结构

```markdown
<!-- memory-entry:start -->
---
schema_version: 1
id: <cid>-<kind>[-N]
correlation_id: <session_id>-<task_seq>
at: 2026-04-30T03:00:00Z
kind: state | decision | pattern
status: in_progress | done | blocked | rejected
author: orchestrator | subagent | human
summary: <单行摘要，最多160字符>
tags: [tag1, tag2]
---

<body — 至少20个非空白字符>

<!-- memory-entry:end -->
```

#### 3.2.3 核心功能

| 命令 | 用途 |
|------|------|
| `append` | 追加entry — 自动生成id/correlation_id/at，验证schema，扫描secret |
| `show` | 按id查询 |
| `list` | 按条件过滤 |
| `latest` | 获取最近N条 |
| `search` | 全文搜索，支持tag/kind/status/date范围 |
| `rotate` | 归档旧entry到日期命名的sidecar文件 |
| `validate` | schema校验 |
| `current-id` | 打印当前correlation_id |
| `bump-task` | 递增task_seq |

#### 3.2.4 Secret 扫描引擎

`memory.py` 内置了**22种凭证模式**的正则检测 + **通用高熵token检测**（Shannon entropy ≥ 4.3 bits/char）：

- AWS AKIA/Secret Key
- GitHub PAT / Fine-grained / OAuth / App tokens
- OpenAI / Anthropic API keys
- Stripe keys
- Slack tokens / webhooks
- JWT tokens
- Private key PEM
- SendGrid / Mailgun / Twilio keys
- Discord / Telegram tokens
- DB URL with credentials
- Generic high-entropy tokens

**Placeholder 豁免**: `<PLACEHOLDER>`, `${VAR}`, `{{template}}`, `%%X%%` 等被自动识别为占位符，不会误报。

#### 3.2.5 重复防护

- **Summary去重**: 拒绝与任何现有entry相同summary的新append
- **Body hash去重**: 使用blake2s(8 bytes)检测内容重复但summary重写的entry
- **自动回滚**: 写入后schema验证失败自动删除刚追加的block

---

### 3.3 Agent 目录与编排系统

**文件**: `agents/` (243个.md文件), `agents/SCHEMA.md`, `agents/index.json`  
**脚本**: `agents/scripts/build_index.py`

#### 3.3.1 Agent 文件格式

每个agent是一个Markdown文件，包含YAML frontmatter + Markdown body：

```yaml
---
schema_version: 2
name: qa-verifier
description: Independently verifies that a task is actually complete...
category: review
protocol: strict | persona
readonly: true | false
is_background: false | true
model: fast | inherit | reasoning
tags: [review, qa, evidence-collection]
domains: [all]
distinguishes_from: [testing-reality-checker]
disambiguation: Strict per-task completeness gate...
version: 1.0.0
updated_at: 2026-04-22
---

<agent's system prompt — the actual behavior definition>
```

#### 3.3.2 编排器路由表 (`index.json`)

`build_index.py` 从所有agent文件生成确定性的JSON索引，供编排器查询：

```json
{
  "counts": {
    "total": 186,
    "by_category": {
      "engineering": 46,
      "marketing": 30,
      "game-development": 20,
      "specialized": 17,
      ...
    },
    "by_protocol": { "persona": 179, "strict": 7 },
    "by_domain": { "all": 133, "china-market": 17, "gamedev": 20, ... }
  },
  "by_category": { "engineering": ["slug1", "slug2", ...] },
  "by_tag": { "security": [...], "review": [...] },
  "disambiguation": { "slug": { "peers": [...], "note": "..." } }
}
```

**关键设计**:
- **slug**: 全局唯一标识，跨系统一致（文件路由、hook标记、内存entry、遥测）
- **disambiguation**: 对称关系图 — 如果A区分于B，则B的索引也自动包含A作为peer
- **version/updated_at**: 全部186个agent都包含版本信息，支持未来淘汰机制

#### 3.3.3 Agent 分类分布

| Category | Count | 典型Agent |
|----------|-------|----------|
| engineering | 46 | backend-architect, devops-automator, security-reviewer |
| marketing | 30 | seo-analyst, content-strategist, china-market-localizer |
| game-development | 20 | unity-developer, unreal-engineer, blender-pipeline |
| specialized | 17 | blockchain-developer, mcp-integrator, zk-researcher |
| design | 8 | ui-ux-designer, accessibility-auditor, brand-guardian |
| review | 6 | qa-verifier, security-reviewer, code-quality-auditor |
| orchestration | 2 | task-router, repo-mapper |

---

### 3.4 供应链完整性层 (Manifest)

**文件**: `agents/scripts/build_manifest.py`  
**输出**: `MANIFEST.sha256`

#### 3.4.1 设计动机

Harmonist 被设计为"drop-in"到项目的 `.cursor/` 目录中。但一旦被复制到项目里，用户可能无意中修改agent文件，或者恶意行为者可能篡改reviewer定义来绕过安全检查。

Manifest 是**供应链锚点** — 记录每个pack文件的SHA256哈希，任何完整性检查工具都可以验证。

#### 3.4.2 覆盖范围

```python
INCLUDE_PATTERNS = [
    "AGENTS.md", "integration-prompt.md", "VERSION", "CHANGELOG.md",
    "agents/SCHEMA.md", "agents/STYLE.md", "agents/TAGS.md", "agents/tags.json",
    "agents/*/*.md", "agents/*/*/*.md",        # 所有agent定义
    "agents/scripts/*.py", "agents/scripts/*.sh",  # 所有脚本
    "agents/templates/rules/*.mdc",            # Cursor规则模板
    "hooks/hooks.json", "hooks/scripts/*.sh",
    "memory/*.py", "memory/*.md",
]
```

**排除项**: `agents/index.json`（生成文件）, `MANIFEST.sha256`（不能自哈希）, `hooks/.state/`（运行时状态）

#### 3.4.3 验证工作流

```bash
# CI gate: 检查manifest是否过期
python3 agents/scripts/build_manifest.py --check

# 现场验证: 比较当前pack内容与现有manifest
python3 agents/scripts/build_manifest.py --verify

# 重新生成
python3 agents/scripts/build_manifest.py
```

---

### 3.5 场景化执行流程 (Playbooks)

**文件**: `playbooks/` (17个Markdown文件)

Playbooks 定义了**6阶段项目生命周期**的agent激活序列：

| Phase | 名称 | 目标 | 典型Agents |
|-------|------|------|-----------|
| 0 | Discovery | 需求探索与市场验证 | Market-Researcher, UX-Researcher |
| 1 | Strategy | 战略与架构定义 | Studio-Producer, Brand-Guardian, System-Architect |
| 2 | Foundation | 技术基础搭建 | DevOps-Automator, Security-Reviewer, Backend-Architect |
| 3 | Build | 核心功能开发 | Fullstack-Developer, AI-Engineer, Database-Architect |
| 4 | Hardening | 质量加固 | QA-Verifier, Performance-Engineer, Accessibility-Auditor |
| 5 | Launch | 发布上线 | Release-Manager, Marketing-Strategist, SRE-Observability |
| 6 | Operate | 持续运营 | Support-Analyst, Analytics-Engineer |

每个Phase Playbook包含：
- 先决条件检查清单
- Agent激活序列（精确到执行命令格式）
- 交付物定义
- Gate Keeper（谁有权批准进入下一阶段）
- 协调模板（handoff格式）

---

## 四、关键技术决策与设计理念

### 4.1 零外部依赖 (stdlib only)

**决策**: 整个框架仅使用Python标准库。

**原因**:
- **嵌入性**: 必须能在任何已安装Python 3.9+的环境中工作，无需 `pip install`
- **可审计性**: 每个文件都是人类可读的Python，没有隐藏的依赖传递
- **可移植性**: Windows/macOS/Linux/WSL 统一行为

**代价**: 没有Pydantic、没有PyYAML解析器、没有requests。所有YAML frontmatter解析、JSON处理、正则匹配都是手写的最小实现。

### 4.2 Markdown 作为数据格式

**决策**: Agent定义、内存条目、Playbook全部使用Markdown。

**优势**:
- Git diff 可读
- IDE/编辑器原生支持
- AI模型训练语料中最熟悉的格式
- 人类可以直接编辑和审查

**挑战**: 需要自定义验证器（`validate.py`），没有现成的schema验证库可用。

### 4.3 状态外置 (External State)

**决策**: Hook状态存储在JSON文件中（`.cursor/hooks/.state/session.json`），而非内存中。

**原因**:
- Cursor的hook进程每次调用都是独立的，没有持久内存
- JSON文件实现跨调用的状态共享
- 崩溃后可恢复，可审计
- 支持并发会话的碰撞防护（pid后缀）

### 4.4 对称式去歧义 (Symmetric Disambiguation)

**决策**: `distinguishes_from` 关系在索引中自动双向填充。

**示例**: 如果 `qa-verifier` 区分于 `testing-reality-checker`，则 `testing-reality-checker` 的索引条目也自动包含 `qa-verifier` 作为peer。

**价值**: 编排器查询时无需知道关系的方向，简化路由逻辑。

### 4.5 Telemetry 最小化

**决策**: 遥测仅记录计数器和时间戳，不包含代码内容、文件路径或敏感信息。

**设计**:
```json
{
  "agents": {
    "qa-verifier": { "invocations": 42, "last_at": "2026-04-30T03:00:00Z" }
  },
  "summaries": {
    "sessions": 15,
    "gate_allow_satisfied": 120,
    "gate_followups": 8,
    "gate_exhausted": 1,
    "protocol_skips": 3,
    "readonly_violations": 0
  }
}
```

---

## 五、源码质量评估

### 5.1 代码组织

| 维度 | 评分 | 说明 |
|------|------|------|
| 模块化 | ★★★★★ | 清晰的子系统边界：hooks/memory/agents/playbooks/scripts |
| 可测试性 | ★★★★★ | 430+测试，每个脚本都有test_*.sh对应测试 |
| 文档完整性 | ★★★★★ | README/SCHEMA.md/STYLE.md/GUIDE_EN.md/SECURITY.md |
| 类型安全 | ★★★★☆ | 使用Python 3.9+ type hints，但无静态类型检查器 |
| 错误处理 | ★★★★☆ | try/except + pass 模式较多，但符合"不阻断IDE"的设计哲学 |
| 性能 | ★★★★★ | O(n)扫描算法，适合<1000 entry的场景 |

### 5.2 设计模式

| 模式 | 应用 |
|------|------|
| 状态机 | Hook Runner的5-phase状态转换 |
| 策略模式 | `trivial_path_patterns` 可配置的白名单策略 |
| 模板方法 | `_render_entry` 统一的内存条目渲染 |
| 命令模式 | memory.py 的子命令体系 (append/show/list/search/rotate/validate) |
| 观察者模式 | Telemetry 计数器自动递增 |
| 备忘录模式 | session.json 快照整个会话状态 |

### 5.3 安全考量

| 层面 | 措施 |
|------|------|
| 凭证防护 | 22种正则 + 高熵token检测 + placeholder豁免 |
| 完整性 | SHA256 manifest + CI gate |
| 审计 | activity.log + incidents.json + telemetry |
| 权限隔离 | readonly agent标记 + 写入检测 |
| 版本控制 | schema_version 字段 + migrate_schema.py |

---

## 六、与同类项目的对比

| 维度 | Harmonist | LangChain | CrewAI | AutoGen |
|------|-----------|-----------|--------|---------|
| **执行模型** | 机械门控 | 提示驱动 | 提示驱动 | 对话驱动 |
| **运行时依赖** | stdlib only |  heavy | heavy | heavy |
| **可审计性** | 文件级 | 运行时 | 运行时 | 运行时 |
| **IDE集成** | 原生(Cursor hooks) | 间接 | 间接 | 间接 |
| **Reviewer强制** | 是 | 否 | 否 | 否 |
| **供应链完整性** | SHA256 manifest | 无 | 无 | 无 |
| **内存持久化** | Markdown文件 | 数据库/向量存储 | 内存 | 内存 |
| **Agent数量** | 186内置 | 少量示例 | 少量模板 | 少量模板 |

---

## 七、总结与启示

### 7.1 核心启示

1. **"机械执行"是AI Agent编排的下一个进化方向**: 提示工程有其极限，Harmonist 证明了通过IDE hook可以实现真正的协议强制执行

2. **Markdown 作为数据层是可行的**: 当目标是"人类+AI共同可读"时，放弃二进制格式和数据库是合理的权衡

3. **零依赖是一种架构选择**: stdlib-only 不是限制，而是一种刻意的设计——它定义了项目的嵌入边界

4. **供应链完整性对AI工具至关重要**: 当AI agent有权修改代码时，确保agent定义本身未被篡改是安全的基础

### 7.2 适用场景

- **个人开发者**: 想要AI助手遵循严格编码规范
- **小团队**: 没有足够的DevOps资源部署重平台
- **合规敏感项目**: 需要文件级审计跟踪
- **多IDE环境**: 团队成员使用不同的AI编程助手

### 7.3 局限性

- **绑定Cursor生态**: Hook系统依赖Cursor的hooks API，其他IDE支持有限（通过integration-prompt适配）
- **Markdown scaling**: 当memory文件增长到数千条目时，全文扫描性能会下降（rotate机制缓解但未根除）
- **无分布式能力**: 单机会话状态，不支持团队协作共享agent状态

---

## 附录：文件清单

| 路径 | 行数 | 职责 |
|------|------|------|
| `hooks/scripts/hook_runner.py` | ~500 | 跨平台协议执行状态机 |
| `hooks/hooks.json` | ~30 | IDE hook注册配置 |
| `memory/memory.py` | ~700 | 结构化内存CLI |
| `memory/validate.py` | ~300 | Schema验证引擎 |
| `agents/scripts/build_index.py` | ~200 | Agent索引生成器 |
| `agents/scripts/build_manifest.py` | ~250 | 供应链清单生成器 |
| `agents/SCHEMA.md` | ~350 | Agent定义规范 |
| `AGENTS.md` | ~600 | 工程编排器主协议 |

---

*本报告由 OpenClaw 每日代码架构分析任务自动生成。*  
*数据来源: GitHub API + 源码静态分析 | 分析模型: kimi/k2p5*
