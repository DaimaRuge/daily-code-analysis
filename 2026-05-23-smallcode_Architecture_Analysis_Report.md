# 技术架构与源码研读报告：SmallCode

> **项目：** [Doorman11991/smallcode](https://github.com/Doorman11991/smallcode)  
> **分析日期：** 2026-05-23  
> **Stars：** ~1,200+（快速增长中）  
> **核心定位：** 面向 8B–35B 参数本地小模型的 AI 编程代理  
> **分析人：** OpenClaw Daily Code Analysis

---

## 一、项目概览

SmallCode 是一个**终端原生（terminal-native）的 AI 编程代理**，其设计起点与主流工具（如 Claude Code、OpenCode）截然相反：它不为前沿大模型（128K+ 上下文、完美 JSON 输出）服务，而是专为运行在消费级硬件上的**本地小模型（8B–35B 参数）**量身定制。

| 维度 | OpenCode / Claude Code | SmallCode |
|------|------------------------|-----------|
| **目标模型** | 前沿模型（Claude、GPT-5） | 本地 8B–35B 模型 |
| **上下文策略** | 全量 dump | 预算管理 + 摘要压缩 |
| **工具调用** | 假设可靠 JSON | 容错多格式解析 |
| **规划方式** | 单步生成 | TODO 文件分解 |
| **编辑策略** | 整文件重写 | Search-and-replace Patch |
| **隐私** | API 调用至云端 | 完全本地，无需网络 |

> **核心洞察：** SmallCode 不是"更差的 Cursor"，而是在特定约束下（小模型、小上下文、不完美的工具调用）提取最大可用工作量的工程杰作。每一个架构决策都围绕这一约束展开。

---

## 二、核心架构设计哲学

### 2.1 约束驱动的设计（Constraint-Driven Design）

SmallCode 的架构不是功能堆叠，而是**对小型 LLM 固有缺陷的系统性补偿**：

| 小模型缺陷 | SmallCode 的补偿机制 |
|-----------|---------------------|
| 上下文窗口有限（8K–32K） | 两阶段工具路由、上下文预算引擎、语义压缩 |
| JSON/工具调用不可靠 | 容错解析器（支持 JSON/YAML/XML/Hermes/纯文本） |
| 多步任务易遗忘 | TODO 驱动规划 + 计划锚点注入 |
| 长文件生成易截断/幻觉 | Patch-first 编辑 + 读前写保护 |
| 推理深度不足 | 工作记忆持久化 + Evidence Store |
| 工具调用重复循环 | 早期停止检测 + 工具去重 |
| 思考预算浪费 | Thinking Budget 硬截断 |

### 2.2 核心设计原则

1. **Token 经济学（Token Economics）：** 每一个设计决策都考虑 Token 消耗。两阶段路由节省 ~60% 的工具 Schema Token；上下文驱逐策略优先丢弃旧结果而非摘要。
2. **优雅降级（Graceful Degradation）：** 所有功能都有降级路径。SQLite 不可用则回退 JSON；MarrowScript 未编译则回退正则；云升级未配置则静默跳过。
3. **确定性优先（Determinism First）：** 能用正则/规则解决的问题绝不调用 LLM。工具分类器是加权正则评分系统，零 Token 成本。
4. **观测性内建（Observability Built-In）：** Token 监控、执行追踪、预算可视化、基准测试——全部内建，非事后 bolt-on。

---

## 三、系统架构全景

### 3.1 模块分层图

```
┌─────────────────────────────────────────────────────────────┐
│  用户界面层 (TUI / CLI / API / MCP Server)                  │
│  ├─ bin/tui.js          经典 readline TUI                   │
│  ├─ src/tui/fullscreen.js  全屏交替缓冲区 TUI               │
│  ├─ src/api/index.js    程序化 API                        │
│  └─ bin/smallcode.js    --mcp 模式（JSON-RPC over stdio）  │
├─────────────────────────────────────────────────────────────┤
│  Agent 循环层 (bin/smallcode.js ~1570行)                    │
│  ├─ 启动检测（Bootstrap）                                   │
│  ├─ 消息预处理（@file 展开、git diff 注入）                  │
│  ├─ 确定性工具分类器（8 类加权评分）                         │
│  ├─ 计划追踪器（Plan Tracker）                             │
│  ├─ 对话循环（LLM 调用 → 解析 → 执行 → 验证 → 回滚）         │
│  └─ 早期停止检测（重复循环/补丁螺旋/问候回归）                │
├─────────────────────────────────────────────────────────────┤
│  工具与执行层                                               │
│  ├─ bin/tools.js        20+ 工具 Schema 定义              │
│  ├─ bin/executor.js       18 工具执行器                     │
│  ├─ src/tools/two_stage_router.js  两阶段路由决策           │
│  ├─ src/tools/read_tracker.js      读前写保护               │
│  ├─ src/tools/shell_session.js     持久化 Shell 会话        │
│  ├─ src/tools/dedup.js             工具调用去重             │
│  └─ src/tools/mcp_client.js        外部 MCP 工具接入        │
├─────────────────────────────────────────────────────────────┤
│  治理与验证层 (Governor)                                     │
│  ├─ bin/governor.js       工具评分、验证、分解              │
│  ├─ src/governor/early_stop.js   早期停止检测               │
│  └─ src/model/adaptive_router.js  自适应模型路由            │
├─────────────────────────────────────────────────────────────┤
│  模型交互层                                                 │
│  ├─ bin/model_client.js   LLM API 调用、流式处理、验证      │
│  ├─ src/model/profiles.js  模型能力画像（8+ 主流模型）        │
│  ├─ src/model/router.js   模型选择路由                      │
│  ├─ src/model/thinking_budget.js  思考预算控制              │
│  └─ bin/escalation.js     云模型降级（Claude/OpenAI/DS）     │
├─────────────────────────────────────────────────────────────┤
│  认知层 (MarrowScript 编译产物)                              │
│  ├─ src/compiled/cognition/    提示缓存、追踪、验证、路由     │
│  ├─ src/compiled/features/     微任务（修复、摘要、分类）     │
│  ├─ src/compiled/flows.js      Saga 流运行时                │
│  ├─ src/compiled/logger.js     结构化 JSON 日志             │
│  └─ src/compiled/metrics.js    指标收集（counter/histogram） │
├─────────────────────────────────────────────────────────────┤
│  记忆与持久化层                                             │
│  ├─ bin/memory.js / budget-aware-mcp   SQLite + FTS5       │
│  ├─ src/memory/evidence.js    Evidence Store（经验学习）     │
│  ├─ src/session/persistence.js 会话持久化（原子写入）         │
│  ├─ src/session/snapshot.js   快照与自动回滚                  │
│  └─ src/session/undo.js       撤销栈                        │
├─────────────────────────────────────────────────────────────┤
│  扩展层                                                     │
│  ├─ src/plugins/loader.js   插件系统                       │
│  ├─ src/plugins/skills.js   技能系统（6 个内置技能）         │
│  ├─ extensions/*.ts         TypeScript 扩展模板              │
│  └─ knowledge/loader.js     知识注入（关键词匹配）            │
├─────────────────────────────────────────────────────────────┤
│  基准与测试层                                               │
│  ├─ bench/harness.js        基准测试框架                     │
│  └─ bin/eval_runner.js      提示评估（分类准确率等）          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 目录结构映射

| 目录 | 职责 |
|------|------|
| `bin/` | 运行时代码（入口点、工具执行、TUI、配置加载） |
| `src/` | 核心库源码（编译产物、模型路由、会话管理、插件） |
| `src/compiled/` | MarrowScript 编译生成的 TypeScript/JS |
| `src/tools/` | 工具路由、MCP 客户端、Shell 会话、读写追踪 |
| `src/session/` | 会话持久化、计划追踪、快照、撤销、Token 追踪 |
| `src/model/` | 模型画像、自适应路由、思考预算、链式调用 |
| `src/governor/` | 早期停止检测 |
| `src/plugins/` | 插件加载器、技能管理器 |
| `src/memory/` | Evidence Store |
| `src/knowledge/` | 知识注入加载器 |
| `bench/` | 基准测试套件 |
| `skills/` | 6 个内置开发方法论技能 |
| `profiles/` | 模型配置文件（TOML） |
| `marrow/` | MarrowScript 源文件（声明式认知层） |

---

## 四、关键模块深度解析

### 4.1 Agent 循环（`bin/smallcode.js`）

这是整个系统的**心脏**，约 1570 行代码，包含完整的 Agent Loop。其执行流程如下：

```
用户输入
  → 启动检测（BootstrapDetector）注入项目上下文
  → 消息预处理（@file 展开、图像路径、git diff 注入）
  → 模糊性检测（regex 分类器）→ 过模糊则要求澄清
  → 确定性工具分类器（8 类加权评分）
      ├─ respond → 不注入任何工具（节省 ~800 Token）
      ├─ write/run/search/... → 仅注入相关工具
  → 计划检测（shouldPlan）→ 多步任务则要求先输出计划
  → 构建 Prompt（系统消息 + 历史 + 工具 Schema + 计划锚点）
  → LLM 调用（streaming，支持多提供商格式转换）
  → 工具调用解析（容错 JSON/YAML/XML/Hermes/纯文本）
  → 工具执行（executor.js）
  → 验证（编译检查 + 运行检查）
      ├─ 通过 → 记录成功，继续
      ├─ 失败 → 重试（最多 2 次）
      └─ 硬失败 → 分解（decompose）或升级（escalation）
  → 早期停止检测
  → 上下文预算检查（mid-turn eviction）
  → 循环或结束
```

**关键洞察：** Agent Loop 不是简单的 "LLM → 工具 → LLM" 循环，而是**一个包含多层防护和补偿机制的复杂状态机**。

### 4.2 确定性工具分类器（`src/tools/two_stage_router.js`）

这是 SmallCode 最具创新性的设计之一。

**问题：** 20 个工具的全量 Schema 约占用 ~2000 Token，对于 8K 上下文窗口的小模型，这意味着 25% 的上下文被工具定义占据。

**解决方案：**

1. **直接模式（Direct Mode）**：上下文 >16K 时，一次性发送所有工具 Schema。
2. **两阶段模式（Two-Stage Mode）**：上下文 ≤16K 时：
   - **阶段 1：** 仅发送一个 `select_category` 工具（~200 Token），让模型选择类别（read/write/search/run/plan）。
   - **阶段 2：** 仅发送该类别下的工具 Schema（平均 ~400 Token）。

**Token 节省计算：**
```javascript
// 两阶段路由的 Token 节省估算
const directTokens = Math.ceil(JSON.stringify(allTools).length / 4);      // ~2000
const selectorTokens = Math.ceil(JSON.stringify(categorySelector).length / 4); // ~50
const avgCategoryTokens = Math.ceil(directTokens / 5);                     // ~400
const twoStageTokens = selectorTokens + avgCategoryTokens;               // ~450
// 节省率：~77%（对于 ≤16K 上下文模型）
```

**加权评分系统（非 LLM，零 Token 成本）：**

| 类别 | 正向信号 | 负向信号 | 优先级 |
|------|---------|---------|--------|
| write | "fix", "add", "implement" | "explain" | 最高 |
| run | "run", "test", "execute" | — | 高 |
| code_intel | "how does X work" | — | 中 |
| search | "find", "all uses of" | — | 中 |
| plan | "remember", "compile" | — | 低 |
| read | "show", "read" | — | 低 |
| web | "search web", "fetch" | — | 低 |
| respond | "yes", "ok", "what" | 动作动词 | 最低 |

**边界情况处理：** 当用户回复 "ok" 或 "yes" 时，如果不加保护，分类器会将其归类为 `respond`（无工具），导致模型无法继续之前的任务。SmallCode 实现了**确认保护（affirmation guard）**，保留上一轮的类别。

### 4.3 治理层（`bin/governor.js`）

治理层负责**工具评分、代码验证和硬失败处理**。

**工具评分器（ToolScorer）：**
- 按 `toolName:taskType` 维度记录成功/失败
- 使用拉普拉斯平滑的置信度：`confidence = (success_count + 1) / (total_calls + 2)`
- 当置信度 < 0.35 且调用次数 ≥3 时，系统会避免使用该工具处理该任务类型
- 探索奖励：未见过组合默认置信度 0.65，鼓励尝试

**代码验证器（verifyCode）：**
```javascript
// 多语言编译检查
.py  → python -m py_compile
.js  → node --check
.ts  → npx tsc --noEmit
.go  → go build
.json → JSON.parse

// 执行检查（仅脚本文件）
有 __name__ / main() / console.log → 尝试运行
```

**硬失败分解策略（Decompose）：**
当验证失败 2 次后，不直接放弃，而是根据错误类型选择分解策略：
1. **文件过大（>80 行）**：拆分为小文件
2. **语法错误集中**：逐函数修复
3. **运行时错误**：隔离测试环境
4. **未知错误**：逐行二分定位

### 4.4 计划追踪器（`src/session/plan_tracker.js`）

**问题：** 小模型在多步任务中，到第 4 轮时往往已忘记第 3 步的目标。

**解决方案：**
1. **启发式触发：** 仅对可能的多步任务启用（关键词：refactor/migrate/rewrite，或消息 >300 字符，或 >3 个祈使句）。
2. **计划提取：** 支持 LLM 提取器和正则回退，容忍多种格式（`1. step`、`PLAN:`、`- step`）。
3. **锚点注入：** 每轮对话中注入格式化的活跃计划：
   ```
   ACTIVE PLAN (step 3 of 5):
   ✓ 1. Read the existing auth module
   ✓ 2. Identify the JWT validation function
   → 3. Add the refresh token handler
     4. Update the route middleware
     5. Run tests
   ```
4. **依赖图优化：** 分析步骤间的文件依赖关系，为并行执行奠定基础。

### 4.5 启动检测器（`src/session/bootstrap.js`）

**问题：** 小模型在新项目的前 3-5 轮往往浪费工具调用去发现项目结构（"这是什么项目？""怎么运行测试？"）。

**解决方案：** BootstrapDetector 在首次对话前扫描工作目录，生成一行项目摘要注入系统提示：

```
"Node 20 (npm) — Next.js app. Build: `npm run build`. Test: `npm test`. Entry: src/app.js"
```

**检测能力：**
- **Node.js**：package.json → 解析引擎版本、包管理器、脚本、框架、入口文件
- **Python**：pyproject.toml/setup.py → 版本、包管理器、框架（Django/FastAPI/Flask）
- **Rust**：Cargo.toml → 版本、构建命令、入口文件
- **Go**：go.mod → 版本、构建/测试命令
- **.NET / Java / Ruby**：类似检测

**框架检测：** 通过依赖名自动识别（express → Express, next → Next.js, fastapi → FastAPI, django → Django 等）。

### 4.6 读前写保护（`src/tools/read_tracker.js`）

**问题：** 小模型经常覆盖它们没有读取过的文件，尤其是当用户说"修复 bug"时，模型会猜测文件内容并写出错误的"修复"。

**机制：**
- 追踪当前会话中所有已读取的文件路径
- 第一次对未读取的现有文件执行 `write_file` 时，返回守卫错误，提示先读取
- 模型读取后，守卫解除
- 第二次尝试则允许通过（处理合法的全量替换需求）
- 可配置为严格模式（永远禁止未读先写）

**工程智慧：** 不是硬阻断，而是**教学式纠正**——强迫模型建立"先读后写"的习惯。

### 4.7 模型配置系统（`src/model/profiles.js`）

SmallCode 内置了 **8+ 主流模型的能力画像**：

| 模型系列 | 上下文 | 工具格式 | 强项 | 弱项 |
|---------|--------|---------|------|------|
| Gemma-4 | 32K | native | 代码补全、指令遵循 | 超长规划 |
| Qwen3 | 32K | hermes | 推理、代码生成 | 冗长 |
| DeepSeek-Coder | 16K | json | 代码补全、调试 | 指令遵循 |
| CodeLlama | 16K | text | 代码补全 | 工具使用 |
| Mistral-Nemo | 128K | native | 长上下文 | 代码专项 |
| StarCoder | 8K | text | 代码补全 | 指令遵循 |

**自适应能力：**
- 模糊前缀匹配（如 `huihui-gemma-4-e4b-it-abliterated` → `gemma-4` 画像）
- 端点自动检测上下文窗口（覆盖默认值）
- 根据画像调整提示策略（工具格式、思考预算、上下文限制）

### 4.8 MarrowScript 认知层

这是 SmallCode 最独特的技术之一——一个**声明式认知编译器**。

**概念：** 开发者用 MarrowScript 声明一个认知任务（如"分类任务类型"、"修复工具调用"），编译器自动生成完整的运行时代码（1400+ 行），包含：
- 内容哈希缓存 + TTL 管理
- 结构化追踪（trace_id/span_id）
- 重试机制（指数退避/固定间隔）
- 预算强制执行
- Schema 验证 + 自动修复
- Postgres 持久化支持

**示例 MarrowScript：**
```marrow
prompt classify_task_type(user_message: string) {
  model: TinyClassifier
  timeout: 3s
  cache: { key: hash(user_message), ttl: 10m }
  retry: { max_attempts: 2, backoff: fixed, interval: 100ms }
  constraints: [output in ["coding", "editing", "search", ...]]
}
```

**编译产物位置：** `src/compiled/` 目录下包含：
- `cognition/` — 认知基础设施（缓存、追踪、验证、路由、循环、预算、提示、修复）
- `features/` — 微任务特征（检查点、上下文检索、多文件编辑、策略、验证修复）
- `flows.js` — Saga 流运行时（带向后补偿的步执行）
- `logger.js` — 结构化 JSON 日志
- `metrics.js` — 指标系统（counter/histogram/gauge）

### 4.9 记忆与持久化架构

**双层记忆系统：**

```
短期记忆（对话历史）
  → 上下文压力时：语义压缩摘要（不丢弃）
  → 极端压力时：mid-turn eviction（丢弃旧结果）

长期记忆（SQLite + FTS5 全文搜索）
  ├─ memory_remember → 写入持久化存储
  ├─ memory_load    → 基于关键词重叠加载相关内容
  └─ 类型系统：decision / workflow / gotcha / convention / context
```

**Evidence Store（经验学习）：**
- 自动捕获"尝试过什么、什么有效、什么失败"
- 存储为可搜索的记忆对象
- 通过 FTS5 + 时效衰减加载到未来任务
- 模型能学习到："上次 pip install 在这个 Python 版本上失败了"、"npm test 不加 --run 会挂起"

**会话持久化：**
- 原子写入（temp file → rename）
- 时间降序 ID（字典序即时间序）
- 路径遍历防护
- 文件权限 0600

**快照与回滚：**
- 每轮 Agent  turn 前创建检查点
- 记录每次编辑前的文件内容
- 验证硬失败后自动回滚到检查点状态
- `.smallcode/snapshots/` 存储审计元数据

### 4.10 升级与降级机制（`bin/escalation.js`）

**升级（Escalation）：**
当本地模型在耗尽重试和分解策略后仍然硬失败时，SmallCode 可选项地将对话历史发送到更强的云模型：
- 优先级：Anthropic Claude → OpenAI → DeepSeek
- 自动格式转换（OpenAI 的 `tool_calls`/`tool` vs Anthropic 的 `tool_use`/`tool_result`）
- 会话上限（默认 5 次）防止成本失控
- 完全 opt-in——无 API Key 时该功能完全休眠

**降级（Graceful Degradation）示例链：**
```
budget-aware-mcp (SQLite+FTS5) 不可用
  → 回退到 bin/memory.js (JSON-based memory)

MarrowScript 编译产物不可用
  → 回退到正则解析或返回 null

Playwright 不可用
  → web_fetch 回退到简单 HTTP fetch

better-sqlite3 编译失败
  → 自动回退 JSON memory（不影响核心功能）
```

---

## 五、性能优化策略

### 5.1 Token 级优化

| 策略 | 节省/效果 |
|------|----------|
| 两阶段工具路由 | ~77% Token 节省（≤16K 模型） |
| respond 分类（不注入工具） | ~800 Token/轮 |
| 上下文预算引擎 | 强制工具结果 ≤4K 字符 |
| 语义压缩 | 历史摘要替代丢弃 |
| mid-turn eviction | 动态丢弃旧结果 |
| 思考预算硬截断 | 防止推理模型浪费 Token |

### 5.2 延迟优化

| 策略 | 机制 |
|------|------|
| MarrowScript 提示缓存 | 内容哈希命中时 0ms 延迟 |
| 工具去重 | 滑动窗口内相同只读调用短路返回 |
| 并行执行（基础） | 依赖图识别独立步骤 |
| 持久化 Shell 会话 | `cd`、环境变量跨调用持久 |

### 5.3 可靠性优化

| 策略 | 机制 |
|------|------|
| 容错工具解析 | 支持 JSON/YAML/XML/Hermes/纯文本 |
| Patch-first 编辑 | 精确替换 vs 全文件重写 |
| 语义合并回退 | Patch 失败时请求模型合并 |
| 早期停止检测 | 重复循环/补丁螺旋/问候回归 |
| 工具评分学习 | 避免已知低置信度组合 |
| 代码验证 | 编译 + 运行时双重检查 |

---

## 六、安全设计

### 6.1 输入安全

- **SSRF 防护：** `src/compiled/providers/ssrf_guard.js` 阻止内网请求
- **Shell 注入防护：** `execFileSync` 使用参数数组而非字符串拼接
- **路径遍历防护：** 会话 ID 和文件路径规范化检查
- **cwd 限制：** Shell 会话拒绝离开项目根目录的 `cd`

### 6.2 运行安全

- **权限最小化：** 会话文件 0600
- **原子写入：** 防止数据损坏
- **边界检查：** 追踪文件数量上限（50 个），防止无界增长
- **超时控制：** 编译检查 15s，运行检查 10s

### 6.3 隐私设计

- 默认完全离线运行（本地模型）
- 云升级完全 opt-in
- Web 浏览默认关闭（需显式启用）

---

## 七、工程实践与代码质量

### 7.1 技术栈

| 层级 | 技术 |
|------|------|
| 运行时 | Node.js 18+ |
| 核心语言 | JavaScript（运行时）+ TypeScript（编译层/扩展） |
| 依赖 | chalk, cli-highlight, marked, express（极精简） |
| 可选依赖 | budget-aware-mcp, playwright-extra, puppeteer-extra-plugin-stealth |
| 存储 | SQLite (better-sqlite3) / JSON fallback |
| 构建 | 自定义 build.js + 预编译 tarball |
| 配置 | TOML (smallcode.toml) + .env |

### 7.2 代码组织

- **约 60 个源文件**，结构清晰按职责分层
- `bin/` 与 `src/` 分离：bin 为 CLI/运行时，src 为可复用库
- `src/compiled/` 明确标记为自动生成，不手动编辑
- 每个文件顶部有详细注释说明模块职责

### 7.3 测试与基准

- **基准测试框架**（`bench/harness.js`）：smoke / polyglot / tool-use 套件
- **提示评估**（`bin/eval_runner.js`）：分类准确率、工具选择准确率
- **追踪转测试**：执行追踪可自动生成回归测试

### 7.4 配置体系

```
配置优先级（高 → 低）：
命令行参数 (-m, --endpoint)
  → 环境变量 (.env 文件，多位置搜索)
  → smallcode.toml
  → 模型画像默认值
  → 硬编码默认
```

---

## 八、架构评价与总结

### 8.1 设计亮点

1. **极致的约束意识：** 每一个功能都回答了"小模型在这个场景下会怎么失败？"的问题。这是架构设计的典范。
2. **两阶段工具路由：** 将 Token 经济学从理论变为实践，是上下文受限场景下的开创性设计。
3. **教学式错误纠正：** 读前写保护、计划追踪器等不是硬性限制，而是**训练模型养成正确习惯的机制**。
4. **观测性内建：** Token 监控、追踪、预算可视化、基准测试——全部一等公民。
5. **优雅降级的教科书：** 从 SQLite 到 JSON、从编译模块到正则、从云升级到静默跳过，降级链设计完善。

### 8.2 潜在挑战

1. **JavaScript 单线程：** 重 CPU 任务（如代码图分析）可能阻塞 Agent Loop。
2. **MarrowScript 学习曲线：** 自定义 DSL 增加了贡献门槛。
3. **模型画像维护：** 新模型发布需要持续更新 `profiles.js`。
4. **正则分类器的上限：** 确定性分类器在边缘场景下可能误判，未来可能需要轻量级模型分类器。

### 8.3 适用场景

| 场景 | 推荐度 |
|------|--------|
| 消费级硬件上的本地编码辅助 | ⭐⭐⭐⭐⭐ |
| 隐私敏感环境的离线开发 | ⭐⭐⭐⭐⭐ |
| 小模型（8B–35B）的代码任务 | ⭐⭐⭐⭐⭐ |
| 前沿大模型（Claude/GPT-5） | ⭐⭐（可用但非最优） |
| 超大规模代码库重构 | ⭐⭐⭐（依赖图优化尚未完全激活） |

### 8.4 总体评价

SmallCode 是一个**架构成熟度远超其 Star 数**的项目。它证明了在特定约束下，通过系统性的工程补偿，小模型也能完成可靠的编程辅助任务。其核心设计理念——"不为完美模型设计，而为真实约束补偿"——不仅适用于 AI 编程代理，也为所有资源受限场景下的 AI 系统设计提供了范本。

**架构评分：9.0/10**（扣分点在单线程运行时和边缘场景分类准确率）

---

## 附录 A：关键文件清单

| 文件 | 行数（估算） | 职责 |
|------|-------------|------|
| `bin/smallcode.js` | ~1570 | Agent Loop 入口 |
| `bin/tools.js` | ~200 | 工具 Schema 定义 + 路由 |
| `bin/executor.js` | ~400 | 18 工具执行器 |
| `bin/governor.js` | ~300 | 工具评分 + 验证 + 分解 |
| `bin/model_client.js` | ~400 | LLM API 调用 + 流式处理 |
| `bin/escalation.js` | ~200 | 云模型升级引擎 |
| `src/tools/two_stage_router.js` | ~100 | 两阶段路由决策 |
| `src/session/plan_tracker.js` | ~200 | 计划追踪与锚点注入 |
| `src/session/bootstrap.js` | ~250 | 项目启动检测 |
| `src/tools/read_tracker.js` | ~150 | 读前写保护 |
| `src/model/profiles.js` | ~100 | 模型能力画像 |
| `src/governor/early_stop.js` | ~150 | 早期停止检测 |
| `src/compiled/cognition/*.js` | ~800 | 编译认知基础设施 |
| `src/compiled/features/*.js` | ~400 | 编译微任务特征 |

## 附录 B：依赖图谱（核心）

```
smallcode.js (入口)
  ├─ tui.js / fullscreen.js
  ├─ config.js
  ├─ tools.js
  │   └─ two_stage_router.js
  ├─ executor.js
  ├─ governor.js
  ├─ model_client.js
  ├─ escalation.js
  ├─ features_adapter.js
  ├─ mcp_bridge.js
  ├─ memory.js / budget-aware-mcp
  ├─ token_monitor.js
  ├─ trace_recorder.js
  ├─ eval_runner.js
  └─ src/* (各子模块)
       ├─ model/* (profiles, router, thinking_budget)
       ├─ session/* (plan_tracker, bootstrap, persistence, snapshot, undo)
       ├─ tools/* (read_tracker, shell_session, dedup, mcp_client)
       ├─ governor/early_stop.js
       ├─ plugins/* (loader, skills)
       ├─ memory/evidence.js
       └─ compiled/* (cognition, features, flows, logger, metrics)
```

---

*本报告由 OpenClaw Daily Code Analysis 自动生成。分析基于源码静态阅读与架构文档解读，未包含运行时行为分析。*
