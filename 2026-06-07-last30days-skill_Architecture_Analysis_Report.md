# last30days-skill 技术架构与源码研读报告

> 分析日期：2026-06-07
> 项目版本：v3.3.2
> 仓库地址：https://github.com/mvanhorn/last30days-skill
> 分析者：OpenClaw AI

---

## 一、项目概述

### 1.1 项目定位

`last30days-skill` 是一个**AI Agent 驱动的多平台研究工具**，能够跨 Reddit、X（Twitter）、YouTube、TikTok、Instagram、Hacker News、Polymarket、GitHub 等多个平台，对任意主题进行过去30天的社区讨论聚合，并通过AI合成引擎生成结构化的研究报告。

### 1.2 核心特性

- **零配置设计**：Reddit、HN、Polymarket、GitHub 开箱即用，无需API密钥
- **多平台并行搜索**：同时搜索12+社交媒体和内容平台
- **智能预研究**：通过WebSearch自动解析实体、社区、账号、仓库等
- **聚类分析**：跨平台内容自动聚类，识别同一事件的多平台讨论
- **AI合成引擎**：使用LLM将原始数据转化为结构化报告
- **HTML简报生成**：支持导出可分享的离线HTML报告
- **竞争者模式**：自动发现对比对象并生成多维度对比分析

### 1.3 项目规模

| 指标 | 数据 |
|------|------|
| 总Star数 | 12,924+（日增2821） |
| Python文件数 | 59个（不含vendor） |
| 总代码行数 | 21,711行 |
| 测试文件数 | 50+个 |
| 测试通过率 | 1,012+ 测试通过 |
| 支持平台 | 15+个数据源 |
| 技能规范版本 | v3.3.2 |

---

## 二、架构全景

### 2.1 架构层次图

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Interface Layer                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Claude Code  │  │  Codex CLI   │  │  OpenClaw / Other  │  │
│  │   Plugin     │  │   Skills     │  │     Agent Hosts     │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                        SKILL.md Contract Layer                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Voice Contract (8 LAWS) · Output Format · Query Planner  │  │
│  │  Pre-flight Resolution · Synthesis Rules · Citation Rules  │  │
│  └──────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                        Engine Core Layer                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  CLI Entry   │  │   Pipeline   │  │    Render Engine     │  │
│  │ last30days.py│  │  pipeline.py  │  │     render.py        │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                        Data Access Layer                         │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐  │
│  │ Reddit  │ │   X    │ │ YouTube│ │ TikTok │ │ Instagram│  │
│  ├────────┤ ├────────┤ ├────────┤ ├────────┤ ├──────────┤  │
│  │ Hacker  │ │Polymark│ │ GitHub │ │ Bluesky│ │   Digg   │  │
│  │  News   │ │  et    │ │        │ │        │ │          │  │
│  └────────┘ └────────┘ └────────┘ └────────┘ └──────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      Provider & Reasoning Layer                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Gemini  │  │  OpenAI  │  │   xAI    │  │  OpenRouter  │  │
│  │  Flash   │  │  GPT-5   │  │  Grok-4  │  │   (Brave)    │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心架构模式

#### 2.2.1 管道-过滤器模式（Pipeline-Filter）

项目采用经典的管道-过滤器架构，数据流单向流动：

```
[Query Input] → [Pre-flight Resolve] → [Planner] → [Parallel Fetch] → [Rerank] → [Cluster] → [Fusion] → [Render] → [Output]
```

每个阶段职责单一，通过数据契约（schema）连接，便于单元测试和独立扩展。

#### 2.2.2 策略模式（Strategy）

多提供商支持通过策略模式实现：
- `ReasoningClient` 抽象基类定义统一接口
- `GeminiClient`、`OpenAIClient`、`XAIClient` 具体实现
- 运行时根据环境变量和配置选择具体策略

#### 2.2.3 工厂模式（Factory）

数据源（Provider）的创建使用工厂模式：
- `available_sources()` 根据配置动态确定可用数据源
- 每个数据源独立实现搜索、获取、解析接口

#### 2.2.4 命令模式（Command）

通过 `--plan` 参数传入 JSON 查询计划，将查询意图与执行分离，支持：
- 模型生成的查询计划（LLM驱动）
- 确定性回退计划（无LLM时）
- 竞争者模式的多实体并行计划

---

## 三、核心模块详解

### 3.1 入口层：`last30days.py`

**职责**：命令行解析、参数校验、流程编排、输出保存

**关键设计**：

```python
# 支持多种emit模式
EMIT_MODES = ["compact", "md", "html", "json", "save"]

# 深度控制
DEPTH_SETTINGS = {
    "quick": {"per_stream_limit": 6, "pool_limit": 15, "rerank_limit": 12},
    "default": {"per_stream_limit": 12, "pool_limit": 40, "rerank_limit": 40},
    "deep": {"per_stream_limit": 20, "pool_limit": 60, "rerank_limit": 60},
}
```

**特性**：
- Python 3.12+ 强制检查
- 跨平台编码处理（Windows UTF-8）
- 子进程生命周期管理（SIGTERM优雅退出）
- 多种输出格式（compact/md/html/json）
- 自动保存到 `LAST30DAYS_MEMORY_DIR`

### 3.2 管道引擎：`pipeline.py`

**职责**：核心编排引擎，管理数据流和并发

**核心流程**：

```python
def run_pipeline(config, query_plan, emit_mode, depth):
    # 1. 解析可用数据源
    sources = available_sources(config, query_plan.sources)
    
    # 2. 并行获取（ThreadPoolExecutor）
    candidates = parallel_fetch(sources, query_plan.subqueries)
    
    # 3. 去重与归一化
    candidates = dedupe.normalize(candidates)
    
    # 4. 重排序（RRF + AI打分）
    candidates = rerank.score(candidates, query_plan)
    
    # 5. 聚类分析
    clusters = cluster.cluster_candidates(candidates)
    
    # 6. 融合与评分
    report = fusion.weighted_rrf(clusters)
    
    # 7. 渲染输出
    return render.render(report, emit_mode)
```

**并发模型**：
- 使用 `ThreadPoolExecutor` 进行 I/O 密集型并行
- 每个数据源独立线程，避免单点阻塞
- 超时控制：单个源超时不影响整体流程

### 3.3 数据模型：`schema.py`

**核心数据结构**：

```python
@dataclass(frozen=True)
class ProviderRuntime:
    reasoning_provider: Literal["gemini", "openai", "xai", "local"]
    planner_model: str
    rerank_model: str
    x_search_backend: Literal["xai", "bird"] | None = None

@dataclass(frozen=True)
class SubQuery:
    label: str
    search_query: str
    ranking_query: str
    sources: list[str]
    weight: float = 1.0

@dataclass
class Report:
    badge: list[str]
    synthesis: list[str]
    evidence_clusters: list[Cluster]
    stats: StatsBlock
    footer: list[str]
```

**设计亮点**：
- 使用 `frozen=True` 的不可变数据类，确保线程安全
- 递归清理 `None` 值的工具函数
- 类型注解完整，支持静态类型检查

### 3.4 渲染引擎：`render.py`

**职责**：将内部报告格式转换为各种输出格式

**输出模式**：

| 模式 | 用途 | 特点 |
|------|------|------|
| `compact` | 默认用户输出 | 压缩格式，适合终端显示 |
| `md` | 调试/检查 | 完整Markdown，包含原始证据 |
| `html` | 分享 | 自包含HTML，离线可用，暗色模式 |
| `json` | 机器处理 | 结构化数据，用于下游消费 |
| `save` | 持久化 | 保存到磁盘，不输出到stdout |

**HTML渲染特色**：
- 自包含（内联CSS，无外部依赖）
- 暗色/亮色模式自适应
- 系统字体回退（Inter + JetBrains Mono）
- 打印友好
- 无JavaScript，完全静态

### 3.5 查询规划器：`planner.py`

**职责**：将用户意图转化为结构化查询计划

**两种工作模式**：

1. **LLM驱动模式**（推荐）：
   - 由宿主模型（Claude、GPT等）生成JSON计划
   - 通过 `--plan` 参数传入
   - 更智能的意图理解和子查询分解

2. **确定性回退模式**：
   - 无LLM时使用规则引擎
   - 基于关键词和类别的模板匹配
   - 保证基础功能可用

**查询计划JSON结构**：

```json
{
  "intent": "breaking_news",
  "freshness_mode": "strict_recent",
  "cluster_mode": "story",
  "subqueries": [
    {
      "label": "primary",
      "search_query": "kanye west",
      "ranking_query": "What notable events involving Kanye West happened in the last 30 days?",
      "sources": ["reddit", "x", "hackernews", "youtube", "tiktok", "instagram"],
      "weight": 1.0
    }
  ]
}
```

### 3.6 数据源模块

每个数据源独立实现，统一接口：

| 模块 | 数据源 | 认证方式 | 特点 |
|------|--------|----------|------|
| `reddit.py` | Reddit | 公开JSON（免费） | 最高信号质量 |
| `reddit_public.py` | Reddit公共 | 无需认证 | 备用方案 |
| `xai_x.py` | X/Twitter | xAI API Key | 官方API |
| `xquik.py` | X/Twitter | 浏览器Cookie | 免费方案 |
| `xurl_x.py` | X/Twitter | xurl CLI | 高级方案 |
| `bird_x.py` | X/Twitter | Bird Search | 专业方案 |
| `youtube_yt.py` | YouTube | yt-dlp（本地） | 转录+评论 |
| `tiktok.py` | TikTok | ScrapeCreators API | 病毒信号 |
| `instagram.py` | Instagram | ScrapeCreators API | 创作者视角 |
| `hackernews.py` | Hacker News | Algolia API（免费） | 开发者共识 |
| `polymarket.py` | Polymarket | Gamma API（免费） | 真金白银 |
| `github.py` | GitHub | gh CLI / API | 代码活跃度 |
| `bluesky.py` | Bluesky | App Password | 去中心化 |
| `truthsocial.py` | Truth Social | Token | 特定社区 |
| `digg.py` | Digg | digg-pp-cli | AI聚合 |
| `perplexity.py` | Perplexity | OpenRouter | 网页搜索 |
| `grounding.py` | Web | Brave/Exa/Serper | 通用搜索 |

### 3.7 聚类与融合算法

**聚类模块 `cluster.py`**：
- 基于实体重叠的跨平台内容合并
- 使用标题相似度和实体提取识别同一事件
- 支持多语言内容匹配

**融合模块 `fusion.py`**：
- 加权倒数排名融合（Weighted RRF）
- 考虑来源权重、时效性、互动信号
- 跨平台信号整合

```python
# RRF公式：score = Σ(weight_i / (rank_i + k))
def weighted_rrf(clusters, k=60):
    for cluster in clusters:
        score = sum(
            subquery.weight / (item.rank + k)
            for subquery in query_plan.subqueries
            for item in cluster.items
        )
        cluster.score = score
    return sorted(clusters, key=lambda c: c.score, reverse=True)
```

### 3.8 重排序模块 `rerank.py`

**职责**：AI驱动的结果质量评估

**两阶段评估**：
1. **相关性评估**：LLM判断内容与查询意图的匹配度
2. **趣味性评估**（Fun Judge）：评估幽默度、 viral 潜力

**评估维度**：
- 信息密度
- 社区互动信号（点赞、评论、转发）
- 来源权威性
- 时效性
- 表达质量（趣味评估）

---

## 四、SKILL.md 契约系统

### 4.1 设计哲学

`SKILL.md` 是一个**1400+行的指令契约**，定义了精确的技能输出格式。它代表了 AI Agent 技能设计的一种**极端规范化**范式：

> "这不是一个通用的'过去30天'研究提示。不要将其视为可以随意即兴发挥的搜索关键词。"

### 4.2 八条不可违法则（LAWS）

| 法则 | 内容 | 目的 |
|------|------|------|
| **LAW 1** | 禁止末尾 `Sources:` 块 | 避免模型自动生成引用列表 |
| **LAW 2** | 禁止发明标题行 | 防止博客式叙事格式 |
| **LAW 3** | 禁止使用破折号 | 避免AI生成文本特征 |
| **LAW 4** | 禁止 `##` 章节标题 | 保持一致的输出结构 |
| **LAW 5** | 必须透传引擎页脚 | 保持数据一致性 |
| **LAW 6** | 禁止原始证据聚类 | 强制合成而非转储 |
| **LAW 7** | 命名实体必须传入 `--plan` | 确保查询质量 |
| **LAW 8** | 所有引用必须是内联Markdown链接 | 规范引用格式 |

### 4.3 预研究智能（Pre-Research Intelligence）

**Step 0.55** 是技能的独特设计：

在运行引擎前，通过WebSearch自动解析：
- X/Twitter 账号（主账号、公司账号、相关评论员）
- Reddit 社区（品牌专属 + 类别同伴）
- GitHub 用户/仓库
- TikTok 标签和创作者
- Instagram 创作者
- YouTube 查询词

**类别同伴扩展**：
对于产品类查询，自动加入类别相关的Reddit社区（如AI图像生成工具自动加入 `r/StableDiffusion`、`r/midjourney`）。

### 4.4 输出格式规范

**通用查询格式**：
```
🌐 last30days v{VERSION} · synced {YYYY-MM-DD}

What I learned:

**{标题1}** - {内容1}，per [@handle](url)

**{标题2}** - {内容2}，per [r/subreddit](url)

KEY PATTERNS from the research:
1. {模式1} - per [@handle](url)
2. {模式2} - per [r/subreddit](url)
3. {模式3} - per [@handle](url)

✅ All agents reported back!
├─ 🟠 Reddit: N threads (M upvotes)
├─ 🔵 X: N posts (M likes)
├─ 🔴 YouTube: N videos (M views)
...

I'm now an expert on {TOPIC}. Some things you could ask:
- {具体问题1}
- {具体问题2}
- {具体问题3}
```

**对比查询格式**：
```
# {A} vs {B}: What the Community Says

## Quick Verdict
## {Entity A}
## {Entity B}
## Head-to-Head
## The Bottom Line
## The emerging stack
```

---

## 五、关键技术决策分析

### 5.1 为什么使用纯Python标准库？

`pyproject.toml` 中 `dependencies = []` —— 零外部依赖！

**设计理由**：
1. **部署简单**：无需处理依赖冲突，直接复制即可运行
2. **跨平台兼容**：减少平台特定问题
3. **Agent环境友好**：AI Agent 运行环境通常有Python但可能没有复杂依赖
4. **版本稳定**：避免依赖版本漂移

**权衡**：
- 优点：部署零摩擦，启动即运行
- 缺点：需要自行实现HTTP请求、JSON解析等基础功能
- 解决方案：使用 `http.py` 封装基础HTTP功能，保持简洁

### 5.2 为什么使用 ThreadPoolExecutor 而非 asyncio？

**选择 ThreadPoolExecutor 的原因**：
1. **I/O 密集但调用简单**：主要是HTTP请求，不需要复杂的异步状态管理
2. **代码可读性**：同步代码更易于理解和调试
3. **错误处理**：同步异常处理更直观
4. **混合调用**：部分操作需要调用子进程（如 `yt-dlp`），线程池更自然

### 5.3 为什么使用 SKILL.md 而非 JSON/YAML 配置？

**SKILL.md 作为契约的优势**：
1. **人类可读**：1400行Markdown，结构和规则清晰可见
2. **模型友好**：LLM原生理解Markdown，上下文学习效果更好
3. **版本控制**：Git diff 清晰展示变更
4. **自文档化**：既是配置也是文档

### 5.4 为什么强调 "模型即规划器"？

```
"YOU generate the JSON query plan. You do not need an API key... YOU are the LLM."
```

**设计哲学**：
- 将LLM的能力最大化利用，而非仅作为后处理工具
- 减少外部API调用，降低延迟和成本
- 允许模型根据上下文动态调整查询策略

### 5.5 为什么有严格的Voice Contract？

**8条LAWS的目的**：
1. **一致性**：无论哪个模型运行，输出格式一致
2. **可预测性**：用户知道会得到什么格式的回复
3. **质量保障**：防止模型"即兴发挥"导致质量下降
4. **Anti-Slop**：避免AI生成文本的常见特征（如em-dash、博客标题等）

---

## 六、代码质量与工程实践

### 6.1 测试策略

**测试规模**：1,012+ 测试通过

**测试分类**：
- 单元测试：每个数据源模块独立测试
- 集成测试：管道端到端测试
- 回归测试：防止已知失败模式复现
- 安全测试：权限和凭证处理测试
- 版本一致性测试：确保版本号同步

### 6.2 版本管理

**版本追踪**：
- `CHANGELOG.md`：47,685字详细变更日志
- `SKILL.md` 前置版本声明
- `.claude-plugin/plugin.json` 插件版本
- 测试确保版本一致性

### 6.3 代码规范

- `ruff` 代码检查（`# ruff: noqa: E402` 等抑制说明）
- 类型注解完整
- 文档字符串规范
- 错误处理显式（`raise SystemExit` 而非隐式退出）

### 6.4 安全设计

**安全原则**：
- 只读访问：不发布、点赞、修改任何平台内容
- 凭证隔离：不同API的密钥不混用
- 无日志凭证：API密钥不写入输出文件
- 本地处理：yt-dlp 等工具本地运行，不上传数据
- 透明权限：`Security & Permissions` 章节明确列出所有数据去向

---

## 七、扩展性分析

### 7.1 添加新数据源

**步骤**：
1. 在 `lib/` 下创建新模块（如 `new_platform.py`）
2. 实现搜索接口：`search(query, config)` → `list[Candidate]`
3. 在 `pipeline.py` 的 `MOCK_AVAILABLE_SOURCES` 中添加标识
4. 在 `providers.py` 中注册运行时配置
5. 在 `render.py` 的 `SOURCE_LABELS` 中添加显示名称
6. 添加单元测试

**设计模式**：每个数据源独立，无需修改核心管道逻辑

### 7.2 添加新输出格式

**步骤**：
1. 在 `render.py` 中添加新emit函数
2. 在 `last30days.py` 的 `EMIT_MODES` 中注册
3. 保持 `Report` 数据模型不变

### 7.3 自定义查询计划

通过 `--plan` 参数传入自定义JSON，支持：
- 自定义子查询权重
- 自定义数据源组合
- 自定义聚类模式
- 自定义时效性策略

---

## 八、竞品对比视角

### 8.1 与通用搜索引擎的差异

| 维度 | Google Search | last30days |
|------|---------------|------------|
| 数据源 | 网页索引 | 社交媒体实时讨论 |
| 排序信号 | SEO、PageRank | 社区互动（点赞、评论、投票） |
| 时效性 | 任意时间 | 最近30天 |
| 内容形式 | 文章、博客 | 帖子、评论、视频转录 |
| 观点多样性 | 编辑观点 | 社区多元观点 |
| 预测市场 | 无 | Polymarket真金白银数据 |

### 8.2 与Perplexity的差异

| 维度 | Perplexity | last30days |
|------|------------|------------|
| 搜索范围 | 通用网页 | 专注社交媒体 |
| 引用格式 | 文末引用 | 内联Markdown链接 |
| 趣味性 | 中立客观 | 社区趣味评估（Best Takes） |
| 对比分析 | 基础 | 多实体并行对比（竞争者模式） |
| 离线使用 | 需要网络 | 可导出离线HTML简报 |

### 8.3 与GitHub Copilot的Skills差异

last30days-skill 是 **Agent Skill** 架构的标杆实现：
- 1400+ 行 SKILL.md 契约定义
- 8条不可违输出法则
- 预研究智能（Step 0.55）
- 多平台并行管道
- 1012+ 测试保障

---

## 九、源码研读感悟

### 9.1 架构设计的极致 pragmatic

这个项目展示了**"足够好"的工程哲学**：
- 使用标准库而非追逐最新框架
- 使用线程池而非asyncio（因为I/O模式简单）
- 使用Markdown而非JSON定义契约（因为LLM读Markdown更自然）
- 零依赖部署，保证在任何环境都能运行

### 9.2 AI原生设计的先驱

`last30days` 是**AI-first** 而非 **AI-enhanced** 的设计：
- 假设LLM是核心参与者（"YOU are the planner"）
- 输出格式专为LLM消费优化（compact模式便于上下文窗口）
- 反AI-slop设计（8条LAWS专门防止AI生成文本特征）
- 自我纠错机制（Pre-flight检查、Self-check步骤）

### 9.3 社区信号的工程化

将分散的社区讨论转化为结构化情报的完整工程方案：
- 15+平台的数据标准化
- 跨平台实体对齐（聚类算法）
- 社区互动信号的量化（RRF融合）
- AI驱动的质量评估（重排序）
- 趣味内容的识别与展示（Fun Judge）

### 9.4 契约驱动的可靠性

SKILL.md 的 1400+ 行契约是一种**软件工程的创新**：
- 将LLM的"不确定性"通过严格契约约束
- 用"LAWS"而非"建议"确保执行
- 用实际失败案例（如 Peter Steinberger disaster）教育模型
- 版本化的契约演进（v3.0.6 → v3.0.7 → v3.3.2）

---

## 十、关键文件索引

| 文件 | 职责 | 行数 |
|------|------|------|
| `skills/last30days/scripts/last30days.py` | CLI入口 | 900+ |
| `skills/last30days/scripts/lib/pipeline.py` | 核心管道 | 1000+ |
| `skills/last30days/scripts/lib/render.py` | 渲染引擎 | 1700+ |
| `skills/last30days/scripts/lib/schema.py` | 数据模型 | 300+ |
| `skills/last30days/scripts/lib/providers.py` | 提供商管理 | 400+ |
| `skills/last30days/scripts/lib/planner.py` | 查询规划 | 400+ |
| `skills/last30days/scripts/lib/rerank.py` | 重排序 | 300+ |
| `skills/last30days/scripts/lib/cluster.py` | 聚类分析 | 200+ |
| `skills/last30days/scripts/lib/fusion.py` | 融合算法 | 100+ |
| `skills/last30days/SKILL.md` | 技能契约 | 1400+ |
| `skills/last30days/scripts/lib/reddit.py` | Reddit数据源 | 300+ |
| `skills/last30days/scripts/lib/youtube_yt.py` | YouTube数据源 | 300+ |
| `skills/last30days/scripts/lib/xai_x.py` | X数据源 | 200+ |
| `skills/last30days/scripts/lib/polymarket.py` | Polymarket数据源 | 150+ |
| `skills/last30days/scripts/lib/github.py` | GitHub数据源 | 200+ |

---

## 十一、总结与评价

### 11.1 项目评分

| 维度 | 评分 | 说明 |
|------|------|------|
| 架构设计 | ★★★★★ | 简洁、清晰、可扩展 |
| 代码质量 | ★★★★☆ | 类型注解完整，测试覆盖良好 |
| 工程实践 | ★★★★★ | 零依赖、多平台、版本管理严格 |
| 创新性 | ★★★★★ | AI原生技能设计的标杆 |
| 文档质量 | ★★★★★ | 1400+行契约，自我纠错机制 |
| 社区热度 | ★★★★★ | GitHub Trending #1，日增2800+ Star |

### 11.2 学习价值

1. **AI Agent 技能设计模式**：SKILL.md 契约系统是可复制的范式
2. **零依赖架构**：如何在复杂功能与简单部署间取得平衡
3. **多平台数据整合**：社交媒体数据的获取、标准化、融合方法
4. **LLM 输出控制**：如何通过工程手段约束模型的输出格式
5. **社区驱动设计**：以真实社区信号替代编辑观点的产品哲学

### 11.3 适用场景

- 竞品分析与市场调研
- 人物背景调查（面试、会议前）
- 热点话题追踪与趋势分析
- 产品反馈收集（跨平台）
- 投资决策支持（Polymarket信号）
- 旅行/消费决策（社区真实评价）

---

> **报告结语**：
> 
> `last30days-skill` 不仅仅是一个搜索工具，它代表了一种**AI原生软件设计**的新范式：将LLM作为核心参与者而非外部API，通过严格契约确保可靠性，以社区信号替代传统权威，用零依赖架构保证可部署性。它的1400+行SKILL.md是人类与AI协作的精密契约，也是未来AI Agent技能设计的参考标准。
> 
> "Google aggregates editors. /last30days searches people."

---

*本报告由 OpenClaw AI 自动生成，基于 last30days-skill v3.3.2 源码分析。*
*分析时间：2026-06-07 03:00 CST*
