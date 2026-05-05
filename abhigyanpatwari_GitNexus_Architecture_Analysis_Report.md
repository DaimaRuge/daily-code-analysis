# abhigyanpatwari/GitNexus 深度架构分析

> 分析日期：2026-05-04
> 项目地址：https://github.com/abhigyanpatwari/GitNexus
> Stars：32k
> 技术标签：Knowledge Graph · MCP · Tree-sitter · LadybugDB · Code Intelligence

---

## 一、项目概述

GitNexus 定位为 **"Building nervous system for agent context"**（为 AI Agent 构建代码感知神经中枢）。它将任意代码库索引为知识图谱，通过 MCP 协议向 AI Agent 暴露深层代码上下文，让 Agent 在修改代码时不再遗漏依赖链、不再破坏调用链、不再盲目编辑。

**核心理念**：DeepWiki 帮助人类*理解*代码；GitNexus 让 AI Agent *分析*代码——因为知识图谱追踪的是每一条关系，而不仅仅是描述性文本。

### 两种使用模式

| 维度 | CLI + MCP（推荐） | Web UI |
|------|------------------|--------|
| 定位 | 本地索引，连接 AI Agent | 浏览器内可视化图谱 + AI 对话 |
| 存储 | LadybugDB 原生（快速、持久） | LadybugDB WASM（内存中，会话级） |
| 解析 | Tree-sitter 原生绑定 | Tree-sitter WASM |
| 隐私 | 全部本地，无网络请求 | 全部在浏览器中，无服务端 |
| 规模 | 任意大小仓库 | 受浏览器内存限制（约 5k 文件），或通过后端模式无限扩展 |
| 安装 | `npm install -g gitnexus` | 免安装：[gitnexus.vercel.app](https://gitnexus.vercel.app) |

**桥接模式**：`gitnexus serve` 可将 CLI 与 Web UI 连接，Web UI 自动发现本地服务，无需重新上传或重新索引。

---

## 二、核心架构

### 2.1 仓库结构（Monorepo）

```
gitnexus/              # CLI + MCP Server（HTTP API + stdio）+ 索引管道 + LadybugDB 图
gitnexus-web/          # Vite + React 轻量客户端（图谱浏览器 + AI 聊天）
gitnexus-shared/       # CLI 和 Web 共用的 TypeScript 类型和常量
.claude/               # Agent Skills（自动安装到项目的 .claude/skills/）
eval/                  # 工具使用基准测试评估工具
.github/              # CI 工作流 + 复合 Action
deploy/kubernetes/     # K8s 部署配置
docs/                  # 文档
```

### 2.2 端到端数据流：Index → Graph → Tools

```
Ingestion（摄取）
  └─ analyze.ts → runFullAnalysis → runPipelineFromRepo
     └─ 12 相位 DAG，输出 in-memory KnowledgeGraph
     └─ 持久化到 .gitnexus/ + 注册到 ~/.gitnexus/registry.json

Persistence（持久化）
  └─ repo-manager.ts（路径、注册表、KuzuDB 清理）
  └─ lbug-adapter.ts（图加载、查询、嵌入批量）

Query Layer（查询层，三套接口共用同一后端）
  ├─ MCP stdio：mcp.ts → LocalBackend → tools.ts + resources.ts
  ├─ HTTP Bridge：serve.ts → Express（api.ts + mcp-http.ts）→ Web UI
  └─ CLI Direct：gitnexus query|context|impact|cypher

Staleness Detection（失效检测）
  └─ staleness.ts：对比 indexed lastCommit 与 HEAD，表面提示
```

### 2.3 12 相位 DAG 管道

管道按拓扑顺序执行，每相位显式声明依赖，类型安全：

```
scan → structure → [markdown, cobol] → parse → [routes, tools, orm]
  → crossFile → mro → communities → processes
```

| 相位 | 依赖 | 输出 |
|------|------|------|
| `scan` | — | 文件路径 + 大小 |
| `structure` | `scan` | File/Folder 节点 + CONTAINS 边 + allPathSet |
| `markdown` | `structure` | Markdown section 节点 + 交叉链接边 |
| `cobol` | `structure` | COBOL 程序/段落/section 节点 |
| `parse` | `structure`, `markdown`, `cobol` | Symbol 节点 + IMPORTS/CALLS/EXTENDS 边，提取路由/Tools/ORM 查询 |
| `routes` | `parse` | 路由节点 + HANDLES_ROUTE 边 |
| `tools` | `parse` | 工具节点 + HANDLES_TOOL 边 |
| `orm` | `parse` | QUERIES 边（Prisma, Supabase） |
| `crossFile` | `parse`, `routes`, `tools`, `orm` | 跨文件类型传播（拓扑导入顺序） |
| `mro` | `crossFile`, `structure` | METHOD_OVERRIDES + METHOD_IMPLEMENTS 边 |
| `communities` | `mro`, `structure` | Community 节点 + MEMBER_OF 边（Leiden 算法） |
| `processes` | `communities`, `routes`, `tools`, `structure` | Process 节点 + STEP_IN_PROCESS 边 |

**设计亮点**：
- 单图累加器模式：所有相位向同一个 `KnowledgeGraph` 写入
- 类型化相位访问：`getPhaseOutput<T>(deps, 'name')` 保证上游类型安全
- 可跳过的相位：`skipGraphPhases` 跳过 MRO/communities/processes（加速测试）
- DAG 验证：Kahn 拓扑排序检测循环并输出具体路径（如 `A -> B -> C -> A`）

### 2.4 调用解析 DAG（Call-Resolution Pipeline）

解析相位内部的 6 阶段 typed pipeline（`call-processor.ts`），负责方法/函数调用解析并发出 CALLS 边：

```
extract-call → classify-form → infer-receiver → select-dispatch → resolve-target → emit-edge
```

- **Stage 1-2（共享）**：在 worker 中运行，按语言提取器
- **Stage 3-4（语言插件钩子）**：通过 `LanguageProvider` 接口注入（如 Ruby 实现 `inferImplicitReceiver` 和 `selectDispatch`）
- **Stage 5（MRO walk）**：通过 `lookupMethodByOwnerWithMRO` 进行方法解析
- **Stage 6（写图）**：以置信度分级写入 CALLS 边

**Scope-Resolution Pipeline（RFC #909）**：与调用解析 DAG 并行，是 Python/C# 等语言的新路径，替换 stage 1-6。两条路径共存，通过 `MIGRATED_LANGUAGES` + `REGISTRY_PRIMARY_<LANG>` 环境变量切换语言。两个路径产生相同图结构，CI 有专项测试（`ci-scope-parity.yml`）保证两边一致。

---

## 三、模块分析

### 3.1 MCP 工具层

**16 个 MCP 工具**（11 个单仓库 + 5 个多仓库组）：

| 工具 | 能力 |
|------|------|
| `list_repos` | 发现所有已索引仓库 |
| `query` | 混合搜索（BM25 + 向量 + RRF）按执行流程分组 |
| `context` | 符号的全向视图——分类引用、参与流程 |
| `impact` | 爆炸半径分析（深度分组 + 置信度） |
| `detect_changes` | Git diff 映射到受影响的符号和流程 |
| `rename` | 图辅助的多文件重命名（dry_run 预览） |
| `cypher` | 原始 Cypher 图查询 |
| `group_*`（5个） | 多仓库组的合约提取、跨仓库查询、状态检查 |

**MCP Resources**（即时上下文）：
- `gitnexus://repos` → 仓库列表
- `gitnexus://repo/{name}/context` → 代码库统计 + 失效检查
- `gitnexus://repo/{name}/clusters` → 功能聚类（含内聚力分数）
- `gitnexus://repo/{name}/processes` → 所有执行流
- `gitnexus://repo/{name}/process/{name}` → 完整流程追踪
- `gitnexus://repo/{name}/schema` → Cypher 查询图模式

**2 个 MCP Prompts**：
- `detect_impact`：提交前变更分析——范围、受影响流程、风险级别
- `generate_map`：从知识图谱生成架构文档（含 Mermaid 图表）

### 3.2 多仓库 MCP 架构

通过 **全局注册表** 实现一个 MCP Server 服务多个仓库：

```
~/.gitnexus/registry.json  ←  全局注册（每个 gitnexus analyze 注册）
      │
      ▼
MCP Server 读取注册表
      │
      ├── LocalBackend ── Connection Pool（lazy open，最多 5 个并发）
      │                    ├─ Conn A ──→ Repo A/.gitnexus/
      │                    └─ Conn B ──→ Repo B/.gitnexus/
```

- 连接按需打开，闲置 5 分钟后关闭
- 单仓库场景 `repo` 参数可选，Agent 无感知切换

### 3.3 知识图谱存储

- **CLI 模式**：LadybugDB 原生（本地持久化）
- **Web 模式**：LadybugDB WASM（内存中，per-session）
- **多仓库**：每个仓库独立索引（`.gitnexus/` 目录，portable，gitignored）+ 全局注册表

### 3.4 AI Agent 深度集成

| Editor | MCP | Skills | Hooks | 支持级别 |
|--------|-----|--------|-------|----------|
| **Claude Code** | ✅ | ✅ | ✅（PreToolUse + PostToolUse）| 完整 |
| Cursor | ✅ | ✅ | ❌ | MCP + Skills |
| Codex | ✅ | ✅ | ❌ | MCP + Skills |
| Windsurf | ✅ | ❌ | ❌ | MCP |
| OpenCode | ✅ | ✅ | ❌ | MCP + Skills |

**Claude Code 特有深度集成**：
- `PreToolUse` 钩子：在工具调用前注入图谱上下文
- `PostToolUse` 钩子：检测提交后索引过期，自动提示 Agent 重新索引
- 自动安装 4 个 Agent Skills（Exploring / Debugging / Impact Analysis / Refactoring）
- `--skills` 生成仓库特定的 Skills（基于 Leiden 社区检测，为每个功能区生成 `SKILL.md`）

### 3.5 Wiki 生成

```
gitnexus wiki [path]
  └─ 从知识图谱自动生成仓库文档
  └─ 支持自定义 LLM 模型和 API base URL
  └─ Enterprise 版支持自动更新
```

---

## 四、竞品对比

| 维度 | GitNexus | DeepWiki | GitHub Copilot | Sourcegraph | SWE-agent |
|------|----------|----------|----------------|-------------|-----------|
| **定位** | AI Agent 代码感知 | 人类代码理解 | IDE 内补全 | 代码搜索 | AI Agent 自主修复 |
| **图谱** | ✅ 完整 KG | 有限图谱 | ❌ | ❌ | ❌ |
| **运行位置** | 客户端本地 | 云端/本地 | 云端 | 云端 | 本地 |
| **MCP 协议** | ✅ 原生 | ❌ | ❌ | ❌ | ❌ |
| **多仓库支持** | ✅（组架构） | ❌ | ❌ | ✅ | ❌ |
| **Scope Resolution** | ✅（RFC #909） | ❌ | ❌ | ❌ | ❌ |
| **调用链解析** | ✅ MRO + C3 + Ruby mixin | 基础 | ❌ | 基础 | ❌ |
| **Tree-sitter** | ✅ | ❌ | ❌ | 部分 | ❌ |
| **社区检测** | ✅ Leiden 算法 | ❌ | ❌ | ❌ | ❌ |
| **许可证** | PolyForm 非商业 | 商业 | 商业 | 商业 | MIT |

**GitNexus 差异化优势**：
1. **AI Native MCP 工具**：不是给人用的图谱浏览，而是给 Agent 用的工具集（blast radius、context、impact、detect_changes）
2. **完全客户端运行**：隐私优先，数据不出机器
3. **12 相位 DAG + 知识图谱**：比简单索引更深，追踪所有关系
4. **Scope-Resolution Pipeline**：现代语言解析路径（Python/C# 迁移中），两条路径 CI 保证一致性
5. **多仓库互联**：通过 group sync 提取跨仓库合约，构建服务间依赖图

---

## 五、洞察总结

### 5.1 架构亮点

- **渐进式图构建**：12 相位 DAG，每相独立演进，依赖关系清晰
- **语言无关解析**：Tree-sitter + LanguageProvider 钩子机制，支持 Ruby/C#/Python 等多语言定制行为
- **双路径解析保障**：Legacy DAG + Scope-Resolution Pipeline 并行，CI parity 测试确保一致性，不影响现有用户
- **多仓库统一视图**：group sync + Contract Bridge 实现跨仓库影响分析，服务间依赖可追踪
- **零服务端隐私**：所有处理在客户端完成，适合对代码隐私有要求的企业场景

### 5.2 实际价值

| 场景 | GitNexus 如何帮助 |
|------|------------------|
| **修改旧代码** | `impact` 工具告诉 Agent 修改会影响哪些流程/模块 |
| **接新项目** | `context` 工具提供符号的全向视图，不再盲目摸索 |
| **跨仓库重构** | `group_query` 跨仓库搜索执行流，`group_sync` 提取并匹配合约 |
| **代码评审** | Enterprise `PR Review` 自动爆炸半径分析 |
| **让小模型变大** | 即使是 GPT-4o-mini 这样的小模型，有了图谱上下文也能获得完整架构洞察 |

### 5.3 潜在局限

- **非商业许可证**：PolyForm Noncommercial 限制商业使用（有 Enterprise 版商业授权通道）
- **大型仓库性能**：Tree-sitter 解析在大型单文件中可能超时，有 `--worker-timeout` 调优参数
- **语言覆盖**：主要支持主流语言，Enterprise 版才支持 OCaml
- **Web UI 内存限制**：纯浏览器模式受 ~5k 文件限制，需后端模式或 Bridge 模式突破

### 5.4 发展趋势

- Enterprise 功能下放（PR Review、Auto-reindexing、Multi-repo）→ 开源版功能增强路径参考
- 更多语言迁移到 Scope-Resolution Pipeline（当前 Python/C#）
- 自动化回归取证（Auto regression forensics）和端到端测试生成（Upcoming）
- 社区插件生态（已有 pi-gitnexus、gitnexus-stable-ops）

---

*报告生成工具：OpenClaw Code Architecture Analyzer*
