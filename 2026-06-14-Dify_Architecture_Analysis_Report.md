# Dify — 技术架构与源码研读报告

> **项目**: [langgenius/dify](https://github.com/langgenius/dify)  
> **版本**: v1.14.2  
> **Stars**: 145K+ | **Forks**: 22K+  
> **分析日期**: 2026-06-14  
> **分析人**: OpenClaw Daily Code Analysis Bot

---

## 一、项目概述

**Dify** 是一个开源的 LLM 应用开发平台，定位为「Production-ready platform for agentic workflow development」。它提供从 Prompt 编排、RAG 检索、Agent 执行到 Workflow 编排的完整闭环，支持自托管（Self-hosted）和云服务（Dify Cloud）两种部署模式。

核心标签覆盖：Agentic AI、Workflow Orchestration、RAG、Low-Code/No-Code、MCP、Plugin Ecosystem。

---

## 二、宏观系统架构

Dify 采用 **前后端分离 + 微服务容器化** 架构，由以下核心服务组成：

```
┌─────────────────────────────────────────────────────────────┐
│                        Nginx (Reverse Proxy)                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Dify Web   │  │  Dify API   │  │   Dify Worker       │  │
│  │  (Next.js)  │  │  (Flask)    │  │   (Celery)          │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │Plugin Daemon│  │   Sandbox   │  │    Dify Agent       │  │
│  │  (gRPC)     │  │ (Code Exec) │  │  (Agent Runtime)    │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│  PostgreSQL / MySQL    Redis    Weaviate/Qdrant/pgvector   │
│  (Relational DB)      (Cache/Queue)    (Vector Store)       │
└─────────────────────────────────────────────────────────────┘
```

### 关键服务角色

| 服务 | 技术栈 | 职责 |
|------|--------|------|
| **dify-web** | Next.js 16 + React 19 | 管理控制台、应用画布、对话界面 |
| **dify-api** | Python 3.12 + Flask + gevent | REST API、WebSocket、业务逻辑 |
| **dify-worker** | Celery + gevent | 异步任务：索引、导入、定时触发 |
| **dify-plugin-daemon** | gRPC / HTTP | 插件生命周期管理、安全沙箱通信 |
| **dify-sandbox** | 独立容器 | 代码执行沙箱（Python/Node.js） |
| **dify-agent** | Python + pydantic-ai | 新一代 Agent 运行时（v1.14+） |
| **ssrf-proxy** | Squid | SSRF 防护代理 |

---

## 三、后端架构深度解析

### 3.1 应用工厂模式（App Factory）

Dify 后端采用 **Flask Application Factory** 模式，核心入口在 `api/app.py`：

```python
# api/app.py
from app_factory import create_app
socketio_app, flask_app = create_app()
app = flask_app
celery = cast("Celery", app.extensions["celery"])
```

工厂函数 `create_app()` 在 `api/app_factory.py` 中完成以下初始化：

1. **配置加载**：从 `.env` 读取 `dify_config`（Pydantic Settings 模型）
2. **扩展注册**：Celery、SQLAlchemy、Flask-Login、Socket.IO、OpenTelemetry
3. **蓝图挂载**：Console API、Web API、Service API、Inner API、OpenAPI
4. **请求钩子**：
   - `before_request`：请求上下文初始化 + 企业许可证校验
   - `after_request`：OpenTelemetry Trace Header 注入

**关键设计点**：
- 数据库迁移命令（`flask db`）走独立工厂 `create_migrations_app()`，避免加载 WebSocket/Celery
- 企业版功能通过 `ENTERPRISE_ENABLED` 开关 + 许可证校验层控制

### 3.2 数据库层（SQLAlchemy + 多数据库支持）

数据库模型位于 `api/models/`，采用 **SQLAlchemy 2.0 Declarative Mapping**（`Mapped`, `mapped_column`）。

核心模型：

| 模型 | 文件 | 职责 |
|------|------|------|
| `Account` | `account.py` | 用户/租户/工作空间 |
| `App` | `model.py` | 应用定义（Chat/Agent/Workflow） |
| `Workflow` | `workflow.py` | 工作流图拓扑、节点配置、变量定义 |
| `Dataset` | `dataset.py` | RAG 知识库、文档、分段 |
| `Provider` | `provider.py` | LLM/Embedding/Rerank 模型提供商配置 |
| `Tool` | `tools.py` | 内置工具与插件工具注册 |
| `Conversation` / `Message` | `model.py` | 对话历史、消息链 |

**数据库设计亮点**：
- **工作流模型**：`Workflow` 模型存储 JSON 化的图拓扑（`graph_config`），节点类型通过 `graphon` 库（Dify 自研的图引擎）进行类型校验和执行
- **文件系统**：`UploadFile` 模型支持多存储后端（本地/MinIO/S3），通过 `Storage` 扩展抽象
- **多租户**：`Tenant` 和 `Workspace` 两级隔离，API 通过 `Account` 上下文注入当前租户

### 3.3 核心引擎（`api/core/`）

`core/` 是 Dify 的心脏，包含以下子系统：

#### 3.3.1 工作流引擎（`core/workflow/`）

Dify 的工作流引擎是 **有向无环图（DAG）执行器**，支持：

- **节点类型**：LLM、知识检索、Agent、条件分支、HTTP 请求、代码执行、变量赋值、回答、问题分类、迭代、循环、模板转换、定时触发、Webhook 触发、插件触发
- **图拓扑**：`graph_topology.py` 负责 DAG 校验（检测循环、孤立节点）
- **运行时**：`node_runtime.py` + `node_factory.py` 按拓扑序执行节点
- **变量池**：`variable_pool_initializer.py` 管理节点间数据传递，支持系统变量、会话变量、环境变量
- **人机协作**：`human_input_adapter.py` + `human_input_policy.py` 支持执行暂停等待用户输入

**执行模型**：
```python
# 伪代码
workflow_entry = WorkflowEntry(workflow_config)
for node in topological_sort(workflow_entry.graph):
    result = node_runtime.execute(node, variable_pool)
    variable_pool.set(node.id, result)
    if node.requires_human_input:
        raise PauseExecution(HumanInputRequired(...))
```

#### 3.3.2 Agent 执行器（`core/agent/`）

Dify 实现了多种 Agent 策略模式：

| 策略 | 类 | 描述 |
|------|-----|------|
| **CoT** | `cot_agent_runner.py` | Chain-of-Thought 推理 |
| **Function Call** | `fc_agent_runner.py` | OpenAI/Anthropic 函数调用 |
| **ReAct** | `base_agent_runner.py` | 推理-行动循环 |

Agent 执行器继承 `AppRunner`（应用运行器基类），通过 `ToolManager` 加载工具集（内置工具 + 插件工具 + 知识库检索工具）。

**v1.14 重大变化**：引入 `agent_v2` 节点，支持：
- **多 Agent 编排**：Agent 可以作为工作流节点嵌套
- **文件回传**：`output_file_rebacker.py` 处理 Agent 产出的文件
- **插件工具构建**：`plugin_tools_builder.py` 将插件能力转换为 Agent 可用工具

#### 3.3.3 RAG 管道（`core/rag/`）

完整的 RAG 流水线：

```
文档上传 → 分段器(splitter) → 索引处理器(index_processor) → 向量存储
                                              ↓
用户查询 → 检索器(retrieval) → 重排序(rerank) → 上下文注入 → LLM
```

关键组件：
- **分段器**：支持语义分段、父-子分段（Parent-Child）、递归字符分段
- **索引处理器**：支持 `economy`（关键词）和 `high_quality`（向量）两种索引模式
- **向量数据库抽象**：支持 Weaviate、Qdrant、Milvus、pgvector、OceanBase、Chroma、Elasticsearch 等 15+ 种向量存储
- **检索策略**：关键词检索、向量检索、混合检索（Hybrid Search）+ 重排序（Rerank）

#### 3.3.4 模型管理层（`core/model_manager.py`）

Dify 的模型管理采用 **Provider-Model-Instance** 三级架构：

- **Provider**：模型供应商（OpenAI、Anthropic、Azure、Gemini、本地 Ollama 等）
- **Model**：模型定义（GPT-4、Claude 3、Llama 3 等）
- **Instance**：带凭证的模型实例，支持多租户隔离

模型运行时通过 `graphon.model_runtime` 抽象层调用，支持：
- LLM 生成（文本/结构化输出）
- Embedding（文本向量化）
- Rerank（重排序）
- Speech-to-Text / Text-to-Speech
- Moderation（内容审核）
- Multi-modal（图像理解）

### 3.4 插件架构（Plugin System）

Dify 的插件系统是其生态扩张的核心机制。

#### 3.4.1 插件守护进程（Plugin Daemon）

`dify-plugin-daemon` 是独立服务，通过 gRPC/HTTP 与主 API 通信：

```python
# api/core/plugin/impl/base.py
plugin_daemon_inner_api_baseurl = URL(str(dify_config.PLUGIN_DAEMON_URL))
```

职责：
- 插件生命周期管理（安装、卸载、启用、禁用）
- 插件沙箱执行（隔离的 Python/Node.js 运行时）
- 插件市场接口（上传、下载、版本管理）

#### 3.4.2 插件类型

`core/plugin/impl/` 下实现了多种插件契约：

| 实现 | 类型 | 说明 |
|------|------|------|
| `model.py` | Model Provider | 扩展 LLM/Embedding/Rerank 提供商 |
| `tool.py` | Tool | 扩展 Agent 可用工具 |
| `agent.py` | Agent Strategy | 扩展 Agent 推理策略 |
| `datasource.py` | Data Source | 扩展 RAG 数据源 |
| `endpoint.py` | Endpoint | 扩展 API 端点 |
| `trigger.py` | Trigger | 扩展工作流触发器 |
| `oauth.py` | OAuth | 扩展 OAuth 认证 |
| `asset.py` | Asset | 扩展静态资源 |

插件通过 **Backwards Invocation**（反向调用）机制与主服务通信：插件守护进程可以回调主 API 获取租户配置、文件存储、数据库访问等能力。

### 3.5 MCP 集成（Model Context Protocol）

Dify v1.14+ 深度集成了 Anthropic 的 MCP 协议，支持作为 MCP 客户端连接外部 MCP 服务器：

```python
# api/core/mcp/mcp_client.py
class MCPClient:
    def __init__(self, server_url: str, headers: dict | None = None, ...):
        self._session: ClientSession | None = None
        # 支持 streamable HTTP 和 SSE 两种传输方式
```

**传输协议**：
- `streamablehttp_client`：基于 HTTP 的 Streamable 传输（MCP 2024-11 规范）
- `sse_client`：Server-Sent Events 传输（MCP 2024-06 规范）

**安全设计**：
- MCP 请求通过 SSRF Proxy（Squid）转发，防止内网扫描
- `create_ssrf_proxy_mcp_http_client()` 创建带代理的 HTTP 客户端

### 3.6 Dify-Agent（新一代 Agent 运行时）

`dify-agent/` 是 v1.14 引入的 **独立 Agent 运行时**，与主服务通过协议解耦：

```
dify-agent/
├── src/
│   ├── agenton/              # Agent 编排引擎（核心框架）
│   │   ├── compositor/       # 组合器：编排 Agent 执行层
│   │   └── layers/           # 抽象层：模型层、工具层、历史层
│   ├── agenton_collections/  # 预设层集合
│   │   ├── layers/plain/     # 基础层（basic, dynamic_tools）
│   │   └── layers/pydantic_ai/  # pydantic-ai 适配层
│   └── dify_agent/           # Dify 特定实现
│       ├── adapters/         # 适配器
│       ├── client/           # 客户端 SDK
│       ├── layers/           # Dify 自定义层
│       ├── protocol/         # 运行协议（Run Protocol）
│       ├── runtime/          # 运行时
│       ├── server/           # 服务端
│       └── storage/          # 状态存储
```

**核心设计**：
- **Agenton**：通用 Agent 编排框架，类似 LangChain/LangGraph 的抽象层
- **Layer 架构**：Agent 执行由多个 Layer 组合而成（Model Layer → Tool Layer → History Layer → Output Layer）
- **Pydantic-AI 集成**：通过 `agenton_collections` 的 pydantic-ai 层，利用 Pydantic 的类型安全进行 Agent 输出校验
- **Plugin Daemon Transport**：Agent 通过插件守护进程安全调用外部工具

**协议设计**：
```python
# dify_agent/protocol/schemas.py
class CreateRunRequest(BaseModel): ...
class RunEvent(BaseModel): ...
class RunStatus(BaseModel): ...
```

支持流式事件（`RunEvent`）驱动 Agent 执行，可取消、可监控、可扩展。

---

## 四、前端架构解析

### 4.1 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Next.js | 16.2.7 | SSR/SSG 框架 |
| React | 19.2.7 | UI 框架 |
| TypeScript | 6.0.3 | 类型系统 |
| Tailwind CSS | 4.3.0 | 样式系统 |
| pnpm | workspace | 包管理 |
| Zustand | 5.0.14 | 状态管理 |
| Jotai | 2.20.0 | 原子化状态 |
| React Query | 5.101.0 | 服务端状态 |
| ReactFlow | 11.11.4 | 工作流画布 |
| Lexical | 0.45.0 | 富文本编辑器 |
| Monaco Editor | 4.7.0 | 代码编辑器 |
| Mermaid | 11.15.0 | 图表渲染 |
| ECharts | 6.1.0 | 数据可视化 |
| Socket.IO | 4.8.3 | 实时通信 |

### 4.2 目录结构

```
web/
├── app/                 # Next.js App Router 页面
├── features/            # 按功能模块组织的组件
│   ├── account-profile
│   ├── system-features
│   └── tag-management
├── service/             # API 调用层（封装后端接口）
├── models/              # 前端数据模型/类型定义
├── hooks/               # 自定义 React Hooks
├── context/             # React Context
├── plugins/             # 前端插件系统
├── i18n/                # 国际化（20+ 语言）
├── themes/              # 主题配置
└── utils/               # 工具函数
```

**Feature-Based 组织**：前端按功能模块（Feature）而非技术类型组织代码，每个 feature 包含自己的组件、hooks、service、types，提升可维护性。

### 4.3 工作流画布（Visual Editor）

工作流画布是 Dify 最具特色的前端功能：

- **ReactFlow**：基于 ReactFlow 的节点-边编辑器
- **ELK.js**：自动布局算法（分层布局）
- **节点类型**：支持 20+ 种节点，每种节点有独立的配置面板
- **实时预览**：画布右侧实时预览执行结果
- **变量系统**：节点间通过变量连线传递数据，支持类型推断和校验

### 4.4 Monorepo 包管理

`pnpm-workspace.yaml` 定义了 workspace：

```yaml
packages:
  - web              # 前端主应用
  - e2e              # E2E 测试
  - sdks/nodejs-client  # Node.js SDK
  - packages/*       # 共享包
    - packages/contracts     # 前后端共享契约
    - packages/dify-ui       # UI 组件库
    - packages/iconify-collections  # 图标集合
    - packages/tsconfig      # 共享 TS 配置
    - packages/dev-proxy     # 开发代理
```

---

## 五、部署架构

### 5.1 Docker Compose 部署

Dify 提供完整的 Docker Compose 编排，支持 **一键启动** 生产环境：

核心服务（`docker-compose.yaml`）：
- `api`：主 API 服务（Gunicorn + gevent workers）
- `worker`：Celery 异步任务队列
- `web`：Next.js 前端（静态导出 + Nginx）
- `plugin_daemon`：插件守护进程
- `sandbox`：代码执行沙箱
- `ssrf_proxy`：SSRF 防护代理

数据层：
- `postgres` / `mysql`：关系数据库
- `redis`：缓存 + 消息队列（Celery broker）
- `weaviate` / `qdrant` / `pgvector`：向量数据库（可选其一）
- `minio`：对象存储（文件上传）
- `nginx`：反向代理 + SSL 终止

**环境变量管理**：`envs/` 目录按服务分组存放环境变量模板，支持 20+ 种向量数据库、多种关系数据库、多种对象存储的灵活切换。

### 5.2 安全设计

| 安全机制 | 实现 |
|----------|------|
| **SSRF 防护** | Squid 代理隔离，限制内网访问 |
| **代码沙箱** | `dify-sandbox` 独立容器，限制执行时间和资源 |
| **文件隔离** | 多租户文件存储隔离，MinIO/S3 前缀隔离 |
| **API 认证** | JWT Token + API Key 双模式 |
| **企业许可证** | 加密许可证校验，支持过期回退 |

---

## 六、关键设计模式与架构决策

### 6.1 设计模式

| 模式 | 应用位置 | 说明 |
|------|----------|------|
| **Factory** | `app_factory.py`, `node_factory.py` | 应用/节点创建工厂 |
| **Strategy** | `core/agent/strategy/` | Agent 推理策略切换 |
| **Adapter** | `dify_agent/adapters/`, `core/plugin/impl/` | 多供应商/多协议适配 |
| **Observer** | `core/events/`, Socket.IO | 事件驱动通信 |
| **Repository** | `api/repositories/` | 数据访问抽象 |
| **Unit of Work** | SQLAlchemy Session | 事务边界管理 |
| **Plugin/Extension** | `core/plugin/`, `api/extensions/` | 可扩展架构 |
| **Graph DAG** | `core/workflow/`, `graphon` | 工作流图执行 |
| **Backwards Invocation** | Plugin Daemon | 插件反向调用主服务 |

### 6.2 架构决策记录（ADR）

**ADR-1：Python 后端 + TypeScript 前端**
- Python 生态在 AI/ML 领域成熟，Flask 轻量适合快速迭代
- Next.js 16 + React 19 提供现代前端体验，App Router 支持 SSR

**ADR-2：自研 Graphon 图引擎**
- 工作流执行需要严格的 DAG 校验、拓扑排序、暂停恢复
- 通用工作流引擎（如 Airflow）过于重型，不适合实时对话场景
- 自研 `graphon` 库提供细粒度控制

**ADR-3：Plugin Daemon 独立进程**
- 插件执行需要安全隔离（不可信代码）
- 独立进程可通过 gRPC 通信，崩溃不影响主服务
- 支持多语言插件（Python、Node.js）

**ADR-4：Celery 异步任务队列**
- 文档索引、批量导入、定时触发等任务需要异步处理
- Redis 作为 Broker，支持任务优先级和重试

**ADR-5：MCP 协议原生支持**
- MCP 正在成为 LLM 工具调用的事实标准
- 原生支持比桥接模式更稳定，用户体验更好

---

## 七、代码质量观察

### 7.1 优势

1. **类型安全**：后端全面使用 Python 3.12 + Pydantic 类型校验，前端 TypeScript 6.x 严格模式
2. **模块化**：`core/` 按子系统清晰划分，每个模块有独立的实体、错误、接口定义
3. **国际化**：支持 20+ 语言，包括克林贡语（Klingon）——体现社区活力
4. **测试覆盖**：`api/tests/` + `e2e/` + `web/test/`，单元测试 + E2E 测试分层
5. **文档完善**：`docs/` 多语言文档，API 文档自动生成（OpenAPI/Swagger）
6. **企业级**：许可证管理、SSRF 防护、审计日志、多租户隔离

### 7.2 潜在改进点

1. **代码体积**：`api/` 目录庞大（30+ 子目录），部分模块耦合度较高（如 `model.py` 同时包含 App、Conversation、Message 等模型）
2. **配置复杂度**：`.env.example` 超过 200 个配置项，新用户上手门槛较高
3. **前端构建**：`eslint-suppressions.json` 20万行，说明遗留代码较多，技术债需要偿还
4. **Monorepo 边界**：`web/` 和 `api/` 之间共享类型契约，但 `packages/contracts` 是否真正被两端消费需要确认

---

## 八、版本演进趋势

从 v1.14.2 的代码结构可观察到 Dify 的演进方向：

1. **Agent 优先**：`dify-agent/` 独立模块 + `agent_v2` 工作流节点，Agent 从功能模块升级为一等公民
2. **MCP 原生化**：`core/mcp/` 完整客户端实现，MCP 工具可直接在 Agent 和 Workflow 中使用
3. **去 LangChain 化**：引入 `graphon` 和 `agenton` 自研框架，降低对外部依赖的耦合
4. **可视化增强**：`reactflow` + `elkjs` + `mermaid` 组合，工作流画布体验持续提升
5. **多模态**：文件上传、图像理解、语音输入输出等能力逐步完善

---

## 九、总结

Dify 是一个**架构成熟、设计精良、生态活跃**的 LLM 应用开发平台。其架构特点可概括为：

- **分层清晰**：前端/后端/插件/Agent 四层解耦，每层可独立演进
- **扩展性强**：Plugin Daemon + MCP 双生态，支持无限扩展
- **生产就绪**：Docker 化部署、多数据库支持、企业安全、国际化、可观测性（OpenTelemetry）
- **Agent 原生**：从 Workflow 编排到 Agent 执行，再到 Dify-Agent 独立运行时，Agent 能力贯穿全栈

对于希望构建 LLM 应用平台的开发者，Dify 的源码是**极佳的学习材料**——它展示了如何在传统 Web 架构基础上，优雅地融入 AI 能力、插件生态、实时协作和可视化编排。

---

*报告生成时间：2026-06-14 03:00 CST*  
*下次分析：2026-06-15*
