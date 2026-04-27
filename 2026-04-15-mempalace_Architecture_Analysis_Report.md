# MemPalace 技术架构与源码研读报告

**项目**: [MemPalace/mempalace](https://github.com/MemPalace/mempalace)  
**⭐ Stars**: 43,367（2026-04-05 上线，10天） | **语言**: Python | **License**: MIT  
**分析日期**: 2026-04-15  
**分析人**: OpenClaw 自动架构分析系统

---

## 一、项目概述

MemPalace 是由 Milla Jovovich 和 Ben Sigman 开发的 AI 记忆系统，2026年4月5日刚上线 GitHub，10天内斩获 43,367 ⭐，创下了 AI 记忆系统领域的新纪录。它在 LongMemEval 基准上以 **96.6% R@5** 的成绩创下零 API 依赖方案的最高分记录，且完全本地运行，无需云服务。

### 核心洞察：简单方法战胜复杂工程

MemPalace 的核心发现颠覆了行业共识：所有竞品（Mem0、Mastra、Supermemory）都假设"需要 AI 来决定记住什么"，然后做信息抽取和摘要。但 MemPalace 的基线方案是：**直接存储原始对话文本，用 ChromaDB 向量搜索**，不抽摘要、不提取、不损失信息。96.6% 的成绩证明——**信息保留 > 智能压缩**。

| 方案 | LongMemEval R@5 | 是否需要 LLM | 成本 |
|------|----------------|-------------|------|
| MemPal 原始模式 | **96.6%** | ❌ 无 | $0 |
| MemPal 混合 v4 + Haiku | **100%** | 可选 | ~$0.001/次 |
| Supermemory (生产) | ~85% | 是 | 付费 |
| Mem0 (RAG) | 30–45% | 是 | 付费 |
| BM25 基线 | ~70% | ❌ | $0 |

---

## 二、核心灵感来源：记忆宫殿法（Method of Loci）

MemPalace 的命名直接来自古希腊演说家西塞罗等人使用的"记忆宫殿"技巧：**将想法放置在建筑的各个房间中，行走时按顺序找到它们**。MemPalace 将这一思想映射为数字记忆系统：

| 物理隐喻 | MemPalace 实现 | 作用 |
|---------|--------------|------|
| 建筑 (Palace) | 整个向量数据库 | 全部记忆的顶层空间 |
| 翼 (Wing) | 按人/项目划分的顶层分组 | 隔离不同领域 |
| 大厅 (Hall) | 按内容类型（情感/技术/家庭）分类 | 横向连接 |
| 房间 (Room) | 特定主题 | 核心组织单元 |
| 壁橱 (Closet) | AAAK 格式的压缩索引卡 | 快速定位 |
| 抽屉 (Drawer) | 800字符的原始文本块 | 最小存储单位 |

**核心哲学**：不依赖 AI 决定什么重要，由用户通过结构自主组织记忆。

---

## 三、系统架构总览

```
┌──────────────────────────────────────────────────────────────────┐
│                         用户层                                     │
│  CLI (cli.py) │ MCP Server │ Claude Code Hooks │ Python API      │
└──────────────────────┬───────────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────────┐
│                      4层记忆栈 (layers.py)                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ L0 Identity (~100 tokens)  — 永远加载，用户是谁               │  │
│  │ L1 Essential Story (~500-800 tokens) — 最重要时刻            │  │
│  │ L2 On-Demand (~200-500 each) — 按需加载                      │  │
│  │ L3 Deep Search — 全量 ChromaDB 语义搜索                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────┬───────────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────────┐
│                    存储层                                          │
│                                                                   │
│  ┌──────────────────┐    ┌──────────────────┐   ┌──────────────┐ │
│  │  ChromaDB         │    │  SQLite           │   │ JSON Files  │ │
│  │  (向量数据库)      │    │  (时序知识图谱)    │   │ (显式隧道)   │ │
│  │                   │    │                   │   │              │ │
│  │ mempalace_drawers │    │ knowledge_graph.  │   │ tunnels.json │ │
│  │ mempalace_closets │    │ sqlite3           │   │ identity.txt│ │
│  └──────────────────┘    └──────────────────┘   └──────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## 四、核心模块详解

### 4.1 抽屉系统（Drawer）— miner.py

抽屉是 MemPalace 的最小存储单元。每个抽屉存储 800 字符的原始文本块（可重叠 100 字符），**完全存储原始内容，不做任何摘要**。

**文件挖掘流程**：

```
scan_project()        — 遍历项目，尊重 .gitignore
    ↓
load_config()         — 读取 mempalace.yaml 获取 wing/room 配置
    ↓
process_file()         — 单文件处理管道
    ↓
    ├── chunk_text()      — 按段落边界切分 800 字符块
    ├── detect_room()     — 关键词评分路由到正确 room
    ├── detect_hall()     — 关键词评分判断 Hall（情感/技术/家庭等）
    ├── add_drawer()      — 写入 ChromaDB，带 wing/room/hall 元数据
    ├── build_closet_lines() — 生成 AAAK 格式索引卡
    └── upsert_closet_lines() — 写入壁橱索引
```

**mempalace.yaml 配置示例**：
```yaml
wing: myproject
rooms:
  - name: api
    description: REST API implementation
    keywords: [route, endpoint, handler]
  - name: database
    description: Database models and migrations
    keywords: [model, schema, query]
```

**交叉平台文件锁**：使用 `fcntl.flock`（Linux/macOS）和 `msvcrt.locking`（Windows）防止多 Agent 并发挖掘同一文件导致重复数据。

### 4.2 壁橱系统（Closet）— palace.py

壁橱是 AAAK（Abreviation Dialect）格式的压缩索引卡，定向指向抽屉。

**格式**：`topic|entities|→drawer_ids`

```
build_closet_lines() 输出示例：
"implemented auth|auth;middleware|→abc123,def456"
"added user endpoint|user;api|→ghi789"
"fixed login bug|bug;login|→jkl012"
```

**AAAK 压缩方法**：
- 从前 5000 字符中提取出现 ≥2 次的专有名词
- 提取关键动词短语（built, fixed, created...）
- 提取引用文本
- 打包进 1500 字符的壁橱（超出则另开新壁橱）

**去重机制**：`NORMALIZE_VERSION` 版本号控制规范化升级时自动重建壁橱，无需用户手动操作。

### 4.3 知识图谱（Knowledge Graph）— knowledge_graph.py

MemPalace 维护一个**时序实体关系图谱**，使用 SQLite 本地存储，完整替代 Zep 的云端 Neo4j。

**三元组模型**：
```python
add_triple("Max", "child_of", "Alice", valid_from="2015-04-01")
add_triple("Max", "does", "swimming", valid_from="2025-01-01")
add_triple("Max", "loves", "chess", valid_from="2025-10-01")
```

**关键设计**：
- **时序有效性**：每个三元组有 `valid_from` 和 `valid_to` 字段，知道"某事实何时为真"
- **查询时过滤**：`query_entity("Max", as_of="2026-01-15")` 只返回当时有效的 Facts
- **自动失效**：`invalidate()` 为关系设置 `valid_to` 日期，而非物理删除
- **WAL 模式**：`PRAGMA journal_mode=WAL` 支持并发读写

**表结构**：
```sql
entities (id, name, type, properties, created_at)
triples  (id, subject, predicate, object, valid_from, valid_to, confidence, source_closet, source_file)
```

### 4.4 图遍历层（Palace Graph）— palace_graph.py

将 ChromaDB 元数据构建为可导航的图结构：

**被动隧道**：同一 Room 名称出现在多个 Wing 时，自动发现跨 Wing 连接（如 `auth` room 同时存在于 `wing_api` 和 `wing_security`）

**显式隧道**：Agent 手动创建的跨抽屉/房间链接，存储在 `~/.mempalace/tunnels.json`（原子写入防崩溃）

**BFS 遍历**：从任意房间出发，按跳数扩散寻找关联房间

### 4.5 混合搜索（Hybrid Search）— searcher.py

搜索管道融合两种检索方法，**始终以向量搜索为底，壁橱命中为加分信号**：

```
用户查询
    ↓
1. ChromaDB 向量语义搜索 → Top-N 候选抽屉
    ↓
2. BM25 关键词重排序 → 对候选集内重新计算 BM25 分数
    ↓
3. 壁橱命中加成 → 若某候选被壁橱直接指向，给予排名加成
    ↓
4. 融合得分排序输出
```

**BM25 参数**：k1=1.5（词频饱和），b=0.75（长度归一化），使用 Lucene 公式的 IDF 计算（防负值）。

**壁橱是信号而非门控**：壁橱命中只加分，不屏蔽任何结果——确保弱壁橱（正则提取）永远不能隐藏直接搜索能找到的内容。

### 4.6 4层记忆栈（Memory Layers）— layers.py

```
L0 (~100 tokens): 始终加载
  → ~/.mempalace/identity.txt
  → "我是谁，我的核心特质，重要的人和项目"

L1 (~500-800 tokens): 始终加载
  → 从 Palace 中提取最重要时刻的摘要
  → 提供"我是Atlas，为Alice服务的AI助手"这类身份感

L2 (~200-500 each): 按需加载
  → 某个 wing/room 被提及时触发
  → 加载相关上下文

L3 (unlimited): Deep Search
  → 全量 ChromaDB 语义搜索
  → Wake-up 代价: ~600-900 tokens
```

### 4.7 MCP Server — mcp_server.py

提供完整的 Model Context Protocol 工具集：

| 工具类型 | 工具名 | 功能 |
|---------|--------|------|
| 读取 | `mempalace_status` | 抽屉总数，wing/room 分布 |
| 读取 | `mempalace_list_wings` | 所有 wing 及抽屉数 |
| 读取 | `mempalace_search` | 语义搜索，可选 wing/room 过滤 |
| 读取 | `mempalace_get_taxonomy` | 完整 wing→room→count 树 |
| 写入 | `mempalace_add_drawer` | 写入原始内容到指定位置 |
| 写入 | `mempalace_delete_drawer` | 按 ID 删除抽屉 |
| 图遍历 | `mempalace_traverse` | BFS 遍历 Palace Graph |
| 图遍历 | `mempalace_find_tunnels` | 发现跨 Wing 隧道 |
| 图遍历 | `mempalace_follow_tunnels` | 跟随显式隧道 |

### 4.8 对话挖掘（Convo Mining）— convo_miner.py

专门处理 AI 对话记录的挖掘器：
- 支持 Claude Code、ChatGPT、Slack 等多种格式
- 按 Q&A 对（一次问答=一个抽屉）组织
- 支持 strip_noise 规范化（去除 JSONL 元数据噪音）
- 假设对话记录不可变（mtime 检查关闭）

---

## 五、关键设计哲学

### 5.1 信息保留优先

整个系统的核心赌注是：**不做摘要，永远保留原始文本**。这与所有竞品（Mem0、Mastra）相反——后者用 LLM 提取"重要事实"并丢弃原文。MemPalace 认为提取过程本身就是信息损失，保留原文 + 好的搜索 > 提取 + 搜索。

Benchmark 数据支持这一论点：MemPalace 在 ConvoMem 上比 Mem0 高出 2 倍以上（92.9% vs 30-45%）。

### 5.2 本地优先，无锁定

- 无外部 API 依赖
- 所有数据在 `~/.mempalace/` 本地存储
- ChromaDB + SQLite 替代云端向量数据库
- 原子文件写入（tunnels.json 用 tmp→replace 防崩溃）

### 5.3 诚实披露文化

项目方在 README 中主动承认了 AAAK 宣传数据错误、30x 压缩夸大、+34% palace boost 误导等问题，并附上详细的修正说明。这种透明文化在开源社区赢得了大量信任。

### 5.4 多 Agent 并发安全

- 文件锁防止多 Agent 同时挖掘同一文件
- ChromaDB upsert 前先 delete（避免 hnswlib updatePoint 线程不安全导致 macOS ARM segfault）
- WAL 模式 SQLite 支持并发

---

## 六、技术栈与依赖

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| 向量数据库 | ChromaDB ≥ 0.5.0 | 本地持久化，无需云服务 |
| KG 存储 | SQLite (WAL) | 时序关系图谱，无外部依赖 |
| 配置 | JSON (~/.mempalace/config.json) | env > config > defaults |
| 命名实体 | 本地正则 + known_entities.json | 无NER API依赖 |
| 身份层 | 纯文本 (~/.mempalace/identity.txt) | 用户自述，无AI处理 |
| MCP 协议 | Python stdio server | 与 Claude Code 等 AI 工具集成 |
| CLI | Python click/argparse | `pip install mempalace` 后 `mempalace` 命令 |

**极简依赖**：仅 `chromadb` + `pyyaml` 两个运行时依赖，是 Mem0（12+ 依赖）的零头。

---

## 七、已识别的问题与限制

项目方在 README 中诚实披露了以下问题：

1. **AAAK 模式回归**：AAAK 压缩模式 R@5 为 84.2%，比原始模式低 12.4 个百分点，团队正在迭代改进
2. **ChromaDB 版本迁移 bug**：0.6.x → 1.5.x 有 BLOB seq_id 迁移问题，已在 `chroma.py` 中有修复代码
3. **macOS ARM64 segfault**：Chromadb 0.6.3 在特定条件下会 segfault，通过先删后插而非 upsert 规避
4. **shell 注入风险**：hooks 中存在注入问题（Issue #110），正在修复
5. **事实核查未接入 KG**：fact_checker.py 独立存在，尚未与 KG 操作联通

---

## 八、源码质量评价

**亮点**：
- 模块边界清晰（palace/miner/searcher/kg 各司其职）
- 文档字符串极其详细，包含设计动机
- 缓存策略合理（entity registry mtime 缓存、Hall keywords 缓存）
- 错误处理到位（try/except 包裹所有 IO 操作）

**可改进处**：
- `_ENTITY_REGISTRY_CACHE` 等 module-level 全局状态，在并发场景下可能存在问题
- ChromaDB 版本兼容性处理涉及 SQL 层面修改，属于脆弱层
- 一些边界 case（如超长 wing/room 名）用正则限制，但元数据字段长度限制未全局统一

---

## 九、总结

MemPalace 是一个在工程和哲学层面都很有主见的项目。它的核心洞察"原始文本 > 提取摘要"有扎实的 benchmark 数据支撑，在 AI 记忆系统这个赛道上走出了一条差异化路线。

其架构最值得学习的地方：
1. **存储分层**（Wing→Room→Closet→Drawer）的组织思想比扁平等宽向量检索更符合人类记忆规律
2. **4层唤醒代价**设计让 AI 在极低 token 开销下获得上下文，极其实用
3. **诚实披露文化**让它在上线 48 小时内就获得了社区的深度信任
4. **本地优先**架构让隐私敏感用户零顾虑采用

代码规模约 4000 行，架构干净，适合作为 AI Agent 记忆系统的架构范本参考。

---

*报告生成时间：2026-04-15 03:00 AM (Asia/Shanghai)*  
*分析工具：OpenClaw 自动代码架构分析系统*
*数据来源：GitHub trending 2026-04-05~2026-04-14*
