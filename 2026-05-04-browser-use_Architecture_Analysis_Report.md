# browser-use 技术架构与源码研读报告

> 分析版本：v0.12.6 | 分析日期：2026-05-04 | 语言：Python | 许可证：MIT

## 一、项目概述

**browser-use** 是当前最热门的开源 AI 浏览器自动化框架之一，GitHub Stars 持续增长，专注于让 AI Agent 能够控制浏览器完成复杂网页任务。

### 核心定位

- **专注领域**：AI Agent 驱动的浏览器自动化
- **差异化**：不追求通用 LLM 框架，而是专注于"AI Agent 控制浏览器完成真实网页任务"这一垂直场景
- **生态**：提供开源 Agent 库 + 商业 Cloud 版本（Browser Use Cloud），形成开源免费 + 商业增值的双层模式
- **Benchmark**：开源 benchmark 项目 `browser-use/benchmark`，在 100 个真实浏览器任务上评测各模型表现

### 技术亮点

1. **DOM 序列化引擎**：自研 EnhancedSnapshot，把 Chromium 渲染树序列化成语义化可点击元素图，LLM 只需要理解"元素#3"而不是原始 HTML
2. **动态 Action Model**：Agent 根据当前页面 URL 和上下文动态生成可用 Actions，页面专属操作自动注入
3. **Loop Detection**：基于滚动窗口的行为重复检测 + 页面停滞检测，双重防止 Agent 陷入死循环
4. **Message Compaction**：历史消息压缩合并，控制长程任务的 prompt token 增长
5. **Planning 系统**：内置计划跟踪、重排触发、探索限制，实现有状态的"任务规划 + 执行反馈"循环

---

## 二、系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         Agent (service.py)                       │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ MessageManager│ │  LoopDetector │ │  PlanTracker  │            │
│  └──────────┘  └──────────────┘  └──────────────┘               │
│         │                    │                  │                │
│  ┌──────▼──────┐     ┌────────▼────┐    ┌──────▼──────┐        │
│  │ LLM Adapter  │     │ ActionModel │    │  HistoryList │        │
│  │ (BaseChatModel)│   │ (动态生成)   │    │              │        │
│  └──────┬──────┘     └──────┬──────┘    └──────┬──────┘        │
│         │                   │                  │                │
│         └───────────────────┼──────────────────┘                │
│                    ┌─────────▼─────────┐                       │
│                    │   Tools Registry   │                        │
│                    │  (action 注册表)   │                        │
│                    └─────────┬─────────┘                       │
│                              │                                  │
│         ┌────────────────────┼────────────────────┐           │
│  ┌──────▼──────┐  ┌──────────▼────┐  ┌──────────▼──────┐      │
│  │ BrowserSession│ │  DomService   │  │  TelemetryService │      │
│  │  (Playwright) │ │ (CDP 协议)    │  │   (产品遥测)     │      │
│  └──────┬──────┘  └──────┬───────┘  └──────────────────┘      │
│         │                │                                      │
│  ┌──────▼────────────────▼──────────────────────────────┐      │
│  │              Browser (Chromium 实例)                  │      │
│  │  CDP WebSocket ← CDP Protocol ← DOM/Accessibility API │      │
│  └──────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心模块职责

| 模块 | 文件 | 职责 |
|------|------|------|
| **Agent** | `agent/service.py` (4131行) | 核心 Agent 逻辑，step 循环，状态管理，LLM 调用 |
| **BrowserSession** | `browser/session.py` (4000行) | Playwright/CDP 生命周期管理，多 Tab，下载追踪 |
| **DomService** | `dom/service.py` (1165行) | Chromium DOM 树抓取，iframe 嵌套处理，元素可视化 |
| **Tools** | `tools/service.py` (2252行) | Action 执行引擎，CDP 事件分发，超时保护 |
| **MessageManager** | `agent/message_manager/` | Prompt 组装，状态消息生成，历史压缩 |
| **LLM Adapters** | `llm/` | OpenAI/Anthropic/Google/BrowserUse 等多 Provider 统一接口 |

---

## 三、核心机制深度解析

### 3.1 Agent Step 循环

`Agent.run()` 是主入口，内部调用 `step()` 执行每个回合。核心流程分三个阶段：

```
step() {
  ├── Phase 0: CaptchaWait (等待验证码解决)
  ├── Phase 1: _prepare_context()
  │   ├── get_browser_state_summary() → screenshot + DOM
  │   ├── _update_action_models_for_page() → 页面专属 Actions
  │   ├── create_state_messages() → 组装 prompt
  │   ├── inject_loop_nudge / budget_warning / replan_nudge
  │   └── maybe_compact_messages()
  ├── Phase 2: _get_next_action() + _execute_actions()
  │   ├── get_model_output() → LLM.ainvoke()
  │   ├── multi_act() → 执行每个 action
  │   └── loop_detector.record_action()
  └── Phase 3: _post_process() + _finalize()
      ├── save history item
      ├── emit CloudEvents
      └── n_steps++
}
```

**关键设计**：step 完成后才递增 `n_steps`，即使超时也保证计数器推进，防止死锁。

### 3.2 DOM 序列化系统

`DomService` 是 browser-use 最核心的技术壁垒之一。它通过 CDP (Chrome DevTools Protocol) 访问渲染树：

```
CDP Accessibility API
        ↓
EnhancedSnapshot (dom/enhanced_snapshot.py)
        ↓
DOMTreeSerializer → 选择性元素检测
        ↓
ClickableElementDetector (interactive elements only)
        ↓
SerializedDOMState → selector_map + accessible_tree
```

**关键特性**：
- **Viewport Threshold**：只序列化视口内（viewport_height 内）的元素，隐藏元素单独追踪滚动距离
- **Cross-origin Iframe 处理**：`max_iframe_depth=5`，跨域 iframe 无法通过 CDP 访问时用占位符标记
- **Paint Order Filtering**：按 paint 顺序过滤元素，解决 CSS z-index 覆盖问题
- **Shadow DOM 支持**：递归进入 shadow roots

### 3.3 动态 Action Model

`Agent._setup_action_models()` 根据当前页面上下文动态生成 `ActionModel`：

```python
# 初始状态：全局 Actions（click/input/scroll/navigate等）
self.ActionModel = self.tools.registry.create_action_model()

# 页面更新时：根据 URL 过滤 Actions
page_filtered_actions = self.tools.registry.get_prompt_description(url)
# → "On GitHub, you can also use: github_search_code, github_view_file..."
```

`Tools` 负责注册 actions，每个 action 都是一个 Python 函数 + Pydantic `param_model`：

```python
@tools.action(description="Click an interactive element by index")
async def click(params: ClickElementAction) -> ActionResult:
    ...
```

### 3.4 Loop Detection 系统

`ActionLoopDetector`（在 `agent/views.py` 中）实现了双重检测：

**Action 重复检测**：
- 对 actions 做语义归一化（搜索 query 去重排序、点击按 index 而非坐标）
- 滚动窗口（window_size=20）内统计最高重复次数
- 5 次重复 → 注入引导消息；8 次 → 升级提示；12 次 → 强制警告

**页面停滞检测**：
- 用 `PageFingerprint`（url + element_count + DOM text SHA256）追踪页面状态
- 连续 5 次相同 fingerprint → 注入"操作可能未生效"提示

**重要**：Loop Detection 只注入上下文消息，**不阻止 Agent 行动**，给了 Agent 自主决策权。

### 3.5 Message Compaction

`MessageCompactionSettings` 控制历史消息压缩：

- 每 25 步触发一次（`compact_every_n_steps=25`）
- 保留最近 6 条（`keep_last_items=6`），其余压缩为摘要
- 摘要使用 `page_extraction_llm`（或主 LLM）生成，最大 6000 chars
- 压缩触发阈值：`trigger_char_count=40000`（约 10k tokens）

### 3.6 Planning 系统

`AgentState` 中维护了 `plan: list[PlanItem]` 和 `current_plan_item_index`：

- **重排触发**（`planning_replan_on_stall=3`）：连续 3 次失败后注入 REPLAN 消息
- **探索限制**（`planning_exploration_limit=5`）：无计划走 5 步后注入 PLANNING NUDGE
- LLM 返回 `AgentOutput` 时可通过 `plan_update` 字段替换整个计划，或 `current_plan_item` 推进索引

### 3.7 多 Provider LLM 架构

`llm/base.py` 定义了 `BaseChatModel` Protocol，统一接口：

```python
@runtime_checkable
class BaseChatModel(Protocol):
    model: str
    @property
    def provider(self) -> str: ...
    async def ainvoke(
        self, messages: list[BaseMessage], output_format: type[T] | None = None, **kwargs: Any
    ) -> ChatInvokeCompletion[T]: ...
```

支持的 Provider：
- OpenAI (ChatOpenAI)
- Anthropic (ChatAnthropic)
- Google (ChatGoogle via google-genai)
- BrowserUse 自研 (ChatBrowserUse) — 专针对浏览器任务 fine-tune
- Azure OpenAI
- Cerebras
- Mistral
- OCI (Oracle)
- Ollama (本地模型)

---

## 四、关键设计哲学

### 4.1 安全设计

**Sensitive Data 保护**：
- `Agent.__init__` 检查 `sensitive_data` 必须配合 `allowed_domains` 使用
- 域名单独配置时（`dict` 格式）验证域名覆盖关系
- 警告用户：如果只传 sensitive_data 但没有锁定 allowed_domains，可能被恶意网站窃取

**沙箱隔离**：Session 管理支持 profile 隔离、downloads_path 追踪

### 4.2 容错设计

**Fallback LLM 机制**：
```python
if isinstance(error, ModelRateLimitError) and self._fallback_llm:
    self.llm = self._fallback_llm  # 自动切换
    return await self.get_model_output(input_messages)
```

**连接恢复**：BrowserSession 检测连接断开时尝试自动重连（`RECONNECT_WAIT_TIMEOUT`）

**部分 Action 结果保留**：当批量 action 执行中途失败，已成功的 action 结果被保留而非回滚

### 4.3 可观测性

**多层日志**：
- 结构化 log_response：thinking/evaluation/next_goal 分色显示
- 截图路径、step 时间、token 消耗全程记录

**Telemetry**：`ProductTelemetry` 采集 anonymized 事件（模型、步骤数、成败率）用于产品改进

**GIT Event Bus**（`bubus`）：基于 WAL 持久化的 eventbus，session 级别事件追踪

---

## 五、Agent 状态机

```
AgentState {
  n_steps: int                           # 当前步数
  consecutive_failures: int             # 连续失败计数
  stopped: bool                          # 外部停止信号
  paused: bool                           # 外部暂停信号
  plan: list[PlanItem]                   # 当前计划
  current_plan_item_index: int           # 计划执行位置
  loop_detector: ActionLoopDetector      # 循环检测器
  last_result: list[ActionResult]        # 上一步结果
  last_model_output: AgentOutput | None # 上一步 LLM 输出
  file_system_state: FileSystemState    # 文件系统快照
  message_manager_state: MessageManagerState
}
```

---

## 六、技术债务与优化点

### 6.1 已知的性能问题

1. **CDP 连接不稳定**（TODO 注释）：
   > "currently we start a new websocket connection PER STEP, we should definitely keep this persistent"
   
2. **历史消息无上限增长**：虽然有 compaction，但 `max_history_items=None` 时理论无上限

3. **Shadow DOM 递归深度**：无限递归风险（TODO：无明确 max_depth 限制）

### 6.2 架构亮点 vs 改进空间

| 方面 | 亮点 | 改进空间 |
|------|------|------|
| DOM 序列化 | Paint-order 过滤 + iframe 嵌套处理完善 | 跨进程 CDP 连接未复用 |
| Action 系统 | 动态注册 + URL 过滤 + 统一执行引擎 | 缺乏 action 级别的 retry 机制 |
| Loop 检测 | 双重检测 + 软注入不强制阻止 | 阈值硬编码（5/8/12），不够自适应 |
| 消息压缩 | LLM summarization + 保留策略 | compaction 触发时机可更动态 |
| 错误处理 | 多层 fallback（fallback_llm → recovery） | 部分错误未区分可恢复/不可恢复 |

---

## 七、依赖生态

```
核心依赖：
├── cdp-use==1.4.5          # CDP 协议封装（类似 Playwright 的底层）
├── pydantic==2.12.5        # 数据模型 + 验证
├── openai==2.16.0          # OpenAI SDK（含完整 function calling）
├── anthropic==0.76.0       # Anthropic SDK
├── google-genai==1.65.0     # Google Gemini SDK
├── groq==1.0.0              # Groq SDK（超低延迟推理）
├── mcp==1.26.0              # Model Context Protocol（工具调用标准）
└── bubus==1.5.6            # EventBus（事件驱动解耦）

可选依赖：
├── [cli] textual==7.4.0     # CLI 终端 UI
├── [video] imageio+ffmpeg   # 视频录制
└── [eval] lmnr[all]==0.7.42  # 追踪/可观测性
```

---

## 八、与 OpenClaw 集成潜力

browser-use 的设计哲学与 OpenClaw 的 Agent 体系高度契合：

1. **DOM → Tool 映射层**：browser-use 的序列化 DOM 直接可以映射为 OpenClaw 的可用工具列表
2. **MCP 协议支持**：`browser_use.mcp` 模块已实现 MCP Server/Client，可作为 OpenClaw 的工具扩展
3. **CLI 工具**：`browser_use/skill_cli/` 提供了类 shell 的交互方式，可与 OpenClaw 的 CLI 模式互补
4. **Skills 体系**：`browser_use/skills/` 提供了 skill 注册机制，与 OpenClaw Skills 架构思路一致

---

## 九、总结

browser-use 是一个**工程化程度极高的 AI Agent 框架**，在浏览器自动化这个垂直领域做到了极致。其核心价值：

1. **可靠的 DOM → LLM 接口**：把混乱的网页结构变成整齐的序列化元素图，是让 LLM 控制浏览器的关键
2. **完整的容错体系**：fallback/重试/循环检测/消息压缩构成了长程任务的生命周期管理
3. **开放封闭原则**：核心 Agent 逻辑稳定，但 Tools/Models/Skills 均可扩展

对于 OpenClaw 而言，browser-use 的 DOM Service 和 Tools Registry 是可以借鉴的核心组件。其开源+商业双层模式也值得参考。

---

*报告生成：OpenClaw Daily Code Analysis | 2026-05-04*