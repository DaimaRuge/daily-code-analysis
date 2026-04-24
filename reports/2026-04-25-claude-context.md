# 技术架构与源码研读报告

## claude-context: 语义代码搜索 MCP 插件深度解析

> 项目地址：https://github.com/zilliztech/claude-context  
> 分析日期：2026-04-25  
> 星标数：8,916 ⭐ | 语言：TypeScript | 许可证：MIT

---

## 一、项目概述

**claude-context** 是由 Zilliz（Milvus 向量数据库背后的公司）开发的开源项目，它是一个基于 **Model Context Protocol (MCP)** 的插件，为 Claude Code 和其他 AI 编程助手提供**语义代码搜索**能力。

### 核心价值主张

| 特性 | 说明 |
|------|------|
| 🧠 全代码库即上下文 | 通过语义搜索从数百万行代码中找到所有相关代码，无需多轮发现 |
| 💰 大代码库成本优化 | 将代码库存储在向量数据库中，仅使用相关代码作为上下文，降低 API 调用成本 |
| 🔍 混合搜索 (Hybrid Search) | 结合稠密向量语义搜索 + 稀疏向量 BM25 关键词搜索，提升召回率 |
| 🌳 AST 感知分块 | 基于 Tree-sitter 语法树进行智能代码分块，保留语义完整性 |

---

## 二、系统架构总览

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        客户端层 (Client Layer)                    │
├─────────────┬─────────────┬─────────────┬───────────────────────┤
│  Claude Code │  Cursor     │  VS Code    │   Chrome Extension    │
│  (CLI)      │  (IDE)      │  (Editor)   │   (Browser)           │
└──────┬──────┴──────┬──────┴──────┬──────┴───────────┬───────────┘
       │             │             │                  │
       └─────────────┴──────┬──────┴──────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │   MCP Protocol Layer       │
              │   (@zilliz/claude-context- │
              │        mcp)                │
              └─────────────┬─────────────┘
                            │
              ┌─────────────▼─────────────┐
              │    Core Engine Layer       │
              │  (@zilliz/claude-context-  │
              │        core)               │
              │  ┌─────────────────────┐  │
              │  │   Context Controller │  │
              │  └─────────────────────┘  │
              │  ┌──────────┐ ┌─────────┐ │
              │  │  Splitter │ │Embedding│ │
              │  │  (AST/   │ │(Multi-  │ │
              │  │LangChain)│ │Provider)│ │
              │  └──────────┘ └─────────┘ │
              │  ┌─────────────────────┐  │
              │  │   Vector Database    │  │
              │  │   (Milvus/Zilliz)    │  │
              │  └─────────────────────┘  │
              └───────────────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │    Infrastructure Layer    │
              │  ┌──────────┐ ┌─────────┐ │
              │  │  Zilliz  │ │  OpenAI │ │
              │  │  Cloud   │ │  (Embed)│ │
              │  └──────────┘ └─────────┘ │
              └───────────────────────────┘
```

### 2.2 模块划分

项目采用 **Monorepo** 架构，使用 pnpm workspace 管理：

```
claude-context/
├── packages/
│   ├── core/                    # 核心引擎包
│   │   ├── src/
│   │   │   ├── context.ts       # 主控制器
│   │   │   ├── splitter/        # 代码分块策略
│   │   │   │   ├── ast-splitter.ts      # AST 语法树分块
│   │   │   │   └── langchain-splitter.ts # 字符级分块(降级)
│   │   │   ├── embedding/       # 嵌入模型适配器
│   │   │   │   ├── openai-embedding.ts
│   │   │   │   ├── gemini-embedding.ts
│   │   │   │   ├── ollama-embedding.ts
│   │   │   │   └── voyageai-embedding.ts
│   │   │   ├── vectordb/        # 向量数据库抽象
│   │   │   │   ├── milvus-vectordb.ts       # gRPC 协议
│   │   │   │   └── milvus-restful-vectordb.ts # REST 协议
│   │   │   └── sync/            # 增量同步
│   │   │       ├── synchronizer.ts
│   │   │       └── merkle.ts    # Merkle 树变更检测
│   │
│   ├── mcp/                     # MCP 协议适配层
│   │   ├── src/
│   │   │   ├── index.ts         # MCP Server 入口
│   │   │   ├── handlers.ts      # 工具处理器
│   │   │   ├── config.ts        # 配置管理
│   │   │   ├── embedding.ts     # 嵌入实例工厂
│   │   │   ├── snapshot.ts      # 索引状态快照
│   │   │   └── sync.ts          # 后台同步管理
│   │
│   ├── vscode-extension/        # VS Code 插件
│   │   ├── src/
│   │   │   ├── extension.ts     # 插件入口
│   │   │   ├── commands/        # 命令实现
│   │   │   └── webview/         # 搜索界面
│   │
│   └── chrome-extension/        # Chrome 浏览器插件
│       └── src/
│           ├── background.ts    # 后台服务
│           └── content.ts       # 内容脚本
│
├── examples/                    # 使用示例
└── evaluation/                  # 效果评估数据
```

---

## 三、核心模块深度解析

### 3.1 Context 控制器 (`context.ts`)

Context 类是整个系统的**中央控制器**，采用**依赖注入**模式管理核心组件：

```typescript
class Context {
  private embedding: Embedding;        // 嵌入模型
  private vectorDatabase: VectorDatabase;  // 向量数据库
  private codeSplitter: Splitter;      // 代码分块器
  private synchronizers: Map<string, FileSynchronizer>; // 同步器缓存
  
  constructor(config: ContextConfig = {}) {
    // 默认使用 OpenAI Embedding + AST 分块
    this.embedding = config.embedding || new OpenAIEmbedding({...});
    this.vectorDatabase = config.vectorDatabase; // 必须提供
    this.codeSplitter = config.codeSplitter || new AstCodeSplitter(2500, 300);
  }
}
```

#### 核心流程：代码库索引

```
indexCodebase(codebasePath)
    │
    ├── 1. loadIgnorePatterns()      # 加载 .gitignore 等忽略规则
    ├── 2. prepareCollection()       # 创建/准备 Milvus Collection
    │   └── detectDimension()        # 自动检测嵌入维度
    ├── 3. getCodeFiles()            # 递归扫描代码文件
    ├── 4. processFileList()         # 流式处理文件
    │   ├── split()                  # AST 分块 → CodeChunk[]
    │   ├── embedBatch()             # 批量生成嵌入向量
    │   └── insert/insertHybrid()    # 写入向量数据库
    └── 5. 返回索引统计
```

#### 混合搜索实现

```typescript
async semanticSearch(codebasePath, query, topK = 5): Promise<SemanticSearchResult[]> {
  // 1. 生成查询向量
  const queryEmbedding = await this.embedding.embed(query);
  
  // 2. 构造混合搜索请求
  const searchRequests: HybridSearchRequest[] = [
    {  // 稠密向量搜索 (语义相似度)
      data: queryEmbedding.vector,
      anns_field: "vector",
      param: { nprobe: 10 },
      limit: topK
    },
    {  // 稀疏向量搜索 (BM25 关键词匹配)
      data: query,
      anns_field: "sparse_vector",
      param: { drop_ratio_search: 0.2 },
      limit: topK
    }
  ];
  
  // 3. RRF (Reciprocal Rank Fusion) 重排序
  const results = await this.vectorDatabase.hybridSearch(
    collectionName, searchRequests, {
      rerank: { strategy: 'rrf', params: { k: 100 } },
      limit: topK
    }
  );
  
  return results;
}
```

### 3.2 AST 代码分块器 (`ast-splitter.ts`)

这是项目的**技术创新点**之一，利用 Tree-sitter 语法树实现**语义感知的代码分块**：

#### 支持的语言

| 语言 | Parser | 可分割节点类型 |
|------|--------|---------------|
| JavaScript/TypeScript | tree-sitter-javascript/typescript | function, class, method, interface, type_alias |
| Python | tree-sitter-python | function, class, decorated_definition |
| Java | tree-sitter-java | method, class, interface, constructor |
| C/C++ | tree-sitter-cpp | function, class, namespace |
| Go | tree-sitter-go | function, method, type, var, const |
| Rust | tree-sitter-rust | function, impl, struct, enum, trait |
| C# | tree-sitter-c-sharp | method, class, interface, struct |
| Scala | tree-sitter-scala | method, class, interface |

#### 分块策略

```typescript
class AstCodeSplitter implements Splitter {
  private chunkSize = 2500;    // 目标块大小 (字符)
  private chunkOverlap = 300;  // 块间重叠 (保持上下文连贯)
  
  async split(code, language, filePath): Promise<CodeChunk[]> {
    // 1. 检查语言是否支持 AST 解析
    const langConfig = this.getLanguageConfig(language);
    if (!langConfig) {
      // 降级到 LangChain 字符级分块
      return this.langchainFallback.split(code, language, filePath);
    }
    
    // 2. 解析 AST
    this.parser.setLanguage(langConfig.parser);
    const tree = this.parser.parse(code);
    
    // 3. 提取可分割节点
    const chunks = this.extractChunks(
      tree.rootNode, code, 
      langConfig.nodeTypes,  // 如 ['function_declaration', 'class_declaration']
      language, filePath
    );
    
    // 4. 细化大块 (如果超过 chunkSize)
    return this.refineChunks(chunks, code);
  }
}
```

#### 为什么 AST 分块更好？

| 分块方式 | 优点 | 缺点 |
|---------|------|------|
| **AST 分块** | 按函数/类边界分割，保留语义完整性 | 需要语言特定的 parser |
| **字符分块** (LangChain) | 通用，支持所有语言 | 可能切断函数/类，破坏语义 |

### 3.3 向量数据库抽象层 (`vectordb/`)

项目使用 **Milvus** 作为向量数据库，并设计了清晰的抽象接口：

```typescript
// 抽象接口
interface VectorDatabase {
  createCollection(name, dimension, description): Promise<void>;
  createHybridCollection(name, dimension, description): Promise<void>;  // 混合搜索
  insert(collection, documents): Promise<void>;
  insertHybrid(collection, documents): Promise<void>;
  search(collection, vector, options): Promise<VectorSearchResult[]>;
  hybridSearch(collection, requests, options): Promise<HybridSearchResult[]>;
  hasCollection(name): Promise<boolean>;
  dropCollection(name): Promise<void>;
}
```

#### Milvus 集合 Schema 设计

**稠密向量集合 (语义搜索)**：
```
Collection: code_chunks_<hash>
├── id: VARCHAR (主键)
├── vector: FLOAT_VECTOR[1536]  (OpenAI embedding)
├── content: VARCHAR (代码内容)
├── relativePath: VARCHAR (相对路径)
├── startLine: INT64
├── endLine: INT64
├── fileExtension: VARCHAR
└── metadata: JSON
```

**混合搜索集合 (稠密+稀疏)**：
```
Collection: hybrid_code_chunks_<hash>
├── id: VARCHAR
├── vector: FLOAT_VECTOR[1536]       (稠密向量 - 语义)
├── sparse_vector: SPARSE_FLOAT_VECTOR (稀疏向量 - BM25)
├── content: VARCHAR                  (用于 BM25 建索引)
├── relativePath: VARCHAR
├── startLine: INT64
├── endLine: INT64
├── fileExtension: VARCHAR
└── metadata: JSON
```

### 3.4 MCP Server 实现 (`mcp/index.ts`)

MCP (Model Context Protocol) 是 Anthropic 推出的开放协议，用于标准化 AI 工具集成。

#### 关键设计：stdout 协议隔离

```typescript
// CRITICAL: 将 console 输出重定向到 stderr
// 只有 MCP 协议消息应该走 stdout
const originalConsoleLog = console.log;
console.log = (...args) => {
  process.stderr.write('[LOG] ' + args.join(' ') + '\n');
};
```

#### 暴露的 Tools

| Tool | 功能 | 参数 |
|------|------|------|
| `index_codebase` | 索引代码库 | path, force, splitter, customExtensions, ignorePatterns |
| `search_code` | 语义搜索 | path, query, limit, extensionFilter |
| `clear_index` | 清除索引 | path |
| `get_indexing_status` | 获取索引状态 | path |

#### Tool Handlers 架构

```typescript
class ToolHandlers {
  private context: Context;           // 核心引擎
  private snapshotManager: SnapshotManager;  // 状态快照
  
  async handleIndexCodebase(args) {
    // 1. 校验路径
    const absPath = ensureAbsolutePath(args.path);
    
    // 2. 执行索引
    const result = await this.context.indexCodebase(absPath, progressCallback, args.force);
    
    // 3. 保存快照 (用于增量同步)
    this.snapshotManager.setCodebaseInfo(absPath, {
      indexedFiles: result.indexedFiles,
      totalChunks: result.totalChunks,
      status: 'indexed'
    });
    
    return result;
  }
  
  async handleSearchCode(args) {
    // 1. 检查是否已索引
    if (!await this.context.hasIndex(args.path)) {
      throw new Error('Codebase not indexed. Please run index_codebase first.');
    }
    
    // 2. 执行搜索
    const results = await this.context.semanticSearch(
      args.path, args.query, args.limit
    );
    
    // 3. 格式化返回
    return results.map(r => ({
      content: truncateContent(r.content),
      path: r.relativePath,
      lines: `${r.startLine}-${r.endLine}`,
      score: r.score
    }));
  }
}
```

### 3.5 增量同步机制 (`sync/`)

项目实现了**智能增量同步**，避免每次全量重建索引：

#### Merkle 树变更检测

```typescript
class FileSynchronizer {
  private merkleTree: MerkleTree;
  
  async checkForChanges(): Promise<{
    added: string[],      // 新增文件
    removed: string[],    // 删除文件
    modified: string[]    // 修改文件
  }> {
    // 1. 构建当前文件系统 Merkle 树
    const currentTree = await this.buildMerkleTree();
    
    // 2. 与上次快照对比
    const diff = this.merkleTree.compare(currentTree);
    
    // 3. 分类变更
    return {
      added: diff.added,
      removed: diff.removed,
      modified: diff.modified
    };
  }
}
```

#### 同步流程

```
reindexByChange(codebasePath)
    │
    ├── checkForChanges()          # Merkle 树对比
    ├── deleteFileChunks()         # 删除 removed/modified 文件的向量
    └── processFileList()          # 重新索引 added/modified 文件
```

---

## 四、设计模式与工程实践

### 4.1 设计模式

| 模式 | 应用位置 | 说明 |
|------|---------|------|
| **策略模式** | Splitter | AstCodeSplitter / LangChainCodeSplitter 可互换 |
| **工厂模式** | Embedding | createEmbeddingInstance() 根据配置创建对应嵌入器 |
| **适配器模式** | VectorDatabase | MilvusVectorDatabase 封装底层 SDK |
| **观察者模式** | Progress Callback | 索引进度回调 |
| **单例模式** | EnvManager | 环境变量管理器 |

### 4.2 错误处理策略

```typescript
// AST 分块失败时优雅降级
try {
  return await this.astSplitter.split(code, language);
} catch (error) {
  console.warn(`AST failed, falling back to LangChain: ${error}`);
  return await this.langchainFallback.split(code, language);
}
```

### 4.3 性能优化

| 优化点 | 实现方式 |
|--------|---------|
| 批量嵌入 | `embedBatch()` 减少 API 调用次数 |
| 流式处理 | `processFileList()` 边读边处理，避免内存峰值 |
| 批量插入 | 向量数据库批量写入 |
| 懒加载 | MilvusClient 异步初始化 |
| 集合预加载 | `ensureLoaded()` 搜索前自动 load collection |

---

## 五、关键代码片段赏析

### 5.1 集合名称生成 (带冲突避免)

```typescript
public getCollectionName(codebasePath: string): string {
  const prefix = this.getIsHybrid() ? 'hybrid_code_chunks' : 'code_chunks';
  const normalizedPath = path.resolve(codebasePath);
  
  // MD5 哈希确保唯一性，同时保留可读前缀
  const pathHash = crypto
    .createHash('md5')
    .update(normalizedPath)
    .digest('hex')
    .substring(0, 8);
  
  return `${prefix}_${pathHash}`;
}
```

### 5.2 忽略模式匹配 (简单 Glob)

```typescript
private isPatternMatch(filePath: string, pattern: string): boolean {
  // 将 glob 转换为正则
  const regexPattern = pattern
    .replace(/[.+^${}()|[\]\\]/g, '\\$&')  // 转义特殊字符
    .replace(/\*/g, '.*');                    // * 匹配任意字符
  
  return new RegExp(`^${regexPattern}$`).test(filePath);
}
```

### 5.3 Chunk ID 生成 (内容寻址)

```typescript
private generateId(relativePath, startLine, endLine, content): string {
  // 基于路径+位置+内容的 SHA256
  const combined = `${relativePath}:${startLine}:${endLine}:${content}`;
  const hash = crypto.createHash('sha256')
    .update(combined, 'utf-8')
    .digest('hex');
  return `chunk_${hash.substring(0, 16)}`;
}
```

---

## 六、项目亮点与创新点

### 6.1 🌟 混合搜索 (Hybrid Search)

项目支持**稠密向量 + 稀疏向量**的混合搜索，通过 RRF 融合排序，兼顾语义理解和关键词精确匹配：

- **稠密向量**：捕捉语义相似性 ("user authentication" ≈ "login verification")
- **稀疏向量 (BM25)**：精确匹配关键词 (如特定函数名、变量名)
- **RRF 融合**：综合两种排序，提升整体召回率

### 6.2 🌟 AST 感知分块

相比传统的字符级分块，AST 分块：
- 按函数/类边界切割，不破坏代码结构
- 每个 chunk 都是语义完整的逻辑单元
- 提升嵌入质量和搜索相关性

### 6.3 🌟 多平台 MCP 集成

支持 10+ 种 AI 编程工具：

- Claude Code (官方)
- OpenAI Codex CLI
- Gemini CLI
- Cursor, Windsurf, Void
- VS Code, Cherry Studio
- Qwen Code

### 6.4 🌟 增量同步与快照

- Merkle 树检测文件变更
- 只更新变更的文件，大幅降低重复索引成本
- 快照持久化索引状态

---

## 七、潜在改进空间

| 方面 | 建议 |
|------|------|
| **多模态支持** | 增加代码图 (Call Graph, Dependency Graph) 索引 |
| **本地部署** | 支持纯本地模式 (Ollama + 本地 Milvus/Faiss) |
| **查询优化** | 添加查询意图识别，自动选择搜索策略 |
| **缓存策略** | 嵌入结果缓存，避免重复计算 |
| **并行处理** | 文件级并行索引，提升速度 |
| **语言扩展** | 增加更多语言支持 (Ruby, PHP, Swift 等) |

---

## 八、总结

**claude-context** 是一个架构清晰、工程实践优秀的开源项目：

1. **清晰的模块划分**：Core / MCP / Extensions 三层架构
2. **良好的抽象设计**：Splitter/Embedding/VectorDatabase 接口便于扩展
3. **务实的降级策略**：AST 失败自动降级 LangChain
4. **完整的生态集成**：支持多种主流 AI 编程工具
5. **云原生设计**：深度集成 Zilliz Cloud，降低使用门槛

项目展示了如何构建一个**生产级的 AI 代码工具**，在语义搜索、向量数据库应用、MCP 协议集成等方面都有值得学习的设计。

---

*报告生成时间：2026-04-25 03:00 CST*  
*分析项目：[zilliztech/claude-context](https://github.com/zilliztech/claude-context)*
