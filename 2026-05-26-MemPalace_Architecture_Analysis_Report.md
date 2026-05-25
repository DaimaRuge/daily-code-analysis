# 技术架构与源码研读报告 — MemPalace

> **项目**: [MemPalace/mempalace](https://github.com/MemPalace/mempalace)  
> **分析日期**: 2026-05-26  
> **Stars**: ⭐ 52,815 | **Forks**: 🍴 6,968  
> **语言**: Python (100%) | **License**: MIT  
> **Slogan**: *The best-benchmarked open-source AI memory system. And it's free.*

---

## 目录

1. [项目概览](#1-项目概览)
2. [核心架构总览](#2-核心架构总览)
3. [数据模型：宫殿隐喻](#3-数据模型宫殿隐喻)
4. [存储层：可插拔后端设计](#4-存储层可插拔后端设计)
5. [检索层：混合搜索引擎](#5-检索层混合搜索引擎)
6. [知识图谱：时态实体关系图](#6-知识图谱时态实体关系图)
7. [MCP 服务器：AI Agent 接口](#7-mcp-服务器ai-agent-接口)
8. [嵌入模型与硬件加速](#8-嵌入模型与硬件加速)
9. [数据挖掘流水线](#9-数据挖掘流水线)
10. [配置与实体检测系统](#10-配置与实体检测系统)
11. [测试与工程实践](#11-测试与工程实践)
12. [性能基准](#12-性能基准)
13. [架构设计亮点](#13-架构设计亮点)
14. [总结与启示](#14-总结与启示)

---

## 1. 项目概览

MemPalace 是一个**本地优先（Local-first）的 AI 记忆系统**，核心定位是将用户的对话历史和项目文件以**逐字原文（verbatim）**形式存储，并通过语义搜索进行检索。与 Mem0、Zep 等竞品不同，MemPalace 坚持不摘要、不转述、不抽取 —— 原始内容完整保留，检索时才按需召回。

### 1.1 关键特性

| 特性 | 说明 |
|------|------|
| **逐字存储** | 原始文本完整保留，不做 LLM 摘要或改写 |
| **结构化索引** | Wing（翼）→ Room（房间）→ Drawer（抽屉）三级结构 |
| **混合检索** | 向量语义搜索 + BM25 关键词匹配 + 时序增强 |
| **知识图谱** | 本地 SQLite 支撑的时态实体关系图，对标 Zep 的 Neo4j 方案 |
| **MCP 协议** | 29 个 MCP 工具，原生支持 Claude Code、Gemini CLI 等 |
| **零 API Key** | 核心路径无需任何云端 LLM，纯本地 ONNX 嵌入模型 |
| **多语言** | embeddinggemma-300m 支持 100+ 语言 |

### 1.2 仓库规模

- **Python 代码**: ~40 个核心模块，总计约 1.5MB 源码
- **测试覆盖**: 85%+ 代码覆盖率要求，100+ 测试文件
- **文档**: 完整的 CLI 参考、Python API 文档、概念指南
- **插件生态**: Claude Code 插件、Codex 插件、MCP 服务器

---

## 2. 核心架构总览

MemPalace 采用**分层架构**，职责边界清晰，依赖关系单向流动：

```
┌─────────────────────────────────────────────────────────────┐
│                    用户接口层 (CLI / MCP / Hooks)               │
│  cli.py · mcp_server.py · hooks_cli.py · .claude-plugin     │
├─────────────────────────────────────────────────────────────┤
│                    业务逻辑层 (Mining / Search / KG)          │
│  miner.py · convo_miner.py · searcher.py · knowledge_graph.py │
│  palace_graph.py · hallways.py · entity_detector.py         │
├─────────────────────────────────────────────────────────────┤
│                    存储抽象层 (Backend Interface)              │
│  backends/base.py (RFC 001) · backends/chroma.py              │
├─────────────────────────────────────────────────────────────┤
│                    向量数据库层 (ChromaDB / HNSW)                │
│  chromadb · SQLite · HNSW 索引 · FTS5 全文索引                  │
├─────────────────────────────────────────────────────────────┤
│                    嵌入模型层 (ONNX Runtime)                   │
│  embedding.py · MiniLM / embeddinggemma-300m · CUDA/CoreML/DML  │
└─────────────────────────────────────────────────────────────┘
```

### 2.1 关键模块职责

| 模块 | 行数 | 核心职责 |
|------|------|----------|
| `palace.py` | ~900 | 集合访问、抽屉管理、衣橱（closet）构建、健康状态检测 |
| `miner.py` | ~1800 | 文件扫描、分块、去重、实体检测、批量写入 |
| `searcher.py` | ~1000 | 混合搜索（向量+BM25）、重排序、候选合并 |
| `mcp_server.py` | ~2900 | MCP JSON-RPC 服务器、29 个工具实现、stdio 保护 |
| `knowledge_graph.py` | ~600 | SQLite 时态三元组图、CRUD、时间过滤查询 |
| `palace_graph.py` | ~900 | 宫殿图遍历、跨翼隧道（tunnel）、图缓存 |
| `hallways.py` | ~300 | 翼内实体共现走廊、JSON 持久化 |
| `entity_detector.py` | ~900 | 两阶段实体检测、COCA 过滤、已知系统词典 |
| `embedding.py` | ~300 | ONNX 模型工厂、硬件加速自动检测 |
| `config.py` | ~700 | 配置系统、输入消毒、环境变量优先级 |
| `cli.py` | ~2000 | 完整 CLI 命令集、进度显示、错误处理 |

---

## 3. 数据模型：宫殿隐喻

MemPalace 的数据模型采用**建筑空间隐喻**，让复杂的数据结构直观可理解：

```
Palace（宫殿）
├── Wing（翼）          —— 项目/人员/概念的大类分区
│   ├── Room（房间）    —— 主题/功能子分区
│   │   ├── Drawer（抽屉）—— 逐字存储的原文块
│   │   └── ...
│   └── Hallway（走廊） —— 实体间的共现关系
├── Closet（衣橱）      —— 可搜索的摘要索引层
├── Tunnel（隧道）      —— 跨翼的连接通道
└── Knowledge Graph     —— 时态实体关系图
```

### 3.1 核心数据结构

**Drawer（抽屉）**
- 存储单元：逐字文本块（chunk），默认 800 字符
- 元数据：wing、room、source_path、line_start、line_end、hash、timestamp
- 向量：384 维嵌入（MiniLM 或 embeddinggemma）

**Closet（衣橱）**
- 排名信号层：从抽屉中提取的 topic + entity 指针
- 格式：`"topic|entities|→drawer_id_a,drawer_id_b"`
- 作用：为搜索提供关键词增强信号，而非独立结果源

**Wing（翼）**
- 由 `mempalace init` 自动创建，映射到项目目录
- 通过 `mempalace.yaml` 配置 wing → rooms 的映射关系
- 实体检测在初始化阶段识别 wing 中的 people / projects

---

## 4. 存储层：可插拔后端设计

### 4.1 RFC 001：后端接口契约

`mempalace/backends/base.py` 定义了完整的后端抽象：

```python
class BaseBackend(ABC):
    @abstractmethod
    def get_collection(self, palace_path, collection_name, create=True) -> BaseCollection

class BaseCollection(ABC):
    @abstractmethod
    def add(self, ids, documents, embeddings, metadatas) -> None
    @abstractmethod
    def query(self, query_embeddings, n_results, where, where_document) -> QueryResult
    @abstractmethod
    def get(self, ids, where, include) -> GetResult
```

**设计理念**：
- 全 kwargs-only 接口，避免位置参数错误
- 强类型返回值：`QueryResult` / `GetResult` dataclass 替代 Chroma 的字典
- 错误层级清晰：`PalaceNotFoundError` → `CollectionNotInitializedError` → `BackendClosedError`

### 4.2 ChromaDB 参考实现

`backends/chroma.py` 是 RFC 001 的首个实现，包含多项生产级加固：

| 加固项 | 说明 |
|--------|------|
| **HNSW 健康检测** | `link_lists.bin / data_level0.bin` 比例 >10x 时标记为损坏 |
| **线程钉扎** | `_pin_hnsw_threads` 限制 ONNX 线程，避免与 HNSW 的 OpenMP 冲突 |
| **容量守卫** | `hnsw_capacity_status` 检测向量段膨胀 |
| **元数据验证** | 写入前验证嵌入维度、模型名称一致性 |

### 4.3 持久化架构

```
~/.mempalace/
├── config.json              —— 用户配置（模型选择、设备、路径）
├── knowledge_graph.sqlite3  —— 时态知识图谱
├── hallways.json            —— 翼内实体共现走廊
├── tunnels.json             —— 跨翼隧道连接
└── <palace_path>/
    ├── chroma.sqlite3       —— ChromaDB 主数据库
    └── <uuid>/              —— HNSW 向量段 + 全文索引段
```

---

## 5. 检索层：混合搜索引擎

### 5.1 三层检索架构

```
用户查询
    │
    ├─→ [Layer 1] 向量语义检索（ChromaDB HNSW）
    │      └── 召回 top-k 候选抽屉
    │
    ├─→ [Layer 2] BM25 关键词检索（FTS5 / 候选内重排序）
    │      └── Okapi-BM25 + 候选内 IDF
    │
    └─→ [Layer 3] 混合融合（凸组合）
           └── score = 0.6 × vector_sim + 0.4 × BM25
           └── 衣橱（closet）信号增强
```

### 5.2 BM25 实现细节

`_bm25_scores()` 函数实现了完整的 Okapi-BM25：

```python
def _bm25_scores(query, documents, k1=1.5, b=0.75):
    # IDF 在候选集内计算 —— 反映查询词在候选中的区分度
    idf = {term: math.log((N - df + 0.5) / (df + 0.5) + 1)
           for term in query_terms}
    # 长度归一化：avgdl 为候选集平均文档长度
    score = sum(idf[t] * tf * (k1+1) / (tf + k1*(1-b+b*dl/avgdl))
              for t, tf in term_freqs.items())
```

**关键设计决策**：
- IDF 在**候选集内**计算，而非全局语料库 —— 适合重排序场景
- 绝对余弦相似度（非相对归一化）—— 添加/删除候选不会影响其他结果的分数
- 候选合并模式：向量未命中的候选可由 BM25 单独贡献分数

### 5.3 衣橱（Closet）信号机制

Closet 是 MemPalace 检索的独特创新：

```
抽屉层（Drawers）: 原文块 —— 检索的 floor（底线）
衣橱层（Closets）: topic|entities|→drawer_ids —— 排名 signal（信号）
```

- Closet 从抽屉中提取实体/主题，构建轻量索引
- 搜索时，Closet 命中为对应抽屉提供**排名 boost**
- Closet 永远是**信号而非 gate** —— 弱 Closet 不会隐藏强抽屉

---

## 6. 知识图谱：时态实体关系图

### 6.1 架构对比

| 维度 | Zep | MemPalace |
|------|-----|-----------|
| 数据库 | Neo4j（云端，$25+/月） | SQLite（本地，免费） |
| 时态支持 | 有 | 有（valid_from / valid_to） |
| 关系类型 | 预定义 | 开放（任意谓词） |
|  Closet 链接 | 无 | 有（指向原始记忆） |

### 6.2 核心 API

```python
kg = KnowledgeGraph()

# 添加带有效期的三元组
kg.add_triple("Max", "child_of", "Alice", valid_from="2015-04-01")
kg.add_triple("Max", "does", "swimming", valid_from="2025-01-01")

# 时间点查询：2026年1月 Max 的状态
kg.query_entity("Max", as_of="2026-01-15")

# 关系失效：伤病恢复
kg.invalidate("Max", "has_issue", "sports_injury", ended="2026-02-15")
```

### 6.3 时态查询引擎

SQLite 中使用 CASE 表达式处理日期/时间戳的统一比较：

```sql
CASE WHEN length(valid_from) = 10 
     AND substr(valid_from, 5, 1) = '-'
     THEN valid_from || 'T00:00:00Z' 
     ELSE valid_from END
```

- 日期-only（`2026-05-26`）→ 扩展为 `2026-05-26T00:00:00Z`
- 完整时间戳 → 原样使用
- 字典序比较即可实现时态范围查询

---

## 7. MCP 服务器：AI Agent 接口

### 7.1 架构位置

`mcp_server.py`（~2900 行）是 MemPalace 的**外部接口层**，通过 MCP（Model Context Protocol）向 Claude Code、Gemini CLI 等工具暴露 29 个操作：

```
Claude Code / Gemini CLI / 任意 MCP 客户端
              │
              ▼
        ┌─────────────┐
        │  stdio/json-rpc  │  ← _stdio.py 保护 stdout
        └─────────────┘
              │
        ┌─────────────┐
        │  mcp_server.py   │  ← 29 个 tool handler
        └─────────────┘
              │
        ┌─────────────┐
        │  palace.py /      │
        │  searcher.py /    │
        │  knowledge_graph.py│
        └─────────────┘
```

### 7.2 stdio 保护机制

MCP 协议要求 stdout **仅**输出有效 JSON-RPC。但 ChromaDB → ONNX Runtime → posthog 的依赖链会在 C 层打印横幅/错误到 stdout，导致 Claude Desktop 解析崩溃。

**解决方案**（issue #225）：
```python
# 在导入任何重型依赖前，将 stdout 重定向到 stderr
os.dup2(2, 1)  # fd 级别
sys.stdout = sys.stderr  # Python 级别

# 进入协议循环前恢复真实 stdout
sys.stdout = _REAL_STDOUT
```

### 7.3 工具分类

| 类别 | 工具示例 | 作用 |
|------|----------|------|
| 读操作 | `mempalace_status`, `mempalace_search` | 查询宫殿状态和内容 |
| 写操作 | `mempalace_add_drawer`, `mempalace_delete_drawer` | 修改存储 |
| 图谱操作 | `mempalace_kg_add`, `mempalace_kg_query` | 知识图谱 CRUD |
| 维护 | `mempalace_reconnect` | 缓存失效与重连 |
| 导航 | `mempalace_list_wings`, `mempalace_list_rooms` | 结构浏览 |

---

## 8. 嵌入模型与硬件加速

### 8.1 模型选择

| 模型 | 维度 | 语言 | 大小 | 适用场景 |
|------|------|------|------|----------|
| `all-MiniLM-L6-v2` | 384 | 英语 | ~30MB | 存量宫殿、纯英文 |
| `embeddinggemma-300m` | 384 (Matryoshka) | 100+ | ~300MB | 新安装推荐、多语言 |

**Matryoshka 截断**：embeddinggemma 原生更高维度，通过截断到 384 维保持与 MiniLM 的向量空间兼容。

### 8.2 硬件加速自动检测

```python
_PROVIDER_MAP = {
    "cpu": ["CPUExecutionProvider"],
    "cuda": ["CUDAExecutionProvider", "CPUExecutionProvider"],
    "coreml": ["CoreMLExecutionProvider", "CPUExecutionProvider"],
    "dml": ["DmlExecutionProvider", "CPUExecutionProvider"],
}

_AUTO_ORDER = [
    ("CUDAExecutionProvider", "cuda"),
    ("CoreMLExecutionProvider", "coreml"),
    ("DmlExecutionProvider", "dml"),
]
```

- `auto` 模式按优先级探测可用加速器
- 请求不可用的加速器 → 警告 + 回退 CPU，**不崩溃**
- 懒加载：ONNX 模型在首次使用时下载，非安装时

---

## 9. 数据挖掘流水线

### 9.1 文件扫描与分块

```
项目目录
  │
  ├─→ 扩展名过滤（.py, .md, .ts, .json, ...）
  ├─→ 跳过目录（.git, node_modules, __pycache__, ...）
  ├─→ 跳过文件名（package-lock.json, yarn.lock, ...）
  ├─→ 文件大小上限（500MB）
  │
  └─→ 内容读取 → 文本规范化 → 分块（chunk）
       │
       ├─→ 块大小：800 字符（默认）
       ├─→ 重叠：100 字符
       ├─→ 最小块：200 字符
       └─→ 每文件上限：50,000 块
```

### 9.2 两阶段实体检测

`entity_detector.py` 实现了语言学级别的实体检测：

**Pass 1：信号提取**
- 扫描文件内容，提取大写候选词
- 应用 COCA（Corpus of Contemporary American English）内容词过滤 —— 排除 "Code"、"Note"、"Line" 等常见大写假阳性
- 应用已知系统词典（known_systems.json）—— 保护 "Claude Code"、"GitHub Actions" 等复合名词

**Pass 2：分类确认**
- 基于信号统计（人称代词、对话标记、项目动词）分类为 person / project / uncertain
- 多语言支持：模式定义在 `mempalace/i18n/<lang>.json` 中

### 9.3 写入流程

```
分块 → 嵌入（ONNX 批处理，1000 块/批次）
      → 去重（内容哈希）
      → 抽屉写入 ChromaDB
      → Closet 构建（实体/topic 提取）
      → 全文索引（FTS5）
      → 知识图谱更新（可选）
      → 走廊计算（实体共现）
```

---

## 10. 配置与实体检测系统

### 10.1 配置优先级

```
环境变量（MEMPALACE_*）> 配置文件（~/.mempalace/config.json）> 代码默认值
```

### 10.2 输入消毒体系

MemPalace 对不可信输入实施多层防御：

| 函数 | 防御目标 |
|------|----------|
| `sanitize_name()` | 路径遍历（`..`、`/`）、空字节、超长字符串、非法字符 |
| `sanitize_kg_value()` | 知识图谱实体值（更宽松，允许标点） |
| `sanitize_content()` | 内容体注入 |
| `strip_lone_surrogates()` | UTF-16 孤立代理（issue #1235） |
| `sanitize_query()` | 查询注入、特殊字符逃逸 |

### 10.3 错误处理哲学

- **状态区分**：`PalaceNotFoundError` vs `CollectionNotInitializedError` —— 给用户可操作的错误信息
- **优雅降级**：ONNX 加速器不可用 → CPU 回退 + 警告
- **批量容错**：单个文件错误不中断整个挖掘流程

---

## 11. 测试与工程实践

### 11.1 测试矩阵

```
tests/
├── test_backends.py           —— 后端抽象层
├── test_chroma_collection_lock.py —— 并发锁
├── test_cli.py                —— CLI 命令
├── test_closets.py            —— 衣橱索引
├── test_convo_miner.py        —— 对话挖掘
├── test_dedup.py              —— 去重逻辑
├── test_embedding.py          —— 嵌入模型
├── test_entity_detector.py    —— 实体检测
├── test_hallways.py           —— 走廊系统
├── test_hooks_cli.py          —— 钩子 CLI
├── test_knowledge_graph.py    —— 知识图谱
├── test_mcp_server.py         —— MCP 服务器
├── test_miner.py              —— 核心挖掘
├── test_palace_graph.py       —— 图遍历
├── test_searcher.py           —— 搜索逻辑
├── test_sync.py               —— 同步机制
└── ... (100+ 测试文件)
```

### 11.2 质量门禁

| 工具 | 用途 |
|------|------|
| **pytest** | 单元测试 + 覆盖率（>85%） |
| **ruff** | 代码风格（line-length=100） |
| **mypy** | 静态类型检查 |
| **hypothesis** | 属性驱动测试 |
| **pre-commit** | 提交前自动检查 |

### 11.3 工程细节

- **锁机制**：`mine_lock`（线程锁）+ `mine_palace_lock`（文件锁）防止并发挖掘冲突
- **幂等性**：同一文件重复挖掘 → 检测内容哈希 → 跳过或更新
- **恢复安全**：`mempalace sweep` 支持中断后恢复
- **模式版本**：`NORMALIZE_VERSION` 控制规范化流水线版本，自动重建旧数据

---

## 12. 性能基准

### 12.1 LongMemEval（500 题，检索召回 R@5）

| 模式 | R@5 | LLM 需求 |
|------|-----|----------|
| Raw（纯语义搜索） | **96.6%** | 无 |
| Hybrid v4（50 题调参，450 题 held-out） | **98.4%** | 无 |
| Hybrid v4 + LLM 重排序 | ≥99% | 任意可用模型 |

### 12.2 其他基准

| 基准 | 指标 | 分数 |
|------|------|------|
| LoCoMo (session, top-10) | R@10 | 60.3% → 88.9% (hybrid v5) |
| ConvoMem (250 items, 5 类) | 平均召回 | 92.9% |
| MemBench (ACL 2025, 8.5K) | R@5 | 80.3% |

### 12.3 竞品对比声明

MemPalace 刻意**不**与 Mem0、Mastra、Hindsight、Supermemory、Zep 做并列表格对比 —— 因为各项目发布的指标在不同拆分上，直接对比不诚实。这种学术诚信态度值得赞赏。

---

## 13. 架构设计亮点

### 13.1 🏛️ 空间隐喻建模

Wing → Room → Drawer → Closet 的层级不是随意的命名游戏，而是**精确映射了人类的空间记忆方式**。这种隐喻让 API 直观（`list_wings`、`list_rooms`），也让用户的心理模型与数据结构同构。

### 13.2 🔌 可插拔后端（RFC 001）

通过 `BaseBackend` / `BaseCollection` 抽象，MemPalace 理论上可支持：
- ChromaDB（已实现）
- Milvus、Pinecone、Weaviate（未来）
- 纯 SQLite（轻量场景）

入口点注册制：`[project.entry-points."mempalace.backends"]` 允许第三方包扩展后端。

### 13.3 🧠 检索的哲学：Closet 是信号，不是门

> "Closets are a ranking signal, never a gate."

这一设计原则确保了弱提取的 Closet 不会隐藏强向量匹配的 Drawer —— 召回优先，重排序优化。

### 13.4 ⏱️ 时态知识图谱

用 SQLite 而非 Neo4j 实现时态三元组，是**架构克制**的典范。通过字典序时间戳和 CASE 表达式，实现了完整的时态查询，零额外依赖，零订阅费用。

### 13.5 🛡️ 防御性编程

- stdio 重定向保护（C 层 stdout 污染）
- HNSW 文件比例检测（防止 segfault）
- 孤立 UTF-16 代理处理（`strip_lone_surrogates`）
- 负值环境变量回退（防 typo 导致崩溃）

### 13.6 🌍 多语言原生设计

- 嵌入模型：embeddinggemma-300m 支持 100+ 语言
- 实体检测：i18n JSON 文件驱动，添加新语言无需改代码
- 跨语言 cos 相似度：0.88（embeddinggemma） vs 0.35（MiniLM）

---

## 14. 总结与启示

### 14.1 MemPalace 的定位

MemPalace 不是又一个向量数据库封装，而是一个**完整的 AI 记忆基础设施**：
- 从文件/对话**挖掘**（mine）→ **存储**（store）→ **索引**（index）→ **检索**（search）→ **图谱**（graph）
- 全链路本地优先，零云端依赖
- 为 AI Agent 提供**长期记忆上下文**（通过 MCP 协议）

### 14.2 可借鉴的设计模式

| 模式 | 应用场景 |
|------|----------|
| 隐喻驱动 API 设计 | 任何需要用户建立心理模型的系统 |
| 信号-门分离原则 | 混合检索系统的安全设计 |
| 懒加载 + 回退降级 | 重型模型/资源的初始化 |
| 时态数据字典序存储 | SQLite 中的时间范围查询 |
| stdio 隔离层 | 任何 json-rpc over stdio 的协议 |
| 幂等写入 + 版本化重建 | 数据流水线的可靠性保证 |

### 14.3 局限性

1. **ChromaDB 锁定**：虽然后端抽象存在，但实际生态仍围绕 ChromaDB 构建
2. **Python 单线程限制**：大量锁机制暗示并发场景有瓶颈
3. **无分布式方案**：本地优先也意味着难以横向扩展
4. **FTS5 依赖**：全文搜索绑定 SQLite 版本

### 14.4 最终评价

MemPalace 以 52K+ stars 成为 2026 年最热门的开源 AI 记忆系统，其成功不仅来自技术指标（96.6% raw R@5），更来自**产品哲学的清晰**：

> **"逐字原文、本地优先、零 API Key、结构化可导航。"**

在 AI 应用从「玩具」走向「生产工具」的拐点，一个可靠、可解释、用户完全可控的记忆层，是 Agent 系统不可或缺的基石。MemPalace 用扎实的工程、优雅的架构和诚实的基准，为这一领域树立了标杆。

---

> **报告生成**: 2026-05-26 03:00 CST  
> **分析工具**: Source code inspection + GitHub API + Static analysis  
> **项目版本**: v3.3.6  
> **Commit**: latest (shallow clone)

---

*本报告为每日代码架构分析系列的一部分，聚焦开源项目深度研读与技术洞察。*
