# NousResearch/hermes-agent 深度架构分析

> **分析日期**: 2026-05-05  
> **项目地址**: https://github.com/NousResearch/hermes-agent  
> **GitHub Stars**: ⭐130k+  
> **开发方**: Nous Research  
> **技术栈**: Python + TUI + RPC + FTS5  
> **许可证**: MIT  
> **项目定位**: 自主进化的 AI Agent——内置学习循环，从经验中创建技能，在使用中自我改进

---

## 一、项目概述

### 1.1 核心定位

Hermes Agent 是 **Nous Research 打造的自进化 AI Agent**：

> "The only agent with a built-in learning loop"

它的核心特点：
- **从经验中创建技能**：完成复杂任务后自动生成新技能
- **在使用中自我改进**：技能会随使用越来越好用
- **跨会话记忆**：搜索历史对话，建立对你的认知模型
- **运行在任何地方**：$5 VPS 到 GPU 集群，从 Telegram 随时接入

### 1.2 解决的问题

现有 Agent 的问题：
- **没有学习能力**：每次对话都是从零开始
- **绑定设备**：必须在电脑前
- **单一接口**：只能用 CLI

### 1.3 Hermes 的解法

```
经验 → 技能创建 → 技能改进 → 记忆积累 → 越来越懂你
```

不是"问答机器"，而是"会学习的助手"。

### 1.4 目标用户

- **需要真正助手的人**：不是每次都要解释背景
- **跨设备用户**：手机/电脑都要能接入
- **技术爱好者**：$5 VPS 就能跑
- **研究者**：需要研究友好的批量轨迹生成

### 1.4 关键指标

| 指标 | 数值 |
|------|------|
| GitHub Stars | ⭐130k+ |
| 模型支持 | 200+（OpenRouter） |
| 平台支持 | Telegram / Discord / Slack / WhatsApp / Signal / Email / CLI |
| 运行环境 | 本地 / Docker / SSH / Daytona / Singularity / Modal |
| 学习系统 | FTS5 全文搜索 + Honcho 用户建模 + agentskills.io |
| 许可证 | MIT |

---

## 二、核心架构

### 2.1 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Hermes Agent 架构                                │
│                                                                     │
│  用户接口层                                                          │
│  ├── CLI (TUI) ── 完整 TUI，多行编辑，命令补全                    │
│  ├── Telegram ── 手机上对话                                        │
│  ├── Discord ── 服务器里协作                                       │
│  ├── Slack / WhatsApp / Signal / Email                            │
│  └── `hermes gateway` 统一网关                                    │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                 核心 Agent 引擎                               │  │
│  │                                                              │  │
│  │  学习循环层                                                   │  │
│  │  ├── Agent-Curated Memory ── 周期性记忆                       │  │
│  │  ├── Skill Creation ── 复杂任务后自动创建技能                 │  │
│  │  ├── Skill Self-Improvement ── 技能在使用中进化               │  │
│  │  ├── FTS5 Session Search ── 全文搜索历史会话                  │  │
│  │  ├── Honcho User Modeling ── 持续建立用户认知模型             │  │
│  │  └── agentskills.io 兼容 ── 开放标准技能                    │  │
│  │                                                              │  │
│  │  执行层                                                       │  │
│  │  ├── Subagent 委派 ── 生成独立子 Agent 并行工作              │  │
│  │  ├── RPC Python Scripts ── 零 context 成本调用工具            │  │
│  │  └── Cron Scheduler ── 自然语言调度自动化                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    模型无关层 (Any Model)                     │  │
│  │                                                              │  │
│  │  Nous Portal / OpenRouter (200+ models) / NVIDIA NIM         │  │
│  │  Xiaomi MiMo / z.ai/GLM / Kimi/Moonshot / MiniMax            │  │
│  │  Hugging Face / OpenAI / 自定义 endpoint                     │  │
│  │                                                              │  │
│  │  `hermes model` 一键切换，不改代码                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                 运行环境层 (6 种终端后端)                      │  │
│  │                                                              │  │
│  │  local ── 本地运行                                            │  │
│  │  Docker ── 容器化                                            │  │
│  │  SSH ── 远程机器                                             │  │
│  │  Daytona ── 云端开发环境                                      │  │
│  │  Singularity ── HPC/超算                                      │  │
│  │  Modal ── serverless，闲置时休眠，几乎不花钱                  │  │
│  │                                                              │  │
│  │  一条命令：`hermes run --backend modal`                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心创新：内置学习循环

这是 Hermes 区别于所有其他 Agent 的关键：

```
任务完成 → 
  ├─ 创建技能（如果复杂）→ 
  │   └─ 技能库 +1
  │
  ├─ 技能自我改进（如果有用）→
  │   └─ 同一技能下次更好用
  │
  ├─ 记忆积累 →
  │   └─ 了解你的偏好/习惯/项目
  │
  └─ FTS5 索引 →
      └─ 下次可以搜索到这个经验
```

**Honcho 用户建模**：从每次对话中学习你是谁（通过 [plastic-labs/honcho](https://github.com/plastic-labs/honcho)）。

### 2.3 多平台网关

```
hermes gateway
  ├── Telegram Bot ── 手机上用
  ├── Discord Bot ── 服务器里用
  ├── Slack Bot ── 企业用
  ├── WhatsApp ── 日常用
  ├── Signal ── 隐私用
  └── Email ── 邮件用
```

一个 gateway 进程支持所有平台，从 Telegram 发消息，在 Discord 继续。

### 2.4 任何模型

```bash
hermes model                    # 交互式选择
hermes model openrouter:anthropic/claude-3-5-sonnet
hermes model nous:llama-3-70b
hermes model kimi:moonshot-v1-128k
```

支持的提供商（200+ 模型）：

| 提供商 | 代表模型 |
|--------|---------|
| OpenRouter | 200+ 模型可选 |
| Nous Portal | Nous 自有模型 |
| NVIDIA NIM | Nemotron |
| Xiaomi MiMo | 小米 MiMo |
| z.ai/GLM | 智谱 GLM |
| Kimi/Moonshot | Moonshot |
| MiniMax | MiniMax |
| Hugging Face | 开源模型 |
| OpenAI | GPT 系列 |
| 自定义 endpoint | 任何兼容 API |

### 2.5 运行在任何地方

| 后端 | 特点 | 成本 |
|------|------|------|
| **local** | 本地运行 | 你的机器 |
| **Docker** | 容器化 | 你的机器 |
| **SSH** | 远程机器 | 远程机器 |
| **Daytona** | 云端开发环境 | 按需 |
| **Singularity** | HPC/超算 | 机构资源 |
| **Modal** | serverless，休眠唤醒 | **闲置几乎不花钱** |

Modal 是亮点——Agent 环境在闲置时休眠，几乎不花钱。

---

## 三、高级功能

### 3.1 Cron 调度

用自然语言设置定时任务：

```bash
/hermes cron "每天早上 9 点给我发天气报告"
/hermes cron "每周一凌晨 3 点运行备份"
/hermes cron "每小时检查一次服务器状态"
```

### 3.2 Subagent 委派

```bash
/hermes delegate "research the latest AI news"  # 在后台并行搜索
/hermes delegate "update the documentation"      # 同时做文档
# 多个任务并行，不占用主对话 context
```

### 3.3 RPC Python Scripts

写 Python 脚本通过 RPC 调用工具，零 context 成本：

```python
# multi_step_pipeline.py
from hermes import tools

result = tools.search("latest AI papers")
tools.analyze(result)
tools.report(summary)
# 多步骤管道，一个 context 也不用占
```

### 3.4 研究功能

- **Batch Trajectory Generation**：批量生成轨迹
- **Atropos RL Environments**：强化学习环境
- **Trajectory Compression**：压缩轨迹用于训练下一代金具

### 3.5 OpenClaw 迁移

```bash
hermes claw migrate
```

有完整的 OpenClaw 迁移路径——之前用 OpenClaw 的用户可以平滑切换到 Hermes。

---

## 四、竞品对比

### 4.1 全局对比

| 工具 | 自学习 | 多平台 | 任何模型 | Stars |
|------|--------|--------|---------|-------|
| **Hermes Agent** | ✅ 内置学习循环 | ✅ 6 平台 | ✅ 200+ | ⭐130k |
| **OpenClaw** | ⚠️ Skills | ✅ 20+ | ✅ | 活跃 |
| **Claude Code** | ❌ | ❌ | ❌ | 官方 |
| **CrewAI** | ❌ | ❌ | ❌ | ⭐30k |

### 4.2 深度对比

#### vs OpenClaw

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **自学习** | ✅ 内置循环 | ⚠️ Skills |
| **多平台** | 6 个 | 20+ |
| **任何模型** | ✅ 200+ | ✅ |
| **Serverless** | ✅ Modal | ⚠️ |
| **OpenClaw 迁移** | ✅ | N/A |

**可以互补**：OpenClaw 擅长多平台消息，Hermes 擅长自主学习和任何模型。

#### vs 其他 Agent 框架

| 维度 | Hermes | 其他框架 |
|------|--------|---------|
| **内置学习** | ✅ 唯一 | ❌ |
| **多平台** | ✅ | ❌ |
| **成本** | $5 VPS 可跑 | 通常需要 GPU |
| **研究友好** | ✅ RL 环境 | ❌ |

### 4.3 核心差异化价值

1. **内置学习循环是真正创新**：不是"记得上次对话"，而是"从经验中创建技能并在下次改进"。这是质变。

2. **任何模型 + 任何平台**：不绑定任何生态，真正做到了"你的 AI 你做主"。

3. **Modal serverless 是成本革命**：$5 VPS 就能跑，闲置时几乎不花钱。这让 AI Agent 变成了人人都能负担的东西。

4. **OpenClaw 迁移路径**：说明项目方在积极吸引 OpenClaw 的成熟用户。

5. **130k stars 的影响力**：这是我们今天分析的最高 stars 项目，说明有大量用户在用。

---

## 五、洞察总结

### 5.1 核心发现

1. **"自进化"是 killer feature**：从经验中创建技能 + 技能自我改进，这是所有 AI Agent 的目标，Hermes 第一个真正做到了。

2. **Honcho 用户建模让 Agent 越来越懂你**：不只是"记得上次说了什么"，而是建立了用户认知模型，理解你的偏好、习惯、项目。

3. **"任何模型"是战略选择**：不绑定任何模型提供商，用户可以随时切换。这让 Hermes 成了一个"模型无关"的 Agent 平台。

4. **Modal serverless 是工程亮点**：闲置时休眠，几乎不花钱。这意味着你可以在 $5 VPS 上跑一个"永不眠"的 AI Agent。

5. **OpenClaw 迁移是聪明的市场策略**：OpenClaw 有成熟的用户群体，提供迁移路径可以直接获取这部分用户。

### 5. 2 局限性

- **自学习可能产生不良技能**：如果初始任务理解错误，创建的技能可能也有问题
- **FTS5 搜索质量依赖索引**：搜索不到可能是索引问题
- **多平台同步体验**：一条消息从 Telegram 到 Discord 的体验可能不一致
- **Python 依赖**：不是纯 JS/TS 实现

### 5.3 架构评分

| 维度 | 评分 | 说明 |
|------|------|------|
| **自学习能力** | ★★★★★ | 内置学习循环，唯一 |
| **多平台** | ★★★★★ | 6 个消息平台 + CLI |
| **模型无关** | ★★★★★ | 200+ 模型，任意切换 |
| **成本效益** | ★★★★★ | $5 VPS + Modal serverless |
| **研究友好** | ★★★★☆ | RL 环境 + 轨迹生成 |
| **生态** | ★★★★☆ | 130k stars，Discord 活跃 |

**综合评价**：Hermes Agent 是今天分析的所有项目中 **stars 最高**（130k）、**最具创新性**的项目。它的核心创新不是"又一个 Agent 框架"，而是真正的**内置学习循环**——从经验中创建技能，在使用中自我改进，跨会话记住你。这让它从"工具"变成了"助手"。Modal serverless 让成本降到最低，任何人都能负担。对于需要一个真正会学习的 AI 助手的人来说，Hermes Agent 值得关注。

---

*报告生成时间：2026-05-05 07:17 | 数据来源：GitHub README.md、hermes-agent.nousresearch.com/docs*