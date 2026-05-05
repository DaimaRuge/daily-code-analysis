# DeepSeek-V4 / V4.1 深度分析报告

> 2026-04-28 | 全面解析 DeepSeek-V4 系列能力、架构与训练方法
> 数据来源：DeepSeek-V4 技术报告 (DeepSeek_V4.pdf, 58页)、HuggingFace Model Card、DeepSeek API 官方文档

---

## 一、概述

DeepSeek-V4 预览版于 **2026年4月24日** 正式发布并开源，包含两个 MoE 模型：

| 模型 | 总参数量 | 激活参数量 | 上下文长度 | 精度 | 下载量 (HF) | Likes |
|------|---------|-----------|-----------|------|------------|-------|
| **DeepSeek-V4-Pro** | 1.6T | 49B | 1M | FP4+FP8 Mixed | 137,784 | 2,998 |
| **DeepSeek-V4-Flash** | 284B | 13B | 1M | FP4+FP8 Mixed | 65,743 | 774 |

> ⚠️ **关于 DeepSeek-V4.1**：截至2026年4月28日，官方渠道（API 文档、HuggingFace、技术报告）**未提及 V4.1**。最新模型为 V4 预览版（4月24日发布），V4 本身已深度优化 Agent 能力。报告中 V4.1 相关分析基于 V4 的 Agent 优化特性推断。

---

## 二、DeepSeek-V4 能力全景

### 2.1 三种推理模式

| 模式 | 特性 | 使用场景 | 输出格式 |
|------|------|---------|---------|
| **Non-think** | 快速直觉响应 | 日常任务、低风险决策 | `<｜end▁of▁thinking｜>` |
| **Think High** | 有意识逻辑分析，较慢但更准确 | 复杂问题求解、规划 | `... response` |
| **Think Max** | 推理推至极限 | 探索模型推理边界 | 特殊 system prompt + `... response` |

### 2.2 V4-Pro-Max vs 前沿模型（核心基准）

| 基准 (Metric) | Opus 4.6 Max | GPT-5.4 xHigh | Gemini 3.1 Pro High | **DS-V4-Pro Max** | 排名 |
|---|---|---|---|---|---|
| MMLU-Pro (EM) | 89.1 | 87.5 | **91.0** | 87.5 | 4th |
| SimpleQA-Verified | 46.2 | 45.3 | **75.6** | 57.9 | 3rd |
| Chinese-SimpleQA | 76.4 | 76.8 | 85.9 | **84.4** | 2nd |
| GPQA Diamond | 91.3 | 93.0 | **94.3** | 90.1 | 5th |
| HLE | 40.0 | 39.8 | **44.4** | 37.7 | 5th |
| **LiveCodeBench** | 88.8 | - | 91.7 | **93.5** 🏆 | **1st** |
| **Codeforces (Rating)** | - | 3168 | 3052 | **3206** 🏆 | **1st** |
| HMMT 2026 Feb | 96.2 | **97.7** | 94.7 | 95.2 | 3rd |
| IMOAnswerBench | 75.3 | **91.4** | 81.0 | 89.8 | 2nd |
| **Apex Shortlist** | 85.9 | 78.1 | 89.1 | **90.2** 🏆 | **1st** |

**Agent 能力**：

| 基准 | Opus 4.6 Max | GPT-5.4 xHigh | Gemini 3.1 Pro | **DS-V4-Pro Max** |
|---|---|---|---|---|
| Terminal Bench 2.0 | 65.4 | **75.1** | 68.5 | 67.9 |
| SWE Verified (Resolved) | 80.8 | - | 80.6 | 80.6 |
| SWE Pro (Resolved) | 57.3 | 57.7 | 54.2 | 55.4 |
| SWE Multilingual | 77.5 | - | - | 76.2 |
| BrowseComp | 83.7 | 82.7 | **85.9** | 83.4 |
| MCPAtlas Public | **73.8** | 67.2 | 69.2 | 73.6 |
| Toolathlon | 47.2 | **54.6** | 48.8 | 51.8 |

### 2.3 三模式内部对比

| 基准 | V4-Flash Non-Think | V4-Flash High | V4-Flash Max | V4-Pro Non-Think | V4-Pro High | V4-Pro Max |
|---|---|---|---|---|---|---|
| GPQA Diamond | 71.2 | 87.4 | 88.1 | 72.9 | 89.1 | **90.1** |
| HLE | 8.1 | 29.4 | 34.8 | 7.7 | 34.5 | **37.7** |
| LiveCodeBench | 55.2 | 88.4 | 91.6 | 56.8 | 89.8 | **93.5** |
| Codeforces | - | 2816 | **3052** | - | 2919 | **3206** |
| SWE Verified | 55.9 | 75.0 | 79.0 | 52.7 | 79.6 | **80.6** |

### 2.4 关键发现

1. **代码能力世界顶级**：V4-Pro-Max 在 LiveCodeBench (93.5) 和 Codeforces (3206) 双双登顶
2. **推理接近闭源前沿**：GPQA Diamond 90.1 (vs 94.3 Gemini)，HMMT 95.2 (vs 97.7 Opus)
3. **中文能力卓越**：Chinese-SimpleQA 84.4，仅次于 Gemini 的 85.9
4. **Agent 能力跨越式提升**：SWE Verified 80.6 达到开源最佳，Terminal Bench 67.9
5. **Flash 性价比极高**：284B/13B 的 Flash 在推理任务上几乎达到 Pro 水平，价格仅为 1/12
6. **长上下文王者**：1M 上下文 + 高效的 KV Cache 使其在 LongBench-V2 (51.5) 等任务领先

---

## 三、架构创新深度解析

### 3.1 模型配置对比

| 配置 | DeepSeek-V4-Flash | DeepSeek-V4-Pro |
|------|-------------------|-----------------|
| Transformer 层数 | 43 | 61 |
| 隐藏维度 d | 4096 | 7168 |
| 注意力头数 n_h | 64 | 128 |
| 头维度 c | 512 | 512 |
| Query 压缩维度 d_c | 1024 | 1536 |
| Output 投影组数 g | 8 | 16 |
| 中间注意力输出维度 d_g | 1024 | 1024 |
| CSA 压缩率 m | 4 | 4 |
| HCA 压缩率 m' | 128 | 128 |
| CSA Top-k | 512 | 1024 |
| CSA Indexer 头数 | 64 | 64 |
| 滑动窗口大小 | 128 | 128 |
| 路由专家数 | 256 | 256 |
| 共享专家数 | 1 | 1 |
| 每 token 激活专家 | 6 | 8 |
| 专家中间维度 | 2048 | 2048 |
| mHC 扩展因子 n_hc | 4 | 4 |

### 3.2 核心创新一：混合注意力架构 (Hybrid CSA + HCA)

#### CSA (Compressed Sparse Attention)

**流程**：
1. **KV 压缩**：每 m=4 个 token 压缩为 1 个压缩 KV entry（实际上从 2m 个 entry 压缩，因为有重叠）
2. **Lightning Indexer**：低秩方式生成 indexer queries，计算 index scores
3. **Top-k 稀疏选择**：选择 top-k 个压缩 KV entries
4. **Shared KV MQA**：选中的 KV entries 同时作为 Key 和 Value，Multi-Query Attention
5. **Grouped Output Projection**：将 n_h 个输出分组投影，减少计算量

**关键技术**：
- 重叠压缩：每个压缩块与相邻块共享原始 KV entries
- 学习的位置偏置 B_a, B_b
- 额外滑动窗口分支 (128 tokens) 增强局部依赖
- Attention Sink 技巧控制注意力分布
- Partial RoPE (最后64维)实现相对位置编码
- QK Norm 防止注意力 logits 爆炸

#### HCA (Heavily Compressed Attention)

**流程**：
1. **激进压缩**：m'=128 个 token 压缩为 1 个 KV entry
2. **密集注意力**：不做稀疏选择，query 关注所有压缩 KV entries
3. **Shared KV MQA + Grouped Output Projection**（与 CSA 相同）
4. 额外滑动窗口分支 (128 tokens)

#### 效率对比（1M 上下文）

| 指标 | DeepSeek-V3.2 | DeepSeek-V4-Pro | **压缩比** |
|------|--------------|----------------|----------|
| 单 token FLOPs | 基准 | 27% | **3.7× 降低** |
| KV Cache 大小 | 基准 | 10% | **9.5× 缩小** |
| DeepSeek-V4-Flash 单 token FLOPs | - | 10% | **9.8× 降低** |
| DeepSeek-V4-Flash KV Cache | - | 7% | **13.7× 缩小** |

### 3.3 核心创新二：流形约束超连接 (mHC)

**问题**：标准 Hyper-Connections (HC) 在堆叠多层时出现数值不稳定

**解决方案**：
- 将残差映射矩阵 B_l 约束到 **Birkhoff 多面体**（双随机矩阵流形）
- 约束条件：B1 = 1, 1^T B = 1^T, B ≥ 0
- 通过 **Sinkhorn-Knopp 算法** 迭代投影
- 输入/输出变换 A_l, C_l 通过 Sigmoid 约束为非负有界

**动态参数化**：
- 参数分解为动态（输入依赖）+ 静态（与输入无关）两部分
- 可学习的 gating factors 初始化为小值
- 扩展因子 n_hc = 4，Sinkhorn 迭代 t_max = 20

**效果**：
- 训练稳定性显著提升
- 总 wall-time 开销仅 **6.7%**（overlapped 1F1B pipeline）

### 3.4 核心创新三：Muon 优化器

**适用模块**：大多数参数（除 Embedding、Prediction Head、RMSNorm、mHC 静态偏置外）

**算法**：
1. 动量累积：M_t = μ·M_{t-1} + G_t
2. Nesterov 加速梯度
3. **混合 Newton-Schulz 迭代**：前8步快速收敛 → 后2步精确稳定
4. 更新 RMS 缩放至 0.18（复用 AdamW 学习率配置）

**关键参数**：
- 动量 = 0.95
- Weight decay = 0.1
- 混合 N-S 阶段系数：(3.4445, -4.7750, 2.0315) → (2, -1.5, 0.5)

**为什么不用 QK-Clip**：CSA/HCA 中的 QK Norm 已经有效防止注意力 logits 爆炸

---

## 四、训练方法论

### 4.1 预训练

| 配置 | DeepSeek-V4-Flash | DeepSeek-V4-Pro |
|------|-------------------|-----------------|
| 训练 Token 数 | 32T | 33T |
| 最大 Batch Size (tokens) | 75.5M | 94.4M |
| Peak LR | 2.7×10⁻⁴ | 2.0×10⁻⁴ |
| End LR | 2.7×10⁻⁵ | 2.0×10⁻⁵ |
| Warmup Steps | 2000 | 2000 |
| 序列长度调度 | 4K→16K→64K→1M | 4K→16K→64K→1M |
| 稀疏注意力引入 | 1T tokens 后 | 更长密集阶段后 |

**训练稳定性增强**：
- **Anticipatory Routing**：解耦主干网络和路由网络的同步更新（使用历史参数计算路由），仅在 loss spike 时自动激活
- **SwiGLU Clamping**：线性部分 [-10, 10]，门控上限 10
- **样本级注意力掩码**（非文档级）
- 辅助损失自由的负载均衡 + 轻量序列级平衡损失 (weight=0.0001)

**数据**：
- 32T+ 多样化高质量 token
- 在 V3 数据基础上增强：过滤自动生成/模板内容、增加 Agent 数据（mid-training）、多语言数据规模扩大
- Tokenizer：128K 词汇表，Token-splitting + FIM 策略

### 4.2 后训练 Pipeline（两阶段范式）

#### 阶段一：专家培养 (Specialist Training)

**流程**：
1. **SFT**（监督微调）：高质量领域特定数据建立基础能力
2. **RL with GRPO**（组相对策略优化）：领域特定 prompt + 奖励信号

**关键创新**：
- **生成式奖励模型 (GRM)**：替代传统标量 RM，直接对 GRM 做 RL 优化
- **三种推理模式独立训练**：不同模式使用不同的长度惩罚和上下文窗口
- **FP4 量化感知训练 (QAT)**：
  - MoE 专家权重量化为 FP4 (MXFP4)
  - CSA Indexer 中的 QK 路径量化为 FP4，加速长上下文注意力
  - Index scores 从 FP32→BF16，top-k selector 2倍加速，保持 99.7% recall

#### 阶段二：统一整合 (On-Policy Distillation)

**OPD (On-Policy Distillation)**：
- 统一模型作为 Student
- 多个领域专家作为 Teacher
- 优化 Reverse KL Loss，将多领域能力整合到单一模型
- 使用全词表 teacher scheduling

### 4.3 基础设施亮点

- **细粒度 Expert Parallelism (EP)**：Wave-based scheduling，计算-通信完全重叠，理论加速 1.92×
- **TileLang**：领域特定语言开发融合 kernel，SMT Solver 辅助整数分析
- **Batch-Invariant & Deterministic Kernels**：保证训练可复现性，Pre/Post-training/Inference 三阶段 bitwise 对齐
- **DualPipe 1F1B** 适配 mHC 增加的流水线通信

---

## 五、推理优化

### 5.1 KV Cache 管理

**异构 KV Cache 布局**：
- **Classical KV Cache**：存储 CSA/HCA 压缩 KV entries
- **State Cache**：存储 SWA KV 和未完成压缩的 tail tokens
- 每个 cache block 覆盖 lcm(m, m') = lcm(4, 128) = 128 个原始 tokens

**磁盘 KV Cache**：
- 共享前缀请求消除重复 Prefill
- SWA 三种策略：Full Caching / Periodic Checkpointing / Zero Caching

### 5.2 精度优化

| 组件 | 精度 |
|------|------|
| MoE 专家权重 | FP4 |
| 其他权重 | FP8 |
| RoPE 维度 (KV) | BF16 |
| 非 RoPE KV 维度 | FP8 |
| CSA Indexer QK 计算 | FP4 |

**理论上限**：FP4×FP8 在未来硬件上可有 1/3 额外效率提升

---

## 六、DeepSeek-V4.1 — Agent 专项优化分析

### 6.1 V4 已内建的 Agent 能力

DeepSeek-V4 在发布之初即针对 Agent 场景进行了深度优化：

1. **主流 Agent 框架适配**：针对 Claude Code、OpenClaw、OpenCode、CodeBuddy 等进行专项优化
2. **DeepSeek 内部使用**：已成为公司内部员工的 Agentic Coding 模型，据反馈：
   - 使用体验 **优于 Sonnet 4.5**
   - 交付质量 **接近 Opus 4.6 非思考模式**
   - 与 Opus 4.6 思考模式仍有差距
3. **R&D 编码基准**：Pro-Max 通过率 67%，超越 Sonnet 4.5 (47%)，接近 Opus 4.5 (70%)
4. **内部调查** (N=85)：52% 认为 V4-Pro 可作默认编码模型，39% 倾向同意

### 6.2 若 V4.1 存在，可能的优化方向

基于技术报告 "Future Directions" 章节和架构特点推测：

1. **长程多轮 Agent 任务**：报告明确指出将继续迭代长程 Agent 任务
2. **模型稀疏化新维度**：更稀疏的 Embedding 模块 (Cheng et al., 2026)
3. **低延迟架构与系统技术**：提升长上下文部署的响应性
4. **多模态能力整合**：报告明确提到正在为模型加入多模态能力
5. **OPD 精炼**：通过更多轮次的 On-Policy Distillation 提升 Agent 一致性
6. **GRPO 奖励模型增强**：更精细的 Agent 任务奖励设计
7. **架构简化**：报告承认当前架构相对复杂，未来方向是简化设计

---

## 七、价格与可用性

| 模型 | 输入（缓存命中） | 输入（缓存未命中） | 输出 |
|------|----------------|-------------------|------|
| deepseek-v4-flash | 0.02 元/M tokens | 1 元/M tokens | 2 元/M tokens |
| deepseek-v4-pro | 0.1 元/M tokens* | 12 元/M tokens* | 24 元/M tokens* |

> *deepseek-v4-pro 限时2.5折优惠至 2026/05/05 23:59，原价：输入4元、48元/M；输出96元/M

**兼容性**：
- OpenAI ChatCompletions 接口
- Anthropic 接口
- 旧模型名 deepseek-chat/deepseek-reasoner 将于 **2026-07-24 停止使用**

---

## 八、总结

### 优势
- 🏆 **代码能力世界第一**：LiveCodeBench 93.5, Codeforces 3206
- 🧠 **推理接近闭源 SOTA**：GPQA 90.1, IMOAnswerBench 89.8
- 📚 **世界知识开源最强**：大幅领先其他开源模型
- 🚀 **长上下文效率革命**：1M 上下文 FLOPs 降至 V3.2 的 27%，KV Cache缩至 10%
- 💰 **Flash 极致性价比**：284B/13B 参数，推理能力接近 Pro，价格仅 1/12
- 🔧 **成熟的训练基础设施**：TileLang、FP4 QAT、MegaMoE mega-kernel
- 🇨🇳 **中文能力顶尖**：Chinese-SimpleQA 84.4

### 局限
- 知识任务仍落后 Gemini 3.1 Pro（SimpleQA 57.9 vs 75.6）
- Agent 能力落后 GPT-5.4（Terminal Bench 67.9 vs 75.1）
- 架构相对复杂，未来需简化
- HLE 37.7 仍有较大提升空间
- V4.1 尚未正式发布（或不存在独立版本号）

### V4.1 展望
DeepSeek-V4 在发布时已深度整合 Agent 优化，**可能不需要单独 V4.1 版本号**。未来迭代方向包括：多模态整合、长程 Agent 优化、低延迟推理、模型稀疏化新范式。

---

*报告基于 DeepSeek-V4 Technical Report (58页)、HuggingFace 官方 Model Card、DeepSeek API 官方文档。2026-04-28 编制。*
