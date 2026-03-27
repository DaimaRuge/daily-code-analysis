# last30days-skill 技术架构与源码研读报告

**分析日期**: 2026年3月28日  
**项目名称**: last30days-skill  
**项目地址**: https://github.com/mvanhorn/last30days-skill  
**分析工具**: OpenClaw AI Assistant

---

## 目录

1. [项目概述](#项目概述)
2. [技术架构设计](#技术架构设计)
3. [核心模块详解](#核心模块详解)
4. [数据流程分析](#数据流程分析)
5. [关键技术亮点](#关键技术亮点)
6. [代码质量评估](#代码质量评估)
7. [使用与部署指南](#使用与部署指南)
8. [总结与建议](#总结与建议)

---

## 项目概述

### 项目简介

`last30days-skill` 是一个用于研究过去 30 天内任何主题的 OpenClaw 技能。它整合了多个社交平台（Reddit、X、Bluesky、YouTube、TikTok、Instagram、HackerNews、小红书、Polymarket 等）的数据，通过 AI 模型进行智能分析和综合，生成高质量的研究简报。

### 核心功能

- **多平台数据源集成**: 支持 14+ 个社交媒体和内容平台
- **智能查询优化**: 根据主题特性自动调整搜索策略
- **AI 驱动的内容筛选**: 利用大语言模型评估内容相关性和质量
- **多模式输出**: 支持紧凑模式、JSON、Markdown、上下文等多种输出格式
- **可配置的研究深度**: 快速模式、默认模式、深度模式三种选择
- **数据持久化**: 支持 SQLite 数据库存储研究结果

### 技术栈

| 类别 | 技术/库 |
|------|---------|
| 编程语言 | Python 3.10+ |
| AI 模型 | OpenAI GPT-4/4.5, Anthropic Claude 3.5, xAI Grok 3, Google Gemini 2.0 Flash |
| HTTP 请求 | httpx, requests |
| 并行处理 | ThreadPoolExecutor, concurrent.futures |
| 数据验证 | pydantic |
| 数据库 | SQLite |
| 终端 UI | rich |
| 配置管理 | python-dotenv |

---

## 技术架构设计

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                          用户接口层                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │   SKILL.md  │  │  README.md  │  │  CLI (Python)│            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
└─────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                        核心协调层 (last30days.py)                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ 参数解析 │  │ 超时管理 │  │ 信号处理 │  │ 子进程   │       │
│  │与配置    │  │          │  │          │  │ 管理      │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
        ▼                          ▼                          ▼
┌───────────────┐        ┌───────────────┐        ┌───────────────┐
│  数据源集成层  │        │  AI 处理层    │        │  输出与渲染层  │
├───────────────┤        ├───────────────┤        ├───────────────┤
│ • reddit.py   │        │ • models.py   │        │ • render.py   │
│ • bird_x.py   │        │ • score.py    │        │ • ui.py       │
│ • bluesky.py  │        │ • dedupe.py   │        │ • normalize.py│
│ • youtube_yt.py│       │ • entity_extract.py │   │ • schema.py   │
│ • tiktok.py   │        │ • query_type.py │      │               │
│ • instagram.py│        │               │        │               │
│ • hackernews.py│       │               │        │               │
│ • xiaohongshu_api.py │ │               │        │               │
│ • polymarket.py│       │               │        │               │
│ • websearch.py│        │               │        │               │
└───────────────┘        └───────────────┘        └───────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                        基础设施层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ env.py   │  │ http.py  │  │ cache.py │  │ dates.py │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

### 设计模式分析

#### 1. 模块化设计模式

项目采用高度模块化的架构，每个数据源和功能组件都封装在独立的模块中：

```python
# 每个数据源都是独立的模块
from lib import (
    bird_x,           # X (Twitter) 数据源
    bluesky,          # Bluesky 数据源
    reddit,           # Reddit 数据源
    youtube_yt,       # YouTube 数据源
    # ... 更多模块
)
```

**优势**:
- 易于维护和测试
- 支持独立开发和更新
- 降低耦合度

#### 2. 策略模式

项目在多个方面采用了策略模式：

1. **模型选择策略**: 支持多种 AI 模型，可根据需要切换
2. **搜索源策略**: 可根据主题特性选择合适的搜索源
3. **输出渲染策略**: 支持多种输出格式

```python
# 模型选择策略示例
MODEL_PREFERENCES = {
    "openai": {
        "primary": "gpt-4.5-preview-2025-02-27",
        "fallback": "gpt-4o-2024-11-20",
    },
    "anthropic": {
        "primary": "claude-3-5-sonnet-20241022",
    },
    # ...
}
```

#### 3. 观察者模式

项目使用信号处理和超时机制来实现观察者模式：

```python
# 超时监控和信号处理
def _install_global_timeout(timeout_seconds: int):
    if hasattr(signal, 'SIGALRM'):
        def _handler(signum, frame):
            sys.stderr.write(f"\n[TIMEOUT] Global timeout ({timeout_seconds}s) exceeded.\n")
            _cleanup_children()
            sys.exit(1)
        signal.signal(signal.SIGALRM, _handler)
        signal.alarm(timeout_seconds)
```

---

## 核心模块详解

### 1. 主协调模块 (last30days.py)

#### 模块职责

- 命令行参数解析
- 全局配置管理
- 数据源调度和协调
- 超时和异常处理
- 子进程管理

#### 关键设计

**超时配置管理**:
```python
TIMEOUT_PROFILES = {
    "quick":   {"global": 90,  "future": 30, ...},
    "default": {"global": 180, "future": 60, ...},
    "deep":    {"global": 300, "future": 90, ...},
}
```

**子进程安全清理**:
```python
_child_pids: set = set()
_child_pids_lock = threading.Lock()

def register_child_pid(pid: int):
    with _child_pids_lock:
        _child_pids.add(pid)

def _cleanup_children():
    with _child_pids_lock:
        pids = list(_child_pids)
    for pid in pids:
        try:
            os.killpg(os.getpgid(pid), signal.SIGTERM)
        except (ProcessLookupError, PermissionError, OSError):
            pass
```

#### 工作流程

```
用户输入主题
    │
    ▼
参数解析与配置加载
    │
    ▼
安装全局超时监控
    │
    ▼
查询类型分析 (query_type)
    │
    ├───────────────────┬───────────────────┐
    │                   │                   │
    ▼                   ▼                   ▼
Reddit 搜索         X (Twitter) 搜索    Bluesky 搜索
YouTube 搜索        TikTok 搜索         Instagram 搜索
HackerNews 搜索     小红书搜索          Polymarket 搜索
Web 搜索
    │                   │                   │
    └───────────────────┴───────────────────┘
                        │
                        ▼
            数据去重与合并 (dedupe)
                        │
                        ▼
            AI 评分与排序 (score)
                        │
                        ▼
            实体提取 (entity_extract)
                        │
                        ▼
            输出渲染 (render)
                        │
                        ▼
            返回给用户
```

### 2. 数据源模块

#### Reddit 模块 (reddit.py)

**核心功能**:
- 使用 OpenAI Responses API 搜索 Reddit
- 支持 ScrapeCreators API 作为替代方案
- 提取帖子和评论数据
- 过滤 NSFW 内容

**关键代码结构**:
```python
# Reddit 搜索的双策略实现
def search_reddit(
    topic: str,
    from_date: str,
    to_date: str,
    limit: int = 20,
    # ...
) -> tuple[list[RedditItem], str | None, str | None]:
    # 优先使用 ScrapeCreators
    if sc_token:
        return _search_reddit_scrapecreators(...)
    # 回退到 OpenAI Responses API
    else:
        return _search_reddit_openai(...)
```

#### X (Twitter) 模块 (bird_x.py, scrapecreators_x.py, xai_x.py)

**多策略搜索**:
1. **bird_x.py**: 使用 OpenAI Responses API 直接搜索 X
2. **scrapecreators_x.py**: 使用 ScrapeCreators API
3. **xai_x.py**: 使用 xAI 的搜索 API

```python
# X 搜索的策略选择
def search_x(
    topic: str,
    from_date: str,
    to_date: str,
    limit: int = 20,
    # ...
) -> tuple[list[XItem], str | None, str | None]:
    # 尝试多种策略，直到成功
    strategies = [
        (_search_x_scrapecreators, "scrapecreators"),
        (_search_x_bird, "bird"),
        (_search_x_xai, "xai"),
    ]
    for strategy, name in strategies:
        try:
            result = strategy(...)
            if result[0]:
                return result
        except Exception as e:
            logger.warning(f"{name} strategy failed: {e}")
    return [], None, "All strategies failed"
```

#### YouTube 模块 (youtube_yt.py)

**功能特点**:
- 使用 YouTube Data API v3
- 支持按日期范围搜索
- 提取视频元数据和统计信息
- 获取视频缩略图

```python
def search_youtube(
    topic: str,
    from_date: str,
    to_date: str,
    limit: int = 20,
) -> list[YouTubeItem]:
    # 调用 YouTube API
    # 处理和规范化结果
    # 返回结构化数据
```

### 3. AI 处理模块

#### 评分模块 (score.py)

**核心功能**:
- 使用 LLM 评估内容的相关性和质量
- 支持批量评分以提高效率
- 实现缓存机制避免重复评分

**评分维度**:
1. **相关性评分** (1-10): 内容与主题的相关程度
2. **质量评分** (1-10): 内容的质量和可信度
3. **整体评分** (1-10): 综合评分

```python
async def score_items(
    items: list[Any],
    topic: str,
    model_config: dict,
    # ...
) -> list[ScoredItem]:
    # 构建评分提示
    # 调用 LLM 进行评分
    # 解析和规范化评分结果
    # 返回带评分的项目列表
```

#### 去重模块 (dedupe.py)

**去重策略**:
1. **URL 去重**: 相同 URL 的内容只保留一个
2. **内容相似度去重**: 使用文本相似度算法
3. **语义去重**: 使用 LLM 判断内容是否语义重复

```python
def deduplicate_items(items: list[Any]) -> list[Any]:
    # URL 去重
    seen_urls = set()
    unique_by_url = []
    for item in items:
        if item.url not in seen_urls:
            seen_urls.add(item.url)
            unique_by_url.append(item)
    
    # 内容相似度去重
    # ...
    
    return unique_by_url
```

#### 实体提取模块 (entity_extract.py)

**提取的实体类型**:
- 人物
- 组织
- 产品
- 地点
- 事件

```python
def extract_entities(
    items: list[Any],
    topic: str,
    model_config: dict,
) -> list[Entity]:
    # 从所有内容中提取实体
    # 统计实体出现频率
    # 过滤和排序实体
    # 返回重要实体列表
```

### 4. 输出渲染模块 (render.py)

**支持的输出格式**:
1. **compact**: 紧凑的终端友好格式
2. **json**: 结构化的 JSON 格式
3. **md**: Markdown 格式
4. **context**: 适合作为 AI 上下文的格式
5. **path**: 保存到文件的路径

**渲染流程**:
```
原始数据
    │
    ▼
数据规范化 (normalize.py)
    │
    ▼
格式选择
    ├─────────┬─────────┬─────────┬─────────┬─────────┐
    │         │         │         │         │         │
    ▼         ▼         ▼         ▼         ▼         ▼
 compact    json       md      context    path    其他...
    │         │         │         │         │         │
    └─────────┴─────────┴─────────┴─────────┴─────────┘
              │
              ▼
         输出结果
```

---

## 数据流程分析

### 端到端数据流

让我们以一个实际的查询 "AI 大模型最新发展" 为例，分析完整的数据流程：

#### 阶段 1: 查询分析与配置

```python
# 用户输入
topic = "AI 大模型最新发展"
mode = "default"

# 步骤 1: 分析查询类型
query_type = qt.analyze_query(topic)
# 返回: {
#   "type": "technology",
#   "recommended_sources": ["reddit", "x", "hackernews", "web"],
#   "time_sensitivity": "high",
#   "estimated_complexity": "medium"
# }

# 步骤 2: 加载配置
config = env.load_config()
# 包含所有 API keys 和模型配置
```

#### 阶段 2: 并行数据采集

```python
# 使用 ThreadPoolExecutor 并行执行多个搜索
with ThreadPoolExecutor(max_workers=8) as executor:
    futures = []
    
    # Reddit 搜索
    futures.append(executor.submit(
        reddit.search_reddit, topic, from_date, to_date, limit=30
    ))
    
    # X 搜索
    futures.append(executor.submit(
        bird_x.search_x, topic, from_date, to_date, limit=30
    ))
    
    # HackerNews 搜索
    futures.append(executor.submit(
        hackernews.search_hackernews, topic, from_date, to_date, limit=20
    ))
    
    # Web 搜索
    futures.append(executor.submit(
        websearch.search_web, topic, from_date, to_date, limit=20
    ))
    
    # YouTube 搜索
    futures.append(executor.submit(
        youtube_yt.search_youtube, topic, from_date, to_date, limit=15
    ))
    
    # 收集结果
    all_results = []
    for future in as_completed(futures):
        try:
            items, _, _ = future.result()
            all_results.extend(items)
        except Exception as e:
            logger.warning(f"Search failed: {e}")
```

#### 阶段 3: 数据处理与筛选

```python
# 步骤 1: 去重
unique_items = dedupe.deduplicate_items(all_results)

# 步骤 2: AI 评分
scored_items = score.score_items(
    unique_items, 
    topic, 
    model_config,
    cache_db_path
)

# 步骤 3: 筛选和排序
top_items = sorted(
    scored_items, 
    key=lambda x: x.overall_score, 
    reverse=True
)[:50]

# 步骤 4: 实体提取
entities = entity_extract.extract_entities(
    top_items, 
    topic, 
    model_config
)
```

#### 阶段 4: 综合与输出

```python
# 构建简报数据
briefing_data = {
    "topic": topic,
    "period": {"from": from_date, "to": to_date},
    "sources": used_sources,
    "key_findings": key_findings,
    "timeline": timeline,
    "entities": entities,
    "top_items": top_items,
    "metadata": metadata
}

# 根据格式渲染输出
if output_format == "compact":
    output = render.render_compact(briefing_data)
elif output_format == "md":
    output = render.render_markdown(briefing_data)
elif output_format == "json":
    output = render.render_json(briefing_data)
# ... 其他格式

# 输出或保存结果
print(output)
```

---

## 关键技术亮点

### 1. 多策略数据源集成

**亮点**: 项目实现了灵活的多策略数据源集成，每个平台都支持多种搜索方法。

```python
# X 搜索的三种策略
strategies = [
    (_search_x_scrapecreators, "scrapecreators"),  # 优先使用 ScrapeCreators
    (_search_x_bird, "bird"),                        # 然后使用 OpenAI Responses
    (_search_x_xai, "xai"),                          # 最后使用 xAI
]
```

**技术价值**:
- 提高了系统的可靠性和容错能力
- 避免了单一 API 依赖的风险
- 允许根据成本和质量灵活选择

### 2. 智能缓存系统

**亮点**: 实现了分层缓存机制，包括内存缓存和 SQLite 持久化缓存。

```python
# 缓存配置
CACHE_TTL = {
    "search": 3600,           # 搜索结果缓存 1 小时
    "scoring": 86400,         # 评分结果缓存 1 天
    "entity_extraction": 86400,  # 实体提取缓存 1 天
}
```

**技术价值**:
- 显著降低 API 调用成本
- 提高响应速度
- 支持离线分析和重复查询

### 3. 并行处理架构

**亮点**: 充分利用 Python 的并发编程能力，实现高效的并行数据采集。

```python
# 并行搜索执行
with ThreadPoolExecutor(max_workers=8) as executor:
    futures = [executor.submit(search_fn) for search_fn in search_functions]
    for future in as_completed(futures):
        process_result(future.result())
```

**技术价值**:
- 将总搜索时间从串行的几分钟降低到几十秒
- 优化资源利用率
- 提升用户体验

### 4. LLM 驱动的智能筛选

**亮点**: 创新性地使用 LLM 进行内容质量评估和相关性判断。

```python
# 评分提示工程
SCORE_PROMPT = """
评估以下内容与主题 "{topic}" 的相关性和质量：

内容: {content}

请给出三个评分（1-10分）：
1. 相关性评分：内容与主题的相关程度
2. 质量评分：内容的质量、可信度和深度
3. 整体评分：综合考虑以上因素

格式: JSON
{{
    "relevance_score": number,
    "quality_score": number,
    "overall_score": number,
    "reasoning": string
}}
"""
```

**技术价值**:
- 超越简单关键词匹配，实现语义级理解
- 自动识别高质量内容
- 持续优化筛选标准

### 5. 健壮的错误处理和超时机制

**亮点**: 实现了多层次的错误处理和超时监控系统。

```python
# 全局超时 + 子进程管理
def _install_global_timeout(timeout_seconds: int):
    def _handler(signum, frame):
        _cleanup_children()  # 清理所有子进程
        sys.exit(1)
    signal.signal(signal.SIGALRM, _handler)
    signal.alarm(timeout_seconds)
```

**技术价值**:
- 防止进程挂起
- 保证资源释放
- 提高系统稳定性

---

## 代码质量评估

### 优点

#### 1. 代码组织清晰
- 模块划分合理，职责单一
- 命名规范，可读性强
- 目录结构清晰易懂

#### 2. 注释完整
- 关键函数都有详细的文档字符串
- 复杂逻辑有注释说明
- 使用类型注解提高代码可维护性

#### 3. 错误处理完善
- 多层异常捕获和处理
- 合理的回退策略
- 详细的日志记录

#### 4. 配置管理规范
- 使用环境变量管理敏感信息
- 支持多种配置文件格式
- 配置验证机制完善

### 改进建议

#### 1. 测试覆盖
**现状**: 项目缺少自动化测试
**建议**:
- 添加单元测试覆盖核心功能
- 添加集成测试验证端到端流程
- 使用 pytest 框架组织测试

```python
# 建议的测试结构
tests/
├── unit/
│   ├── test_dedupe.py
│   ├── test_score.py
│   └── test_normalize.py
├── integration/
│   └── test_end_to_end.py
└── fixtures/
    └── sample_data.json
```

#### 2. 类型注解完善
**现状**: 部分函数缺少类型注解
**建议**:
- 为所有公共 API 添加类型注解
- 使用 mypy 进行静态类型检查
- 考虑使用 dataclasses 或 pydantic 模型

#### 3. 性能优化
**现状**: 部分处理可以进一步优化
**建议**:
- 考虑使用 asyncio 替代线程池
- 实现更智能的批量处理
- 添加性能监控和分析

#### 4. 文档完善
**现状**: 缺少 API 文档和开发者指南
**建议**:
- 使用 Sphinx 生成 API 文档
- 添加架构设计文档
- 提供更多使用示例

---

## 使用与部署指南

### 环境要求

```bash
# 系统要求
- Python 3.10+
- 4GB+ RAM
- 稳定的网络连接

# Python 依赖
- httpx
- requests
- pydantic
- rich
- python-dotenv
```

### 快速开始

#### 1. 安装和配置

```bash
# 克隆项目
git clone https://github.com/mvanhorn/last30days-skill.git
cd last30days-skill

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env 文件，填入你的 API keys
```

#### 2. 基本使用

```bash
# 快速模式研究一个主题
python scripts/last30days.py --mode quick "AI 大模型"

# 默认模式，输出 Markdown
python scripts/last30days.py --output md "气候变化"

# 深度模式，保存到文件
python scripts/last30days.py --mode deep --output report.md "量子计算"
```

### 配置详解

#### API Keys 配置

```env
# OpenAI (必需)
OPENAI_API_KEY=sk-...

# 其他可选 API
ANTHROPIC_API_KEY=sk-ant-...
XAI_API_KEY=xai-...
GOOGLE_API_KEY=...

# 社交媒体 API
SCRAPECREATORS_API_KEY=...
REDDIT_CLIENT_ID=...
REDDIT_CLIENT_SECRET=...
YOUTUBE_API_KEY=...
```

#### 模型配置

```python
# 在脚本中自定义模型
model_config = {
    "provider": "openai",
    "model": "gpt-4.5-preview-2025-02-27",
    "temperature": 0.7,
    "max_tokens": 2000,
}
```

---

## 总结与建议

### 项目总结

`last30days-skill` 是一个设计精良、功能强大的 AI 驱动研究工具。它的主要优势包括：

1. **架构优雅**: 模块化设计清晰，易于理解和扩展
2. **功能全面**: 支持 14+ 个数据源，覆盖主流社交平台
3. **技术先进**: 创新性地使用 LLM 进行内容筛选和评分
4. **健壮可靠**: 完善的错误处理和缓存机制
5. **用户友好**: 多种输出格式，灵活的配置选项

### 应用场景

这个项目可以应用于以下场景：

1. **市场研究**: 快速了解某一领域的最新动态
2. **竞争情报**: 监控竞争对手的社交媒体活动
3. **学术研究**: 收集某一主题的多源数据
4. **内容创作**: 为文章和报告收集素材
5. **趋势分析**: 识别新兴趋势和热点话题

### 扩展建议

#### 1. 新增数据源
- 微博、知乎等中文平台
- LinkedIn 等专业平台
- 新闻媒体 API
- 学术文献数据库

#### 2. 增强功能
- 实时监控和告警
- 趋势预测和分析
- 多语言支持
- 可视化仪表板

#### 3. 技术改进
- 添加 REST API 接口
- 支持 Docker 部署
- 实现分布式处理
- 添加用户认证和权限管理

### 学习价值

对于开发者来说，这个项目有很高的学习价值：

1. **架构设计**: 学习如何设计模块化、可扩展的系统
2. **API 集成**: 了解如何集成多个第三方 API
3. **AI 应用**: 学习如何将 LLM 集成到实际应用中
4. **性能优化**: 了解并行处理和缓存技术
5. **最佳实践**: 学习 Python 项目的最佳实践

---

## 附录

### 参考资源

- 项目 GitHub: https://github.com/mvanhorn/last30days-skill
- OpenClaw 官网: https://openclaw.ai
- OpenAI API 文档: https://platform.openai.com/docs
- Python 并发编程: https://docs.python.org/3/library/concurrent.futures.html

### 相关项目

- https://github.com/VoltAgent/awesome-openclaw-skills - OpenClaw 技能合集
- https://github.com/modelcontextprotocol - Model Context Protocol

### 术语表

| 术语 | 说明 |
|------|------|
| LLM | Large Language Model，大语言模型 |
| API | Application Programming Interface，应用程序编程接口 |
| TTL | Time To Live，生存时间 |
| CLI | Command Line Interface，命令行界面 |
| JSON | JavaScript Object Notation，数据交换格式 |

---

**报告完成日期**: 2026年3月28日  
**分析工具**: OpenClaw AI Assistant  
**报告版本**: 1.0
