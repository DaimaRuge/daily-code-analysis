# 🔥 代码架构分析与源码研读报告

> **分析项目**: [planning-with-files](https://github.com/OthmanAdi/planning-with-files)  
> **Stars**: 22,492 | **Forks**: 1,994 | **Version**: v2.43.0  
> **分析日期**: 2026-06-02  
> **报告作者**: OpenClaw Daily Code Analysis  
> **项目定位**: Manus-style 持久化 Markdown 规划工作流实现，Meta 以 $2B 收购的 Manus 背后的核心模式开源实现

---

## 目录

1. [项目概述与背景](#1-项目概述与背景)
2. [整体架构设计](#2-整体架构设计)
3. [核心模块深度解析](#3-核心模块深度解析)
4. [多 IDE 适配架构](#4-多-ide-适配架构)
5. [设计模式与关键技术](#5-设计模式与关键技术)
6. [数据流与状态管理](#6-数据流与状态管理)
7. [安全机制](#7-安全机制)
8. [测试策略](#8-测试策略)
9. [源码质量评估](#9-源码质量评估)
10. [可借鉴的设计思想](#10-可借鉴的设计思想)
11. [总结与评价](#11-总结与评价)

---

## 1. 项目概述与背景

### 1.1 项目背景

`planning-with-files` 是一个实现 **Manus-style 持久化 Markdown 规划** 的开源项目。Manus 是一家被 Meta 以 **20 亿美元** 收购的 AI Agent 公司，其核心工作流模式就是：**将 AI 的"工作记忆"持久化到文件系统，而非仅依赖有限的上下文窗口**。

### 1.2 核心思想

```
Context Window = RAM (易失性, 有限)
Filesystem = Disk (持久性, 无限)

→ 任何重要信息都写入磁盘。
```

这个项目的核心洞察非常深刻：当 AI Agent 执行超过 50+ 个 tool call 后，原始目标往往会被遗忘。通过将计划、发现、进度分别写入三个 Markdown 文件，AI 可以像人类一样拥有一个"外脑"。

### 1.3 项目规模

| 指标 | 数值 |
|------|------|
| 总代码行数 | ~43,689 行 |
| 支持 IDE 数量 | 10+ (Claude Code, Cursor, Continue, Hermes, Kiro, Pi, OpenCode, Gemini, Codex, CodeBuddy, Factory 等) |
| 核心 Python 模块 | 6 个 |
| Shell 脚本 | 14+ 个 |
| 测试文件 | 22 个 |
| 版本发布 | v2.43.0 (持续迭代) |

---

## 2. 整体架构设计

### 2.1 架构全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                      User Project Directory                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ task_plan.md │  │  findings.md  │  │ progress.md  │        │
│  │   (计划)      │  │   (发现)      │  │   (进度)      │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
│         ▲                 ▲                 ▲                 │
│         │                 │                 │                 │
│  ┌──────┴─────────────────┴─────────────────┴──────┐           │
│  │              planning-with-files Skill             │           │
│  │  ┌──────────────┐  ┌──────────────┐              │           │
│  │  │   Templates  │  │   Scripts    │              │           │
│  │  │  (markdown)  │  │  (sh/py/ps1) │              │           │
│  │  └──────────────┘  └──────────────┘              │           │
│  └─────────────────────────────────────────────────┘           │
│                           ▲                                      │
│                           │                                      │
│  ┌────────────────────────┼────────────────────────────────────┐  │
│  │           IDE Plugin/Adapter Layer                        │  │
│  │  ┌─────┐ ┌──────┐ ┌───────┐ ┌────────┐ ┌──────────┐     │  │
│  │  │Hermes│ │Claude│ │Cursor │ │Continue│ │OpenCode  │ ... │  │
│  │  └─────┘ └──────┘ └───────┘ └────────┘ └──────────┘     │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 三层架构模型

项目采用清晰的 **三层架构**：

| 层级 | 职责 | 代表组件 |
|------|------|----------|
| **技能层 (Skill)** | Markdown 模板、工作流规则、使用文档 | `templates/`, `SKILL.md`, `commands/` |
| **适配层 (Adapter)** | IDE 特定的插件/Hook 实现 | `.hermes/plugins/`, `.cursor/hooks.json`, `.claude-plugin/` |
| **运行时层 (Runtime)** | Shell 脚本、Python 工具、状态管理 | `scripts/`, `session-catchup.py`, `check-complete.sh` |

---

## 3. 核心模块深度解析

### 3.1 核心数据模型：三文件规划系统

这是整个项目的灵魂。三个 Markdown 文件构成一个完整的"外脑"系统：

#### `task_plan.md` — 任务计划（战略层）

```markdown
# Task Plan: [简要描述]

## Goal
[一句话描述最终目标]

## Current Phase
Phase 1

## Phases
### Phase 1: Requirements & Discovery
- [ ] Understand user intent
- [ ] Identify constraints
- **Status:** in_progress

### Phase 2: Planning & Structure
- [ ] Define technical approach
- **Status:** pending
...
```

**设计亮点**：
- 使用 Markdown 结构，人类可读、AI 可解析
- 状态标记系统：`pending` → `in_progress` → `complete` → `failed`
- Phase 数量统计：支持通过 `**Status:**` 格式或 `[complete]` 内联标记统计
- 错误追踪区：专门的 `## Errors Encountered` 表格区域

#### `findings.md` — 研究发现（知识层）

记录所有研究、发现、决策依据。避免"2-Action Rule"失效（即每做2个浏览/搜索操作后，必须保存关键发现）。

#### `progress.md` — 进度日志（执行层）

按时间顺序记录会话中所有操作。包含：
- 开始/结束时间戳
- 具体执行的动作
- 创建/修改的文件
- 遇到的问题和解决方案

### 3.2 Python 核心模块架构

位于 `.hermes/plugins/planning-with-files/` 目录下，是 Hermes IDE 的适配实现：

```python
# __init__.py — 插件注册入口
├── register(ctx)          # 注册 3 个 Tool + 2 个 Hook
│
├── tools.py               # 工具函数层
│   ├── planning_with_files_init()      # 初始化三文件
│   ├── planning_with_files_status()    # 状态摘要
│   └── planning_with_files_check_complete()  # 完成检查
│
├── hooks.py               # 生命周期钩子层
│   ├── pre_llm_call()      # LLM 调用前注入上下文
│   └── post_tool_call()    # 工具调用后触发提醒
│
├── planning_files.py      # 文件操作核心层
│   ├── ensure_planning_files()         # 确保文件存在
│   ├── phase_counts()       # 阶段统计
│   ├── extract_current_phase()         # 提取当前阶段
│   └── summarize_status()  # 状态汇总
│
├── paths.py               # 路径解析层
│   ├── resolve_skill_dir()  # 解析技能目录
│   ├── find_skill_dir()     # 查找技能目录
│   └── normalize_cwd()      # 标准化工作目录
│
├── constants.py           # 常量定义
│   └── PLANNING_FILES, PLAN_PREVIEW_LINES, etc.
│
└── hook_state.py          # 会话状态管理
    ├── add_reminder()       # 添加提醒
    └── pop_reminders()      # 弹出提醒（FIFO）
```

### 3.3 `pre_llm_call` Hook 机制

这是整个架构中最精妙的部分。在每次 LLM 调用前，自动将规划上下文注入到用户消息中：

```python
def pre_llm_call(**kwargs: Any) -> dict[str, str] | None:
    project_dir = normalize_cwd()
    if not (project_dir / "task_plan.md").exists():
        return None
    
    # 构建上下文
    parts = ["[planning-with-files] ACTIVE PLAN — current state:"]
    
    # 1. 读取 task_plan.md 前 N 行（计划概览）
    head = head_lines(task_plan, READ_PREVIEW_LINES)
    
    # 2. 读取 progress.md 尾部（最近进度）
    progress = tail_lines(project_dir / "progress.md", PROGRESS_TAIL_LINES)
    
    # 3. 检查 findings.md 是否存在
    if findings.exists():
        parts.append("Read findings.md for research context.")
    
    # 4. 注入任何待处理的提醒（如 post_tool_call 触发的）
    reminder_messages = pop_reminders(session_id)
    
    return {"context": "\n\n".join(parts)}
```

**核心设计思想**：
- **非侵入式**：只在 `task_plan.md` 存在时生效，不会干扰不使用规划的用户
- **上下文注入**：将规划内容自动送入 LLM 的上下文窗口，如同"内存加载"
- **KV-Cache 友好**：只注入固定格式的前缀和尾部内容，避免重复加载大文件

### 3.4 `post_tool_call` Hook 机制

在文件写入/修改工具调用后，自动触发进度更新提醒：

```python
def post_tool_call(**kwargs: Any) -> None:
    tool_name = str(kwargs.get("tool_name", ""))
    
    # 只监听 write_file 和 patch 工具
    if tool_name not in ("write_file", "patch"):
        return None
    
    # 添加一个提醒到会话队列
    add_reminder(session_id, 
        "[planning-with-files] Update progress.md with what you just did.")
```

**设计精妙之处**：
- 使用**延迟提醒**模式（添加到队列而非立即执行），避免打断当前操作流
- 在下一个 `pre_llm_call` 时统一弹出，合并到一次上下文注入中

---

## 4. 多 IDE 适配架构

这是项目最 impressive 的架构设计。它通过一个 **"Write Once, Adapt Everywhere"** 的策略，支持 10+ 种 IDE/Agent：

### 4.1 适配矩阵

| IDE/Agent | 适配方式 | 核心文件 | 自动化程度 |
|-----------|----------|----------|------------|
| **Claude Code** | 原生 Skill + Hooks | `.claude-plugin/`, `commands/` | 全自动 (Hooks) |
| **Hermes** | 插件系统 | `.hermes/plugins/`, `.hermes/skills/` | 全自动 (pre/post hooks) |
| **Cursor** | 自定义 Hooks | `.cursor/hooks.json`, `.cursor/skills/` | 半自动 |
| **Continue** | 技能包 | `.continue/skills/` | 手动触发 |
| **Kiro** | 技能适配 | `.kiro/skills/` | 手动触发 |
| **Pi** | 扩展包 | `.pi/skills/` | 扩展集成 |
| **OpenCode** | 技能适配 | `.opencode/skills/` | 手动触发 |
| **Gemini CLI** | 技能适配 | `.gemini/skills/` | 手动触发 |
| **Codex** | 技能适配 | `.codex/skills/` | 手动触发 |
| **CodeBuddy** | 技能适配 | `.codebuddy/skills/` | 手动触发 |
| **Factory** | 技能适配 | `.factory/skills/` | 手动触发 |

### 4.2 同步机制：单一事实源

项目使用 `sync-ide-folders.py` 脚本保持所有 IDE 变体同步到同一版本：

```python
# 核心逻辑：以 canonical SKILL.md 为源，同步到所有 IDE 变体
def sync_to_all_ide_variants():
    canonical = load_canonical_skill_md()  # 从 .hermes/skills/ 读取
    for ide in ALL_SUPPORTED_IDES:
        variant = adapt_for_ide(canonical, ide)  # 适配 IDE 特定语法
        write_to_ide_folder(variant, ide)
```

**版本一致性策略**：
- 使用 `test_skill_md_version_parity.py` 测试确保所有 IDE 变体版本号一致
- `CHANGELOG.md` 驱动版本发布，每个版本更新所有变体
- 已有 9 个版本同步的修复记录（如 v2.43.0 将 `.continue`、`.gemini`、`.kiro` 从落后版本追平）

---

## 5. 设计模式与关键技术

### 5.1 模式一：文件系统作为状态机

```
           ┌─────────────────┐
           │   pending       │
           │   (未开始)       │
           └────────┬────────┘
                    │ 开始工作
                    ▼
           ┌─────────────────┐
           │  in_progress    │ ◄── 写入 progress.md
           │   (进行中)       │     更新 task_plan.md
           └────────┬────────┘
                    │ 完成
                    ▼
           ┌─────────────────┐
           │   complete      │
           │   (已完成)       │
           └─────────────────┘
                    │ 遇到问题
                    ▼
           ┌─────────────────┐
           │   failed        │
           │   (失败/阻塞)    │
           └─────────────────┘
```

整个工作流的状态转换由 Markdown 文件中的文本标记驱动，而非内存中的对象状态。这是**声明式状态管理**的极致体现。

### 5.2 模式二：Hook 驱动的 AOP 编程

项目大量应用**面向切面编程 (AOP)** 思想：

- `pre_llm_call`：在 LLM 调用前**横切**注入上下文
- `post_tool_call`：在工具调用后**横切**触发提醒
- 无需修改 IDE 核心代码，通过 Hook 注册实现功能扩展

### 5.3 模式三：Slug 模式（多任务隔离）

v2.40+ 引入的 **Slug 模式** 是一个精巧的设计：

```bash
# 传统模式：单任务
./task_plan.md
./findings.md
./progress.md

# Slug 模式：多任务并行隔离
.planning/
├── .active_plan          ← 当前激活的任务标记
├── 2026-06-01-backend-refactor/
│   ├── task_plan.md
│   ├── findings.md
│   └── progress.md
├── 2026-06-01-frontend-rewrite/
│   ├── task_plan.md
│   ├── findings.md
│   └── progress.md
└── 2026-06-01-bug-fix-147/
    ├── task_plan.md
    ├── findings.md
    └── progress.md
```

**解析算法**（`resolve-plan-dir.sh`）：
1. 环境变量 `$PLAN_ID` → 直接定位
2. `.planning/.active_plan` 文件内容 → 读取当前激活的任务
3. 最新的 `.planning/<dir>/`（按 mtime 排序）→ 默认回退
4. 传统模式 `./task_plan.md` → 向后兼容

### 5.4 模式四：Attestation（防篡改）机制

`attest-plan.sh` 实现了**内容完整性校验**：

```bash
# 1. 计算 task_plan.md 的 SHA-256 哈希
sha256sum task_plan.md > .plan-attestation

# 2. 后续 Hook 读取时校验
if [ "$(sha256sum task_plan.md)" != "$(cat .plan-attestation)" ]; then
    echo "[PLAN TAMPERED] 计划已被修改，请重新确认"
fi
```

**设计意图**：防止 AI 在未经明确指令的情况下自动修改计划，确保人工可控性。

### 5.5 关键技术：跨平台可移植性

项目对 POSIX 兼容性做了极致追求：

```bash
# mtime_of() 函数：5 层 fallback 机制
mtime_of() {
    target="$1"
    # 1. GNU stat
    out="$(stat -c '%Y' "${target}" 2>/dev/null)"
    # 2. BSD stat
    out="$(stat -f '%m' "${target}" 2>/dev/null)"
    # 3. BSD date -r
    out="$(date -r "${target}" +%s 2>/dev/null)"
    # 4. Python
    python3 -c "import os,sys;print(int(os.stat(sys.argv[1]).st_mtime))"
    # 5. Perl
    perl -e 'print((stat($ARGV[0]))[9])'
}
```

---

## 6. 数据流与状态管理

### 6.1 完整数据流图

```
User Input
    │
    ▼
┌─────────────────┐     ┌─────────────────┐
│  pre_llm_call   │────▶│  读取 task_plan │
│    Hook         │     │  progress.md    │
└─────────────────┘     │  findings.md     │
                        └────────┬────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │  构建上下文注入   │
                        │  "ACTIVE PLAN"   │
                        │  + 最近进度       │
                        │  + 提醒队列       │
                        └────────┬────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │   LLM 推理       │
                        │  (with context)  │
                        └────────┬────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │  Tool Execution  │
                        │  (write/patch)   │
                        └────────┬────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │ post_tool_call  │
                        │    Hook         │
                        │ 添加提醒到队列  │
                        └─────────────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
            ┌─────────┐  ┌─────────┐  ┌─────────┐
            │task_plan│  │findings │  │progress │
            │ .md     │  │ .md     │  │ .md     │
            └─────────┘  └─────────┘  └─────────┘
```

### 6.2 会话恢复机制（Session Catchup）

`session-catchup.py` 是另一个精妙设计：

```python
def detect_ide() -> str:
    # 根据环境变量和目录结构检测 IDE 类型
    # 支持 Claude Code (.claude/projects/) 和 OpenCode (.local/share/opencode/)

def get_sessions_sorted(project_dir) -> List[Path]:
    # 按 mtime 排序所有会话文件

def scan_for_context_sync_gap() -> Optional[Tuple]:
    # 1. 找到所有 planning 文件的最后修改时间
    # 2. 找到所有会话文件
    # 3. 检测"文件已更新但会话未读取"的间隙
    # 4. 返回需要同步的上下文范围
```

**核心场景**：用户在 IDE A 中工作，然后切换到 IDE B，如何确保 B 知道 A 做了什么？

---

## 7. 安全机制

### 7.1 多层安全设计

| 安全层 | 机制 | 实现 |
|--------|------|------|
| **文件安全** | 原子写入 | `attest-plan.sh` 使用 temp-rename 模式 |
| **并发安全** |  advisory lock | `flock` 可选加锁 |
| **命令安全** | 危险命令检测 | Pi 扩展中的 `isDangerousBashCommand` |
| **内容安全** | 防篡改 | SHA-256 attestation |
| **路径安全** | Slug 校验 | 安全标识符正则 `^[A-Za-z0-9_][A-Za-z0-9._-]*$` |
| **脚本安全** | Windows 兼容 | `skipif(sys.platform == "win32")` 跳过 exec-bit 测试 |

### 7.2 安全边界声明

在 SKILL.md 中明确声明：

> "Planning files go in your project root, not the skill installation folder."

这防止了技能模板被意外覆盖，也避免了用户数据与技能代码混在一起。

---

## 8. 测试策略

### 8.1 测试架构

```
tests/
├── test_hermes_adapter.py          # Hermes 插件核心测试
├── test_hook_body_v240.py          # Hook 体 v2.40 兼容性测试
├── test_hook_resolver_integration.py # Hook 解析器集成测试
├── test_check_complete_resolver.py   # 完成检查解析器测试
├── test_resolve_plan_dir.py          # 路径解析测试
├── test_plan_attestation.py          # 防篡改测试
├── test_session_catchup.py           # 会话恢复测试
├── test_session_catchup_opencode.py  # OpenCode 会话恢复测试
├── test_set_active_plan.py           # 激活计划测试
├── test_init_session_slug.py         # Slug 模式初始化测试
├── test_script_permissions.py        # 脚本权限测试
├── test_canonical_script_sync.py     # 脚本同步测试
├── test_skill_md_version_parity.py   # 版本一致性测试
├── test_codex_hooks.py               # Codex Hook 测试
├── test_codex_session_isolation.py     # Codex 会话隔离测试
├── test_pi_extension_packaging.py    # Pi 扩展打包测试
├── test_pi_extension_capabilities.py # Pi 扩展能力测试
└── test_clear_recovery.sh            # 恢复清理测试
```

### 8.2 测试覆盖率策略

| 测试类型 | 覆盖范围 | 数量 |
|----------|----------|------|
| 单元测试 | Python 核心模块 | 130+ |
| 集成测试 | Hook 全流程 | 20+ |
| 兼容性测试 | 多平台 (GNU/BSD/macOS/Windows) | 跨平台 |
| 版本一致性测试 | 所有 IDE 变体 | 每次发布 |

**测试结果**：v2.40 达到 **130 pass / 2 pre-existing Windows exec-bit fails**。

---

## 9. 源码质量评估

### 9.1 代码质量评分

| 维度 | 评分 | 说明 |
|------|------|------|
| **可读性** | ⭐⭐⭐⭐⭐ | Markdown 模板大量注释，自文档化；Python 代码简洁明了 |
| **可维护性** | ⭐⭐⭐⭐⭐ | 模块化清晰，22 个测试文件覆盖核心逻辑；CHANGELOG 详细 |
| **可扩展性** | ⭐⭐⭐⭐⭐ | 新增 IDE 只需添加对应文件夹和适配器；Slug 模式支持并行 |
| **跨平台** | ⭐⭐⭐⭐⭐ | 5 层 fallback 的 mtime 解析；POSIX 兼容脚本；Windows PS1 脚本 |
| **安全性** | ⭐⭐⭐⭐⭐ | Attestation 防篡改；危险命令检测；路径安全校验 |
| **文档** | ⭐⭐⭐⭐⭐ | 每个模板都有详细的 HTML 注释；多语言命令支持 (EN/ES/DE/AR/ZH) |
| **工程化** | ⭐⭐⭐⭐⭐ | GitHub Actions 自动化；版本管理规范；社区贡献者 100+ |

### 9.2 代码风格亮点

1. **渐进式解析策略**：`phase_counts()` 函数实现 3 层 fallback 解析（`**Status:**` → `|table|` → `[marker]`），最大化兼容性
2. **防御性编程**：所有文件操作都有 `exists()` 检查；所有正则都有边界检查
3. **最小惊讶原则**：`check-complete.sh` 总是 exit 0，通过 stdout 报告状态，避免中断 Agent 循环

---

## 10. 可借鉴的设计思想

### 10.1 对 AI Agent 架构的启示

1. **文件系统即状态管理**
   - 不要依赖易失的内存状态，将关键状态持久化到文件
   - Markdown 是人类和 AI 都能理解的通用格式

2. **上下文注入而非上下文加载**
   - 不是把整个文件塞给 LLM，而是提取关键片段（head + tail）注入
   - 这保持 KV-Cache 稳定，减少 token 浪费

3. **延迟提醒 + 批量注入**
   - `post_tool_call` 添加提醒，`pre_llm_call` 统一弹出
   - 避免每次操作都触发一次上下文注入

4. **声明式状态 > 命令式状态**
   - 修改 Markdown 中的文本标记，而非调用 API 更新状态
   - 状态是"读出来的"，不是"算出来的"

### 10.2 对多平台适配的启示

1. **Canonical Source + Adaptation Layer**
   - 单一事实源（canonical SKILL.md），所有变体从它派生
   - 使用自动化脚本保持同步，而非手动复制

2. **渐进降级策略**
   - 功能优先度：全自动 Hook > 半自动命令 > 手动触发
   - 不同 IDE 能力不同，适配到各自能达到的最高自动化程度

### 10.3 对工程化的启示

1. **向后兼容是生命线**
   - v2.40 引入 Slug 模式，但传统模式（root-level 文件）完全保留
   - 所有新功能都是 opt-in，不破坏现有用户

2. **测试即文档**
   - 22 个测试文件本身就是最好的使用说明
   - 测试覆盖了所有边缘场景（Windows 权限、并发写入、跨平台路径等）

---

## 11. 总结与评价

### 总体评价

`planning-with-files` 是一个**架构设计精良、工程实践成熟**的开源项目。它成功地将一个高价值的产品模式（Manus-style 规划）转化为可复用的开源基础设施，并且在以下方面表现出色：

1. **架构创新**：将文件系统作为 AI 状态机的载体，Hook 驱动的 AOP 编程模式
2. **工程严谨**：22 个测试文件、多平台兼容、版本一致性检查、向后兼容策略
3. **生态开放**：支持 10+ IDE，社区 Fork 活跃，形成技能生态

### 核心贡献

这个项目最大的贡献是证明了：**AI Agent 的"记忆"不应该只在内存中，而应该像人类一样有一个外化的笔记本系统**。这个洞察简单却深刻，直接影响了 Agent 架构的设计范式。

### 推荐学习路径

| 优先级 | 学习内容 | 文件 |
|--------|----------|------|
| 🔴 最高 | Hook 机制 | `hooks.py`, `__init__.py` |
| 🔴 最高 | 模板设计 | `templates/task_plan.md` |
| 🟡 高 | 路径解析 | `paths.py`, `resolve-plan-dir.sh` |
| 🟡 高 | 状态管理 | `planning_files.py`, `hook_state.py` |
| 🟢 中 | 多 IDE 适配 | `sync-ide-folders.py`, 各 IDE 目录 |
| 🟢 中 | 会话恢复 | `session-catchup.py` |
| ⚪ 低 | 安全机制 | `attest-plan.sh`, `test_plan_attestation.py` |

---

*报告生成时间: 2026-06-02 03:00 CST*  
*分析工具: OpenClaw Code Analysis Agent*  
*项目来源: [GitHub - OthmanAdi/planning-with-files](https://github.com/OthmanAdi/planning-with-files)*
