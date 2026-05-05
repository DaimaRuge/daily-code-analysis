# agent-infra/sandbox 深度架构分析

> **分析日期**: 2026-05-05  
> **项目地址**: https://github.com/agent-infra/sandbox  
> **GitHub Stars**: ⭐4.5k+  
> **技术栈**: Docker + Python/TypeScript/Go SDK + CDP + MCP  
> **许可证**: MIT  
> **项目定位**: All-in-One Agent 沙箱环境——浏览器 + 终端 + 文件系统 + VSCode + Jupyter + MCP，一站式统一

---

## 一、项目概述

### 1.1 核心定位

AIO Sandbox 是一个**一体化的 AI Agent 沙箱环境**，在一个 Docker 容器中同时运行：
- 🌐 浏览器（VNC + CDP 协议）
- 💻 终端（WebSocket 终端）
- 📁 文件系统
- 🔧 VSCode Server
- 📊 Jupyter Notebook
- 🤖 MCP 服务

核心价值：**统一文件系统 + 多接口访问 = 跨工具无缝协作。**

### 1.2 解决的问题

传统沙箱工具都是单用途的：
- Playwright：只能做浏览器自动化
- Docker + 某些 CLI：只能跑代码
- 云端 IDE：只有 VSCode

它们之间的文件共享和功能协调极其困难。AIO Sandbox 的解法是**所有工具在一个容器里，共享同一个文件系统**：

```
浏览器下载文件 → Shell 直接访问 → VSCode 直接编辑 → Jupyter 直接处理
```

### 1.3 目标用户

- **AI Agent 开发者**：需要统一的执行环境
- **自动化测试**：需要浏览器 + 终端 + 文件系统联动
- **远程开发**：需要云端 VSCode + Jupyter
- **爬虫开发者**：需要浏览器 + 文件操作 + 代码执行

### 1.4 关键指标

| 指标 | 数值 |
|------|------|
| GitHub Stars | ⭐4.5k+ |
| 部署方式 | Docker（一键启动） |
| SDK 支持 | Python + TypeScript + Golang |
| 内置服务 | VNC / VSCode Server / Jupyter / MCP |
| MCP 服务 | Browser + File + Shell + Markitdown |
| 安全 | Seccomp 沙箱 + 隔离 Python/Node.js 执行 |

---

## 二、核心架构

### 2.1 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AIO Sandbox 架构                                 │
│                                                                     │
│  部署层                                                              │
│  └── Docker 容器 (一键启动，端口 8080)                               │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │           统一文件系统 (Shared Filesystem)                   │   │
│  │  ├── 浏览器下载 → Shell 访问 → VSCode 编辑 → Jupyter 处理   │   │
│  │  └── 所有组件共享同一 /home/gem 目录                         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    六大核心组件                                │   │
│  │                                                              │   │
│  │  🌐 Browser (浏览器)                                         │   │
│  │  ├── VNC ── 远程桌面可视化操控                               │   │
│  │  ├── CDP ── Chrome DevTools Protocol 程序化控制              │   │
│  │  └── MCP ── 高层浏览器自动化工具                              │   │
│  │                                                              │   │
│  │  💻 Shell (终端)                                             │   │
│  │  └── WebSocket 终端，共享文件系统                             │   │
│  │                                                              │   │
│  │  📁 File (文件系统)                                          │   │
│  │  └── 读取/写入，跨组件共享                                   │   │
│  │                                                              │   │
│  │  🔧 VSCode Server (开发环境)                                  │   │
│  │  └── 浏览器内完整 IDE                                         │   │
│  │                                                              │   │
│  │  📊 Jupyter (交互环境)                                        │   │
│  │  └── 交互式 Python，数据处理                                  │   │
│  │                                                              │   │
│  │  🤖 MCP Services (MCP 服务)                                   │   │
│  │  ├── browser_mcp ── 浏览器自动化                              │   │
│  │  ├── file_mcp ── 文件操作                                    │   │
│  │  ├── shell_mcp ── 命令执行                                   │   │
│  │  └── markitdown_mcp ── 文档转换                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  接口层                                                             │
│  ├── 🌐 http://localhost:8080/vnc ── VNC 浏览器                   │
│  ├── 💻 http://localhost:8080/vscode ── VSCode Server             │
│  ├── 📊 http://localhost:8080/jupyter ── Jupyter                  │
│  ├── 📖 http://localhost:8080/v1/docs ── API 文档                 │
│  └── 🤖 http://localhost:8080/mcp ── MCP 服务端点                  │
│                                                                     │
│  SDK 层                                                             │
│  ├── Python: pip install agent-sandbox                             │
│  ├── TypeScript: npm install @agent-infra/sandbox                 │
│  └── Golang: go get github.com/agent-infra/sandbox-sdk-go         │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 统一文件系统的工作流

AIO Sandbox 最核心的价值就是这个统一文件系统：

```
典型工作流示例（网页转 Markdown）：

Step 1: Browser CDP → 下载 HTML 内容，截图
Step 2: File System → 保存 HTML 到共享目录
Step 3: Jupyter → 用 markdownify 转换 HTML → Markdown
Step 4: VSCode → 打开并编辑生成的 Markdown
Step 5: Shell → git commit 提交更改
```

所有步骤之间**无需文件传输**，因为都在同一个容器里。

### 2.3 MCP 集成

AIO Sandbox 预配置了 4 个 MCP 服务：

| MCP 服务 | 用途 |
|----------|------|
| **browser_mcp** | 浏览器自动化（截图表单填） |
| **file_mcp** | 文件系统操作（读/写/目录） |
| **shell_mcp** | 命令执行 |
| **markitdown_mcp** | 文档转换（HTML → Markdown） |

这些 MCP 服务让 AI Agent 可以通过标准 MCP 协议控制整个沙箱环境。

### 2.4 安全模型

```bash
docker run --security-opt seccomp=unconfined --rm -it -p 8080:8080 ...
```

使用 `seccomp=unconfined` 允许沙箱内运行需要系统调用的操作。同时：
- Python 和 Node.js 执行都是沙箱隔离的
- 每个组件在容器内有独立的安全边界
- Seccomp 限制不允许的系统调用

---

## 三、功能模块分析

### 3.1 浏览器模块

三种访问方式：

| 接口 | 用途 | 使用场景 |
|------|------|---------|
| **VNC** | 可视化远程桌面 | 手动操作、调试 |
| **CDP** | Chrome DevTools Protocol | 程序化控制、爬虫 |
| **MCP** | 高层工具调用 | AI Agent 自动化 |

CDP 连接信息可以从 `client.browser.get_info()` 获取。

### 3.2 开发工具模块

| 工具 | 说明 |
|------|------|
| **VSCode Server** | 完整 IDE 体验，浏览器内运行 |
| **Jupyter Notebook** | 交互式 Python 环境 |
| **WebSocket Terminal** | 实时终端访问 |

Port forwarding 支持预览 Web 应用。

### 3.3 SDK 展示

**Python**：
```python
from agent_sandbox import Sandbox
client = Sandbox(base_url="http://localhost:8080")
home_dir = client.sandbox.get_context().home_dir
result = client.shell.exec_command(command="ls -la")
content = client.file.read_file(file=f"{home_dir}/.bashrc")
screenshot = client.browser.screenshot()
```

**TypeScript**：
```typescript
const sandbox = new Sandbox({ baseURL: 'http://localhost:8080' });
const result = await sandbox.shell.exec({ command: 'ls -la' });
const content = await sandbox.file.read({ path: '/home/gem/.bashrc' });
const screenshot = await sandbox.browser.screenshot();
```

---

## 四、竞品对比

### 4.1 全局对比

| 工具 | 定位 | 统一文件系统 | 多接口 | MCP | SDK |
|------|------|------------|--------|-----|------|
| **AIO Sandbox** | 全功能沙箱 | ✅ | ✅ | ✅ | Python/TS/Go |
| **Playwright MCP** | 浏览器自动化 | ❌ | ❌ | ✅ | JS/TS |
| **mcp-chrome** | 浏览器操控 | ❌ | ❌ | ✅ | ❌ |
| **bb-browser** | 网站数据访问 | ❌ | ❌ | ✅ | CLI |
| **JupyterHub** | 交互式 Notebook | ❌ | ❌ | ❌ | Python |
| **code-server** | 云端 VSCode | ❌ | ❌ | ❌ | ❌ |

### 4.2 深度对比

#### vs Playwright MCP

| 维度 | AIO Sandbox | Playwright MCP |
|------|-------------|---------------|
| **能力范围** | 浏览器+终端+文件+VSCode+Jupyter | 仅浏览器 |
| **统一文件系统** | ✅ | ❌ |
| **MCP 服务数** | 4 个 | 10+ |
| **SDK 丰富度** | Python/TS/Go | JS/TS |
| **部署** | Docker 一键 | npm 包 |

两者不是竞争关系——AIO Sandbox 实际上在其 browser 模块中**使用了 Playwright**。

#### vs 自建 Docker 环境

自建一个类似的组合环境需要：
- Docker + 手动配置 VNC
- 手动安装 Playwright/Selenium
- 手动搭建 VSCode Server
- 手动配置 Jupyter
- 手动串接文件共享

AIO Sandbox 把这些打包成一个 Docker 镜像，一键启动。

### 4.3 核心差异化价值

1. **All-in-One**：不是把多个工具放在一起，而是真正统一的执行环境
2. **统一文件系统**：这是与其他工具的本质区别
3. **MCP 协议原生**：预置 4 个 MCP 服务，AI Agent 可直接调用
4. **多语言 SDK**：Python/TypeScript/Go，降低接入门槛

---

## 五、洞察总结

### 5.1 核心发现

1. **"All-in-One"不只是口号**：AIO Sandbox 不是把浏览器、终端、VSCode 简单打包在一个 Docker 里，而是通过**统一文件系统**让它们真正协同工作。这是工程上的关键创新。

2. **MCP 是它的灵魂**：4 个预置 MCP 服务让 AI Agent 可以通过标准协议控制整个环境。对于 MCP 生态来说，AIO Sandbox 是一个功能完整的"MCP 服务端"。

3. **Playwright 位于底层**：AIO Sandbox 的浏览器自动化底层依赖 Playwright（通过 CDP 连接）。这说明 AIO Sandbox 是一个"基于现有工具的编排层"而非自研。

4. **中国区镜像**：提供了北京区域的 Docker 镜像（enterprise-public-cn-beijing.cr.volces.com），说明项目有中国用户群体。

5. **SDK 多语言支持**：Python/TypeScript/Go 三种语言的 SDK，意味着它可以服务各种技术栈的 AI Agent 开发者。

### 5.2 局限性

- **依赖 Docker**：需要 Docker 环境，无法在无 Docker 环境中运行
- **资源占用**：一个容器包含所有服务，资源消耗不小
- **没有真正的进程隔离**：虽然有 seccomp，但组件之间共享同一个容器
- **VNC 性能**：复杂 Web 应用通过 VNC 操控可能有延迟

### 5.3 架构评分

| 维度 | 评分 | 说明 |
|------|------|------|
| **架构创新** | ★★★★☆ | 统一文件系统是真正的工程创新 |
| **实用性** | ★★★★★ | All-in-One，一站式解决多种需求 |
| **MCP 集成** | ★★★★★ | 4 个预置 MCP 服务，原生支持 AI Agent |
| **部署体验** | ★★★★★ | Docker 一键启动，30 秒起步 |
| **生态扩展** | ★★★★☆ | SDK + MCP + Docker，扩展性强 |

**综合评价**：AIO Sandbox 是 AI Agent 开发领域的"瑞士军刀"环境。它通过统一文件系统让浏览器、终端、文件、VSCode、Jupyter 真正协同工作，解决了多工具协作时文件共享的痛点。4 个预置 MCP 服务和 Python/TS/Go 三语言 SDK 让 AI Agent 可以通过标准协议控制整个环境。对于需要"浏览器自动化 + 代码执行 + 文件操作"组合能力的 AI Agent 开发者来说，这是目前最完整的一站式方案。

---

*报告生成时间：2026-05-05 06:57 | 数据来源：GitHub README.md*