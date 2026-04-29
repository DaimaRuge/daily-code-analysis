# OpenClaw Agent Runtime 架构分析报告

> 2026-04-29 | OpenClaw Sub-Agent 与 Multi-Agent 运行时深度解析
> 数据来源：OpenClaw 官方文档（agent.md, multi-agent.md, subagents.md）

---

## 一、Agent Runtime 全景

OpenClaw 有三层 Agent 抽象：

```
┌────────────────────────────────────────────────────┐
│                   Gateway（进程）                    │
│  ┌──────────────────────────────────────────────┐  │
│  │              Agent A（主 Agent）               │  │
│  │  workspace/ SOUL.md  USER.md  AGENTS.md       │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐      │  │
│  │  │Sub-Agent │ │Sub-Agent │ │Sub-Agent │ ...  │  │
│  │  │  (run)   │ │  (run)   │ │  (run)   │      │  │
│  │  └──────────┘ └──────────┘ └──────────┘      │  │
│  └──────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │              Agent B（隔离 Agent）             │  │
│  │  workspace-b/ 独立 SOUL.md 独立 sessions/     │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

**三层关系**：

| 层 | 是什么 | 隔离级别 | 你的场景 |
|----|--------|---------|---------|
| **Gateway** | 一个 Node.js 进程 | 最高（进程级） | 你只有一个 |
| **Agent** | 一个独立"人格" | 高（workspace/session/auth 独立） | 你只有一个 `main` |
| **Sub-Agent** | Agent 派出的子任务 | 中（独立 session，可 fork context） | 每次分析项目都派 |

---

## 二、Sub-Agent（你的主力）

### 2.1 工作原理

```python
# 你触发（或 HEARTBEAT 自动）：
sessions_spawn(
    task="分析 bytedance/deer-flow 架构",
    context="isolated"   # 默认值
)

# → 创建一个子 session: agent:main:subagent:<uuid>
# → 子Agent 独立运行，有自己的上下文
# → 完成后自动 announce 结果给主 session
```

### 2.2 两种上下文模式

| 模式 | Context 大小 | 用途 | Token 成本 |
|------|-------------|------|-----------|
| **isolated**（默认） | 只有 task 文本 | 独立分析、不依赖对话历史 | 低 |
| **fork** | 复制主 session 全部上下文 | 需要主对话中的工具结果/指令 | 高 |

**你的场景**：分析 GitHub 项目 → 用 `isolated`（默认）。任务描述里写清楚就行，不需要主 session 的历史。

### 2.3 Sub-Agent 的能力边界

默认情况下 Sub-Agent **没有** session 工具（不能读其他 session），但拥有：
- `exec` / `read` / `write` / `web_fetch` / `web_search` / `browser` 等
- 可以用不同模型（`model` 参数覆盖）
- 可以设超时（`runTimeoutSeconds`）

你今天的三个子分析都用了默认配置：
```
gemini-cli:  4分42秒, 118k tokens
ruflo:       5分21秒, 105k tokens  
deer-flow:   5分54秒, 112k tokens
```

### 2.4 并行限制

```
主 Agent: 最多 4 个并发 Sub-Agent
总并发: 8 个（含跨 Agent）
```

你的自动流水线一次只派一个，完全不会冲突。

---

## 三、Multi-Agent（你不需要）

这是给**多用户/多渠道**场景用的。比如一个人用 WhatsApp、另一个用 Telegram，各自独立的 workspace 和人格。

```json
{
  "agents": {
    "list": [
      { "id": "群哥", "workspace": "~/.openclaw/workspace" },
      { "id": "同事", "workspace": "~/.openclaw/workspace-colleague" }
    ]
  },
  "bindings": [
    { "agentId": "群哥", "match": { "channel": "telegram" } },
    { "agentId": "同事", "match": { "channel": "whatsapp" } }
  ]
}
```

你只有一个用户（群哥）+ 一个 channel（webchat），所以**不需要多 Agent**。`main` Agent 足够。

---

## 四、Session 路由

每条消息的路由链路：

```
消息进入 → channel → accountId → binding → Agent ID → Session Key
                                                        ↓
                                            agent:main:<mainKey>
                                            agent:main:subagent:<uuid>
```

Sub-Agent 完成后：
```
Sub-Agent finish → announce → requester chat channel → 你看到结果
```

---

## 五、你的架构总结

```
OpenClaw Gateway（1个进程）
  │
  └── Agent: main（你唯一的 Agent）
        │
        ├── 主 Session（跟你对话）
        │     ├── 读 MEMORY.md、HEARTBEAT.md
        │     ├── 心跳任务：同步报告、抓趋势、查天气
        │     └── 派 Sub-Agent 分析项目
        │
        └── Sub-Agent（每次分析一个）
              ├── 独立 session
              ├── isolated context（不继承对话）
              ├── 完成 → announce → 我收到 → 同步 → 告诉你
              └── 每天 09:00 自动触发
```

**关键特性**：
- ✅ Sub-Agent 互不干扰（独立 session）
- ✅ 完成自动通知（push-based，不需要轮询）
- ✅ 可以设超时、指定模型
- ✅ 失败不会影响主 Agent
- ✅ HEARTBEAT 全自动编排

---

*报告基于 OpenClaw 官方文档 agent.md / multi-agent.md / subagents.md。2026-04-29 编制。*
