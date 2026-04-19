# 技术架构与源码研读报告：Hermes Agent

**项目名称：** NousResearch/hermes-agent  
**仓库地址：** https://github.com/NousResearch/hermes-agent  
**Stars：** 65,964（+32,572 本周）| **Forks：** 预估 8,000+  
**技术标签：** Self-Improving AI Agent · DSPy · GEPA · Multi-Platform Gateway · SQLite/FTS5  
**报告日期：** 2026-04-17  
**分析深度：** 架构设计 · 核心模块 · 关键设计决策 · 源码树状结构

---

## 一、项目概述与核心定位

Hermes Agent 是 **Nous Research** 开发的一款**自进化 AI Agent 框架**，核心理念是"越用越聪明"——它不只是执行任务，而是从每次对话中提取技能（Skills）、自动改进提示词、持久化用户记忆，并在跨会话中积累对用户的认知模型。

### 1.1 为什么选择 Hermes？

| 维度 | 传统 Agent | Hermes Agent |
|------|------------|--------------|
| 技能管理 | 手动维护，静态配置 | 从经验中自动创建并持续改进 |
| 记忆 | 仅限当前会话 | FTS5 跨会话搜索 + Honcho 用户建模 |
| 部署 | 强依赖本地环境 | 6 种后端：本地/Docker/SSH/Daytona/Singularity/Modal |
| 模型选择 | 锁定单一 provider | Nous Portal / OpenRouter（200+）/ OpenAI / HuggingFace 等 |
| 多平台 | 各平台独立开发 | 单一 gateway 进程支持 Telegram/Discord/Slack/WhatsApp/Signal 等 |
| 自我进化 | 无 | DSPy + GEPA（ICLR 2026 Oral）闭环自优化 |

### 1.2 版本里程碑

| 版本 | 核心特性 |
|------|----------|
| v0.2–0.4 | 基础 Agent 框架、工具集、CLI |
| v0.5 | Gateway 多平台支持（首个 messaging 版本） |
| v0.6 | Browser Use 集成、远程后端支持 |
| v0.7 | MCP 工具支持（~1050 行 mcp_tool.py） |
| v0.8 | **Browser Use 集成**、remote backend、工作树并行（209 PRs，本周爆发主因） |
| v0.9 | 最新稳定版 |

---

## 二、整体架构

### 2.1 Monorepo 组织结构

```
hermes-agent/
├── run_agent.py              # AIAgent 核心对话循环（11,538 行，单文件之王）
├── model_tools.py            # 工具编排层（562 行）
├── toolsets.py               # 工具集定义（_HERMES_CORE_TOOLS）
├── tools/                    # 工具实现（每个工具独立文件 + registry）
│   ├── registry.py           # 中心注册表（自动发现机制）
│   ├── delegate_tool.py      # 子 Agent 委托（1,143 行）
│   ├── mcp_tool.py           # MCP 客户端（~1050 行）
│   ├── browser_tool.py       # 浏览器自动化
│   ├── terminal_tool.py      # 终端编排
│   ├── file_tools.py         # 文件读写搜索
│   ├── web_tools.py          # Web 搜索/提取
│   └── environments/         # 终端后端（local/docker/ssh/modal/daytona/singularity）
├── agent/                    # Agent 内部模块
│   ├── prompt_builder.py     # 系统提示词构建（~1000+ 行）
│   ├── context_compressor.py # 上下文压缩
│   ├── prompt_caching.py     # Anthropic prompt caching
│   ├── memory_manager.py     # 记忆上下文构建
│   ├── display.py            # KawaiiSpinner 动画UI
│   ├── models_dev.py         # models.dev registry 集成
│   ├── skill_commands.py     # Skill slash 命令
│   └── trajectory.py          # 轨迹保存
├── hermes_cli/              # CLI 子系统
│   ├── main.py               # 入口点
│   ├── commands.py           # 中心化命令注册表（CommandDef）
│   ├── config.py             # 配置管理 + 迁移
│   ├── setup.py              # 交互式设置向导
│   ├── skin_engine.py        # 皮肤/主题引擎
│   ├── skills_config.py      # 平台级 skill 配置
│   └── model_switch.py       # 模型切换管线
├── gateway/                  # 消息网关
│   ├── run.py                # 主循环 + slash 命令 + 消息分发
│   ├── session.py            # SessionStore 持久化
│   └── platforms/            # 平台适配器
│       ├── telegram.py       # 飞书、微信、WhatsApp、Discord...
│       ├── discord.py
│       ├── slack.py
│       ├── signal.py
│       ├── feishu.py         # 飞书
│       └── wechat/           # 企业微信等
├── cron/                     # 定时任务调度器
├── environments/             # RL 训练环境（Atropos）
├── skills/                   # 内置 Skill 包（按领域分类）
├── optional-skills/         # 可选 Skill（autonomous-ai-agents/devops/research...）
├── batch_runner.py           # 并行批处理
├── rl_cli.py                 # 强化学习 CLI
├── hermes_state.py          # SQLite 会话存储 + FTS5 搜索（1,238 行）
└── tests/                    # ~3000 测试用例
```

### 2.2 核心技术栈

```
语言：Python >= 3.11（纯 Python，无 C 扩展依赖）

模型交互：
  openai>=2.21.0,<3        # 统一 OpenAI 兼容接口
  anthropic>=0.39.0,<1    # Anthropic API（含 prompt caching）

CLI/UI：
  rich>=14.3.3,<15         # 终端富文本输出
  prompt_toolkit>=3.0.52   # 带自动补全的交互输入

工具生态：
  exa-py>=2.9.0            # Web 搜索
  firecrawl-py>=4.16.0     # 网页抓取
  fal-client>=0.13.1       # 图像生成
  edge-tts>=7.2.7          # 免 API key TTS

数据：
  httpx[socks]             # 异步 HTTP（SOCKS 代理支持）
  python-dotenv            # .env 管理

消息平台：
  python-telegram-bot>=22.6 # Telegram Bot
  discord.py>=2.7.1        # Discord
  slack-bolt>=1.18.0       # Slack

可选：
  faster-whisper>=1.0.0   # 本地语音识别
  modal/daytona            # 无服务器计算
  honcho-ai>=2.0.1         # 用户记忆建模
```

---

## 三、核心模块详解

### 3.1 AIAgent 核心对话循环（run_agent.py, 11,538 行）

`run_agent.py` 是 Hermes 的"心脏"，包含了完整的 Agent 运行时。

#### 对话循环结构

```python
class AIAgent:
    def __init__(self,
        model: str = "anthropic/claude-opus-4.6",
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        platform: str = None,   # "cli", "telegram", etc.
        session_id: str = None,
        save_trajectories: bool = False,
        # ... 更多参数
    ): ...

    def run_conversation(self, user_message: str, ...) -> dict:
        """完整接口 — 返回 {final_response, messages, usage}"""
```

**核心循环（纯同步）：**
```python
while api_call_count < self.max_iterations and self.iteration_budget.remaining > 0:
    response = client.chat.completions.create(
        model=model, messages=messages, tools=tool_schemas
    )
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content  # 最终回复，退出循环
```

**消息格式：** OpenAI 兼容格式 `{"role": "system/user/assistant/tool", ...}`，reasoning content 存在 `assistant_msg["reasoning"]` 中。

#### 导入依赖链

```
run_agent.py
  ├── model_tools.py         # get_tool_definitions, handle_function_call
  │     └── tools/registry.py  # discover_builtin_tools（自动发现所有工具）
  ├── tools/terminal_tool.py  # 终端管理
  ├── agent/prompt_builder.py  # 系统提示词构建
  ├── agent/context_compressor.py # 自动上下文压缩
  ├── agent/memory_manager.py  # 记忆上下文
  ├── agent/display.py        # KawaiiSpinner UI
  └── hermes_state.py         # SQLite 会话持久化
```

---

### 3.2 工具系统

#### 自动发现机制（tools/registry.py）

每个工具文件通过 `registry.register()` 在 import 时自动注册：

```python
# tools/web_search.py
def check_requirements() -> bool:
    return bool(os.getenv("EXA_API_KEY"))

def web_search(query: str, task_id: str = None) -> str:
    return json.dumps({"results": [...]})

registry.register(
    name="web_search",
    toolset="web",
    schema={"name": "web_search", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: web_search(query=args["query"], task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXA_API_KEY"],
)
```

**自动发现：** 任何 `tools/*.py` 文件中包含 `registry.register()` 调用都会在 `discover_builtin_tools()` 时被自动导入，无需维护导入列表。

#### 中心工具注册表（model_tools.py）

```python
# 公开 API
get_tool_definitions(enabled_toolsets, disabled_toolsets, quiet_mode) -> list
handle_function_call(function_name, function_args, task_id, user_task) -> str
TOOL_TO_TOOLSET_MAP: dict   # 工具→工具集映射
TOOLSET_REQUIREMENTS: dict  # 工具集→环境要求
get_all_tool_names() -> list
check_toolset_requirements() -> dict
```

#### 异步桥接

工具处理器既有同步也有异步。`model_tools.py` 维护一个**持久化事件循环**（`threading.local()` 存储），避免 `asyncio.run()` 反复创建/销毁循环导致的 "Event loop is closed" 错误：

```python
# 单例循环（主线程）
_tool_loop = None
_tool_loop_lock = threading.Lock()

# 每线程独立循环（Worker 线程）
_worker_thread_local = threading.local()
```

#### 核心工具集（_HERMES_CORE_TOOLS）

```python
_HERMES_CORE_TOOLS = [
    # Web
    "web_search", "web_extract",
    # Terminal + process
    "terminal", "process",
    # File manipulation
    "read_file", "write_file", "patch", "search_files",
    # Vision + image
    "vision_analyze", "image_generate",
    # Skills
    "skills_list", "skill_view", "skill_manage",
    # Browser automation
    "browser_navigate", "browser_snapshot", "browser_click",
    "browser_type", "browser_scroll", "browser_back",
    "browser_press", "browser_get_images", "browser_vision",
    # TTS
    "text_to_speech",
    # Planning & memory
    "todo", "memory",
    # Session history search
    "session_search",
    # Clarifying questions
    "clarify",
    # Code execution + delegation
    "execute_code", "delegate_task",
    # Cron
    "cronjob",
    # Cross-platform messaging
    "send_message",
]
```

---

### 3.3 子 Agent 委托系统（delegate_tool.py, 1,143 行）

Hermes 支持**并行多子 Agent** 工作，通过 Worktree 隔离实现。

#### 关键设计

```python
# 子 Agent 禁止使用的工具（安全隔离）
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # 禁止递归委托
    "clarify",         # 禁止用户交互
    "memory",          # 禁止写共享记忆
    "send_message",    # 禁止跨平台副作用
    "execute_code",    # 应逐步推理而非写脚本
])

MAX_DEPTH = 2  # 深度限制：parent -> child -> grandchild（拒绝）
_DEFAULT_MAX_CONCURRENT_CHILDREN = 3
```

**每个子 Agent 获得：**
- 全新对话（无父 Agent 历史）
- 独立 task_id（独立终端会话、文件缓存）
- 受限工具集（可配置，blocked 工具始终剥离）
- 聚焦型系统提示词（来自委托目标 + 上下文）

**进程间隔离：** `_last_resolved_tool_names` 是 `model_tools.py` 中的进程级全局变量，`delegate_tool.py` 的 `_run_single_child()` 会保存/恢复该全局变量，防止子 Agent 影响父 Agent 的工具名解析。

---

### 3.4 会话状态存储（hermes_state.py, 1,238 行）

#### SQLite + FTS5 架构

```python
# WAL 模式：并发读 + 单写（gateway 多平台场景）
_conn.execute("PRAGMA journal_mode=WAL")

# FTS5 全文搜索虚拟表
CREATE VIRTUAL TABLE messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=id
)
```

#### 写冲突处理（反 Convoy 策略）

传统 SQLite 在高并发下会产生 **convoy effect**（写锁等待队列）。Hermes 的解法：

```python
# SQLite 超时 1s（而非默认 30s）
timeout=1.0

# 应用层随机 jitter 重试
jitter = random.uniform(0.020, 0.150)  # 20-150ms 随机退避
# 替代 SQLite 内置确定性退避，避免所有写者同步等待

_WRITE_MAX_RETRIES = 15
```

#### 消息 Schema

```sql
CREATE TABLE messages (
    id INTEGER PRIMARY KEY,
    session_id TEXT,
    role TEXT,
    content TEXT,
    tool_call_id TEXT,
    tool_calls TEXT,         -- JSON 序列化
    tool_name TEXT,
    timestamp REAL,
    token_count INTEGER,
    finish_reason TEXT,
    reasoning TEXT,           -- v6 新增：保留推理链
    reasoning_details TEXT,   -- v6 新增
    codex_reasoning_items TEXT -- v6 新增
);
```

#### 搜索安全（FTS5 Query Sanitization）

FTS5 有自己的查询语法（`AND/OR/NOT/+/*/{}/"` 等），直接拼接用户输入会引发 `sqlite3.OperationalError`。Hermes 实现了六步净化：

1. 保护成对引号短语
2. 剥离 FTS5 特殊字符
3. 处理 `*` 通配符
4. 去除悬空布尔运算符
5. 将点/连字符词用双引号包裹（防止 `chat-send` → `chat AND send`）
6. 还原保护短语

---

### 3.5 消息网关（gateway/）

Gateway 是 Hermes 的**多平台消息接入层**，单一进程支持所有消息平台。

#### 架构

```
gateway/run.py          # 主循环 + slash 命令分发 + 消息路由
gateway/session.py       # SessionStore — 对话持久化
gateway/platforms/       # 平台适配器
  ├── telegram.py        # Telegram Bot
  ├── discord.py         # Discord
  ├── slack.py           # Slack
  ├── feishu.py          # 飞书
  ├── whatsapp.py         # WhatsApp
  ├── signal.py          # Signal
  ├── wecom.py           # 企业微信
  ├── weixin.py          # 微信
  └── ...
```

#### Slash 命令注册（hermes_cli/commands.py）

所有 slash 命令集中在 `COMMAND_REGISTRY` 的 `CommandDef` 对象中，下游消费者自动派生：

| 消费者 | 用途 |
|--------|------|
| CLI | `process_command()` 解析别名并分发 |
| Gateway | `GATEWAY_KNOWN_COMMANDS` hook 发射 + 分发 |
| Telegram | `telegram_bot_commands()` 生成 BotCommand 菜单 |
| Slack | `slack_subcommand_map()` 生成 `/hermes` 子命令 |
| Autocomplete | `SlashCommandCompleter` 提供补全 |

**添加新命令只需修改一处：** `COMMAND_REGISTRY` → 所有消费者自动更新。

---

### 3.6 皮肤/主题引擎（hermes_cli/skin_engine.py）

Hermes 支持数据驱动的 CLI 视觉定制，无需修改代码：

```yaml
# ~/.hermes/skins/cyberpunk.yaml
colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
spinner:
  thinking_verbs: ["jacking in", "decrypting", "uploading"]
  wings: [["⟨⚡", "⚡⟩"]]
branding:
  agent_name: "Cyber Agent"
  response_label: " ⚡ Cyber "
```

内置皮肤：`default`（经典金/萌系）、`ares`（战神红铜）、`mono`（灰度）、`slate`（冷蓝开发者）。

---

### 3.7 多实例 Profiles 支持

Hermes 支持 **Profiles**——完全隔离的多实例，每个实例有独立的 `HERMES_HOME` 目录：

```python
# 核心机制：_apply_profile_override() 设置 HERMES_HOME 环境变量
# 所有 119+ 处 get_hermes_home() 调用自动作用域隔离
hermes -p coder profile list   # 列出所有 profiles
```

**Profile 安全规范：**
- ✅ 使用 `get_hermes_home()` 获取路径
- ✅ 使用 `display_hermes_home()` 显示给用户
- ❌ 禁止硬编码 `~/.hermes`（破坏 profiles 隔离）
- ❌ 禁止在 tmux/iTerm2 中使用 `simple_term_menu`（渲染 bug）

---

## 四、自我进化机制（DSPy + GEPA）

Hermes 的"越用越聪明"并非营销噱头，而是有具体的算法支撑：

### 4.1 GEPA（Genetic Evolution Prompt Architecture）

**ICLR 2026 Oral** 论文提出的框架，核心思想：

1. **轨迹收集**：Agent 执行任务时的 tool calling 序列作为进化原材料
2. **种群进化**：维护候选提示词种群，通过交叉变异生成新提示词
3. **适应度评估**：在下游任务上评估提示词质量
4. **闭环**：优秀提示词进入下一轮进化

Hermes 的 `batch_runner.py` + `trajectory.py` 支撑了这一过程。

### 4.2 Honcho 用户建模

集成 `honcho-ai`（Plastic Labs）进行** dialectic 用户建模**，构建跨会话用户画像。

### 4.3 技能自改进

每次复杂任务完成后，Agent 会：
1. 从对话中提取可复用的技能模式
2. 存储到 `~/.hermes/skills/`
3. 在后续使用中持续微调改进

---

## 五、关键设计决策

### 5.1 为什么不使用 LangChain？

| 维度 | LangChain | Hermes |
|------|-----------|--------|
| 抽象层级 | 极高（很多"聪明"的设计反而束缚） | 低抽象——直接调用 API |
| Provider 锁定 | 是 | 否（OpenAI 兼容接口 + 适配器） |
| 工具注册 | LangChain Tools | 自研 registry（无反射开销） |
| 上下文管理 | LCEL 表达式 | 直接操作 messages 列表 |
| 调试体验 | 黑箱 trace | 完全透明，源码即文档 |

### 5.2 同步 Agent 循环

Agent 循环保持**纯同步**（而非 async），因为：
- 工具调用（文件系统/网络）本身有阻塞 I/O
- Python GIL 限制真正的并行 CPU 操作
- 异步不会带来显著收益，但增加复杂度

### 5.3 进程全局状态 vs 线程局部存储

```python
# _last_resolved_tool_names — 进程级全局量（model_tools.py）
# delegate_tool 会保存/恢复它，防止子 Agent 污染父 Agent

# _tool_loop — 主线程持久化事件循环（避免 Event loop is closed）
# _worker_thread_local — 每线程独立循环（子 Agent 用）
```

### 5.4 WAL + 应用层重试 vs 默认 SQLite 行为

高并发写入场景（gateway 多平台 + CLI 并发），默认 SQLite 的 30s 超时 + 确定性退避会导致 **convoy effect**（写锁队列堵塞）。Hermes 选择 1s 超时 + 随机 20-150ms 重试，将 convoy 转化为随机碰撞。

---

## 六、源码树状结构

```
hermes-agent/
├── run_agent.py                    ← 核心 AIAgent（11,538 行）
│   └── 导入链：registry → tools → model_tools → run_agent
├── model_tools.py                  ← 562 行，工具编排 + 异步桥接
├── toolsets.py                     ← 工具集定义（_HERMES_CORE_TOOLS）
├── tools/
│   ├── registry.py                 ← 中心注册表，自动发现
│   ├── delegate_tool.py            ← 1,143 行，子 Agent 委托
│   ├── mcp_tool.py                 ← ~1,050 行，MCP 客户端
│   ├── browser_tool.py             ← BrowserBase 浏览器自动化
│   ├── terminal_tool.py            ← 终端编排 + environments/
│   ├── file_tools.py               ← 文件操作
│   ├── web_tools.py                ← Web 搜索 + Firecrawl 提取
│   ├── code_execution_tool.py      ← 代码沙箱执行
│   ├── memory_tool.py              ← 记忆读写
│   ├── cronjob_tools.py            ← 定时任务
│   └── environments/                ← 后端适配器
│       ├── local.py
│       ├── docker.py
│       ├── ssh.py
│       ├── modal.py
│       ├── daytona.py
│       └── singularity.py
├── agent/
│   ├── prompt_builder.py            ← 系统提示词构建
│   ├── context_compressor.py        ← 自动上下文压缩
│   ├── prompt_caching.py           ← Anthropic 缓存标记
│   ├── memory_manager.py           ← 记忆上下文块构建
│   ├── display.py                  ← KawaiiSpinner 动画
│   ├── model_metadata.py            ← 模型元数据 + 上下文长度
│   ├── models_dev.py                ← models.dev registry
│   ├── skill_commands.py            ← Skill slash 命令
│   ├── trajectory.py               ← 轨迹保存
│   ├── retry_utils.py              ← 指数退避重试
│   └── usage_pricing.py            ← 成本估算
├── hermes_cli/
│   ├── main.py                      ← 入口（hermes subcommands）
│   ├── commands.py                  ← CommandDef 中心注册表
│   ├── config.py                    ← DEFAULT_CONFIG + 迁移
│   ├── setup.py                     ← 交互式向导
│   ├── skin_engine.py               ← 皮肤引擎
│   ├── model_switch.py              ← 模型切换管线
│   ├── skills_config.py             ← per-platform skill 配置
│   └── auth.py                      ← Provider 凭证解析
├── gateway/
│   ├── run.py                       ← 主循环 + 消息分发
│   ├── session.py                   ← SessionStore
│   └── platforms/
│       ├── telegram.py
│       ├── discord.py
│       ├── slack.py
│       ├── feishu.py
│       ├── whatsapp.py
│       ├── signal.py
│       ├── wecom.py
│       ├── weixin.py
│       └── ...
├── hermes_state.py                  ← 1,238 行，SQLite + FTS5
├── cron/
│   ├── scheduler.py
│   └── jobs.py
├── skills/                          ← 内置 Skill 包
├── optional-skills/                 ← 可选 Skill（按领域）
│   ├── autonomous-ai-agents/
│   ├── devops/
│   ├── research/
│   ├── mlops/
│   └── ...
├── environments/                     ← RL 训练环境（Atropos）
├── tests/                           ← ~3,000 测试用例
├── batch_runner.py                  ← 并行批处理
└── rl_cli.py                        ← 强化学习 CLI
```

---

## 七、工程亮点与值得学习的实践

### ✅ 优点

1. **自注册工具系统**：零配置添加新工具，无需维护导入列表
2. **进程全局状态的安全管理**：明确的保存/恢复模式处理全局量
3. **WAL + 应用层 jitter 重试**：优雅处理高并发写冲突
4. **FTS5 查询净化**：六步防御 FTS5 注入/语法错误
5. **CommandDef 单点定义**：所有消费者自动派生，添加命令只改一处
6. **持久化事件循环**：避免 asyncio.run() 的循环关闭问题
7. **Profile 隔离**：真正的多租户架构
8. **深度 11,538 行单文件**：高度内聚，所有 Agent 逻辑一目了然

### ⚠️ 潜在风险

1. **11,538 行单文件**：`run_agent.py` 过于庞大，局部修改风险较高
2. **进程全局变量**（`_last_resolved_tool_names`）：虽已文档化，但仍是隐式状态
3. **Profile 路径硬编码规范**：依赖开发者遵守 `get_hermes_home()` 规范，编译器无法强制
4. **多平台 token 锁**：Gateway adapter 需要 `acquire_scoped_lock()` 防止跨 Profile 凭证冲突

---

## 八、与 OpenClaw 的关系

Hermes Agent 与 OpenClaw 有**直接的互操作通道**：

- **OpenClaw 迁移脚本**：`hermes claw migrate` 命令将 OpenClaw 配置导入 Hermes
- **ACP 适配器**：`acp_adapter/` 目录支持 VS Code / Zed / JetBrains 集成
- **技能格式兼容**：Skills 系统与 OpenClaw Skill 格式高度兼容（`agent/skill_commands.py` 扫描 `~/.hermes/skills/`）
- **平台层共用**：飞书、企业微信等消息平台适配器在两者间可复用

---

*本报告基于 v0.9.0 源码分析生成，分析深度：架构设计 · 核心模块 · 关键设计决策 · 源码树状结构*
