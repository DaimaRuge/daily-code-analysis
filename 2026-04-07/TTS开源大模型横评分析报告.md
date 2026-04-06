# 开源 TTS 大模型源代码分析报告

> 生成时间：2026-04-07  
> 数据来源：电磁波Studio 横向评测视频

## 📋 目录

1. [项目概览](#1-项目概览)
2. [LongCat-AudioDiT 深度分析](#2-longcat-audiodit-深度分析)
3. [Qwen3-TTS 深度分析](#3-qwen3-tts-深度分析)
4. [Fish Audio S2 Pro 深度分析](#4-fish-audio-s2-pro-深度分析)
5. [技术架构对比](#5-技术架构对比)
6. [性能评测对比](#6-性能评测对比)
7. [适用场景分析](#7-适用场景分析)
8. [结论与建议](#8-结论与建议)

---

## 1. 项目概览

| 项目 | 开发者 | 参数规模 | GitHub 仓库 | 许可证 |
|------|--------|----------|-------------|--------|
| **LongCat-AudioDiT** | 美团 (Meituan) | 1B / 3.5B | [meituan-longcat/LongCat-AudioDiT](https://github.com/meituan-longcat/LongCat-AudioDiT) | MIT |
| **Qwen3-TTS** | 阿里云 (Alibaba) | 0.6B / 1.7B | [QwenLM/Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS) | Apache 2.0 |
| **Fish Audio S2 Pro** | Fish Audio | 4B | [fishaudio/fish-speech](https://github.com/fishaudio/fish-speech) | Fish Audio Research |

### 评测结论回顾

| 排名 | 模型 | 生成速度 | 音质评价 | 音色迁移 |
|------|------|----------|----------|----------|
| 🥇 | LongCat-AudioDiT | 6秒 | 清晰 | 效果好 |
| 🥈 | Qwen3-TTS | 33秒 | 稳定 | 适中 |
| 🥉 | Fish Audio S2 Pro | >1分钟 | 最好 | 优秀 |

---

## 2. LongCat-AudioDiT 深度分析

### 2.1 项目信息

- **GitHub**: https://github.com/meituan-longcat/LongCat-AudioDiT
- **HuggingFace**: [meituan-longcat/LongCat-AudioDiT-1B](https://huggingface.co/meituan-longcat/LongCat-AudioDiT-1B), [3.5B](https://huggingface.co/meituan-longcat/LongCat-AudioDiT-3.5B)
- **论文**: [arXiv:2603.29339](https://arxiv.org/abs/2603.29339)
- **许可证**: MIT License

### 2.2 核心架构

LongCat-AudioDiT 是一款**基于扩散模型的高保真文本转语音模型**，其核心创新在于直接在波形潜空间（waveform latent space）进行扩散生成。

```
┌─────────────────────────────────────────────────────────────┐
│                    LongCat-AudioDiT 架构                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   文本输入 ──→ [文本编码器] ──→ [扩散骨干网络] ──→ [Wav-VAE] ──→ 音频输出
│                              ↑                              │
│                        自适应投影引导                          │
│                     (Adaptive Projection Guidance)           │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 关键技术点

#### 核心创新 1：波形潜空间扩散
- **传统方法**：依赖梅尔频谱（mel-spectrogram）等中间声学表征
- **LongCat 方法**：直接在波形潜空间进行扩散，简化 TTS 流水线
- **优势**：有效避免误差累积，减少多阶段训练流程

#### 核心创新 2：自适应投影引导（APG）
- 识别并纠正了长期存在的训练-推理不匹配问题
- 用自适应投影引导替代传统的无分类器引导（CFG）
- 显著提升生成质量

### 2.4 性能指标

| 模型 | 中文 CER ↓ | 中文 SIM ↑ | 英文 WER ↓ | 英文 SIM ↑ | Hard CER ↓ | Hard SIM ↑ |
|------|------------|------------|------------|------------|------------|------------|
| GT | 1.26 | 0.755 | 2.14 | 0.734 | - | - |
| **LongCat-3.5B** | **1.09** | **0.818** | **1.50** | **0.786** | **6.04** | **0.797** |
| LongCat-1B | 1.18 | 0.812 | 1.78 | 0.762 | 6.33 | 0.787 |

> SIM = 说话人相似度（Speaker Similarity），CER/WER = 字符/词错误率

### 2.5 源代码结构

```
LongCat-AudioDiT/
├── inference.py           # 推理脚本
├── batch_inference.py    # 批量推理脚本
├── requirements.txt      # 依赖
├── LICENSE               # MIT 许可证
└── assets/
    ├── architecture.png  # 架构图
    └── prompt.wav        # 示例音频
```

### 2.6 快速使用

```python
import audiodit
from audiodit import AudioDiTModel
from transformers import AutoTokenizer
import torch, soundfile as sf

# 加载模型
model = AudioDiTModel.from_pretrained("meituan-longcat/LongCat-AudioDiT-1B").to("cuda")
model.eval()

tokenizer = AutoTokenizer.from_pretrained(model.config.text_encoder_model)

# 零样本语音合成
inputs = tokenizer(["今天晴暖转阴雨，空气质量优至良，空气相对湿度较低。"], 
                   padding="longest", return_tensors="pt")
output = model(
    input_ids=inputs.input_ids,
    attention_mask=inputs.attention_mask,
    duration=62,
    steps=16,
    cfg_strength=4.0,
    guidance_method="cfg",
)
sf.write("output.wav", output.waveform.squeeze().cpu().numpy(), 24000)
```

### 2.7 语音克隆

```python
import librosa, torch

# 加载提示音频
audio, _ = librosa.load("prompt.wav", sr=24000, mono=True)
prompt_wav = torch.from_numpy(audio).unsqueeze(0).unsqueeze(0)

# 克隆语音
prompt_text = "小偷却一点也不气馁，继续在抽屉里翻找。"
gen_text = "今天晴暖转阴雨，空气质量优至良，空气相对湿度较低。"
inputs = tokenizer([f"{prompt_text} {gen_text}"], padding="longest", return_tensors="pt")

output = model(
    input_ids=inputs.input_ids,
    attention_mask=inputs.attention_mask,
    prompt_audio=prompt_wav,
    duration=138,
    steps=16,
    cfg_strength=4.0,
    guidance_method="apg",
)
```

---

## 3. Qwen3-TTS 深度分析

### 3.1 项目信息

- **GitHub**: https://github.com/QwenLM/Qwen3-TTS
- **HuggingFace**: [Qwen/Qwen3-TTS](https://huggingface.co/collections/Qwen/qwen3-tts)
- **ModelScope**: [Qwen/Qwen3-TTS](https://modelscope.cn/collections/Qwen/Qwen3-TTS)
- **论文**: [arXiv:2601.15621](https://arxiv.org/abs/2601.15621)
- **许可证**: Apache 2.0

### 3.2 核心架构

Qwen3-TTS 是阿里云 Qwen 团队开发的**多语言文本转语音模型系列**，采用离散多码本 LM（Multi-Codebook LM）架构。

```
┌─────────────────────────────────────────────────────────────┐
│                      Qwen3-TTS 架构                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   文本 ──→ [Qwen3-TTS-Tokenizer-12Hz] ──→ [离散码本 LM] ──→ 语音
│                                                             │
│   支持：语音设计 / 语音克隆 / 流式生成 / 自然语言控制         │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 已发布模型

| 模型 | 参数量 | 功能 | 语言支持 | 流式 | 指令控制 |
|------|--------|------|----------|------|----------|
| Qwen3-TTS-12Hz-1.7B-VoiceDesign | 1.7B | 语音设计 | 10种 | ✅ | ✅ |
| Qwen3-TTS-12Hz-1.7B-CustomVoice | 1.7B | 自定义音色 | 10种 | ✅ | ✅ |
| Qwen3-TTS-12Hz-1.7B-Base | 1.7B | 基础模型/微调 | 10种 | ✅ | |
| Qwen3-TTS-12Hz-0.6B-CustomVoice | 0.6B | 自定义音色 | 10种 | ✅ | |
| Qwen3-TTS-12Hz-0.6B-Base | 0.6B | 基础模型/微调 | 10种 | ✅ | |

### 3.4 核心技术特点

#### 特点 1：Qwen3-TTS-Tokenizer-12Hz
- 自研语音分词器，实现高效声学压缩
- 高维语义建模，完全保留副语言信息和声学环境特征
- 轻量级非 DiT 架构实现高速、高保真语音重建

#### 特点 2：双轨混合流式生成架构
- 单模型同时支持流式和非流式生成
- 输入单个字符即可输出第一个音频包
- 端到端合成延迟低至 **97ms**

#### 特点 3：通用端到端架构
- 离散多码本 LM 实现全信息端到端语音建模
- 完全绕过传统 LM+DiT 的信息瓶颈和级联误差

### 3.5 支持的音色

| 音色名称 | 描述 | 母语 |
|----------|------|------|
| Vivian | 明亮、略带锐利的年轻女声 | 中文 |
| Serena | 温暖、柔和的年轻女声 | 中文 |
| Uncle_Fu | 成熟男声，低沉圆润 | 中文 |
| Dylan | 北京青年男声，清澈自然 | 中文(北京话) |
| Eric | 活泼成都男声，略带沙哑 | 中文(四川话) |
| Ryan | 动感男声，节奏感强 | 英文 |
| Aiden | 阳光美国男声，清亮中音 | 英文 |
| Ono_Anna | 俏皮日本女声，轻快 | 日文 |
| Sohee | 温暖韩国女声，情感丰富 | 韩文 |

### 3.6 快速使用

```python
import torch
import soundfile as sf
from qwen_tts import Qwen3TTSModel

model = Qwen3TTSModel.from_pretrained(
    "Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice",
    device_map="cuda:0",
    dtype=torch.bfloat16,
    attn_implementation="flash_attention_2",
)

# 自定义音色生成
wavs, sr = model.generate_custom_voice(
    text="其实我真的有发现，我是一个特别善于观察别人情绪的人。",
    language="Chinese",
    speaker="Vivian",
    instruct="用特别愤怒的语气说",
)
sf.write("output.wav", wavs[0], sr)
```

### 3.7 语音设计

```python
# 语音设计 - 通过自然语言描述生成音色
wavs, sr = model.generate_voice_design(
    text="哥哥，你回来啦，人家等了你好久好久了，要抱抱！",
    language="Chinese",
    instruct="体现撒娇稚嫩的萝莉女声，音调偏高且起伏明显，营造出黏人、做作又刻意卖萌的听觉效果。",
)
```

### 3.8 语音克隆

```python
ref_audio = "https://qianwen-res.oss-cn-beijing.aliyuncs.com/Qwen3-TTS-Repo/clone.wav"
ref_text = "Okay. Yeah. I resent you. I love you. I respect you."

wavs, sr = model.generate_voice_clone(
    text="I am solving the equation: x = [-b ± √(b²-4ac)] / 2a?",
    language="English",
    ref_audio=ref_audio,
    ref_text=ref_text,
)
```

---

## 4. Fish Audio S2 Pro 深度分析

### 4.1 项目信息

- **GitHub**: https://github.com/fishaudio/fish-speech
- **HuggingFace**: [fishaudio/s2-pro](https://huggingface.co/fishaudio/s2-pro)
- **官网**: [fish.audio](https://fish.audio/)
- **论文**: [arXiv:2603.08823](https://arxiv.org/abs/2603.08823)
- **许可证**: Fish Audio Research License

### 4.2 核心架构

Fish Audio S2 Pro 采用**双自回归（Dual-AR）主从架构**，结合 RVQ 音频编解码器。

```
┌─────────────────────────────────────────────────────────────┐
│                    Fish Audio S2 Pro 架构                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   文本 + 标签 ──→ [慢速 AR (4B)] ──→ 主语义码本              │
│                              │                              │
│                    [快速 AR (400M)] ──→ 9个残差码本          │
│                              ↓                              │
│                    [RVQ Codec (10码本, ~21Hz)] ──→ 音频     │
│                                                             │
│   使用 GRPO (Group Relative Policy Optimization) 对齐     │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 双自回归架构详解

- **慢速 AR（4B 参数）**：沿时间轴运作，预测主要语义码本
- **快速 AR（400M 参数）**：每步生成剩余 9 个残差码本
- **非对称设计**：峰值音频保真度 + 显著提升推理速度

### 4.4 性能指标

| 评测基准 | Fish Audio S2 | 备注 |
|----------|---------------|------|
| Seed-TTS Eval — WER (中文) | **0.54%** (最佳) | 优于 Qwen3-TTS (0.77) |
| Seed-TTS Eval — WER (英文) | **0.99%** (最佳) | 优于 Qwen3-TTS (1.24) |
| Audio Turing Test (with instruction) | **0.515** | 超 Seed-TTS 24%，超 MiniMax 33% |
| EmergentTTS-Eval — Win Rate | **81.88%** (最高) | |
| Fish Instruction Benchmark — TAR | **93.3%** | |
| Fish Instruction Benchmark — Quality | **4.51/5.0** | |

### 4.5 核心技术特点

#### 特点 1：细粒度情感控制
- 使用 `[tag]` 语法在文本任意位置嵌入情感指令
- 支持 **15,000+** 独特标签
- 支持自由形式文本描述

**情感标签示例**：
```
[pause] [emphasis] [laughing] [inhale] [chuckle]
[whisper] [screaming] [shouting] [angry] [sad]
[speaking in a small voice] [professional broadcast tone]
```

#### 特点 2：GRPO 对齐技术
- 使用 Group Relative Policy Optimization 进行后训练对齐
- 多维奖励信号：
  - 语义准确性
  - 指令遵从度
  - 声学偏好评分
  - 音色相似度

#### 特点 3：SGLang 原生支持
- 连续批处理（Continuous Batching）
- Paged KV Cache
- CUDA Graph
- RadixAttention 前缀缓存

### 4.6 性能数据（H200 GPU 单卡）

| 指标 | 数值 |
|------|------|
| 实时因子 (RTF) | 0.195 |
| 首音频时间 (TTFA) | ~100ms |
| 吞吐量 | 3,000+ acoustic tokens/s |

### 4.7 支持语言

**第一梯队**：日语 (ja)、英语 (en)、中文 (zh)

**第二梯队**：韩语 (ko)、西班牙语 (es)、葡萄牙语 (pt)、阿拉伯语 (ar)、俄语 (ru)、法语 (fr)、德语 (de)

**全球覆盖**：80+ 语言

### 4.8 快速使用

```python
# 详细使用方法请参考官方文档
# https://speech.fish.audio/install/
```

---

## 5. 技术架构对比

### 5.1 架构类型对比

| 特性 | LongCat-AudioDiT | Qwen3-TTS | Fish Audio S2 Pro |
|------|------------------|-----------|-------------------|
| **生成方式** | 扩散模型 (Diffusion) | 离散多码本 LM | 双自回归 (Dual-AR) |
| **空间操作** | 波形潜空间 | 离散码本空间 | 语义码本 + 残差码本 |
| **文本编码器** | - | Qwen3 | - |
| **音频编码器** | Wav-VAE | Qwen3-TTS-Tokenizer | RVQ Codec |

### 5.2 参数量对比

```
LongCat-AudioDiT-1B     ████████████████ 1B
LongCat-AudioDiT-3.5B   ████████████████████████████████████████ 3.5B
Qwen3-TTS-0.6B          █████ 0.6B
Qwen3-TTS-1.7B          ██████████████ 1.7B
Fish Audio S2 Pro       ███████████████████████████████████ 4B
```

### 5.3 延迟对比

| 模型 | 延迟 | 备注 |
|------|------|------|
| Qwen3-TTS | **97ms** | 流式输出首包 |
| Fish Audio S2 Pro | ~100ms | TTFA |
| LongCat-AudioDiT | - | 未明确标注 |

---

## 6. 性能评测对比

### 6.1 评测结论对比表

| 评测维度 | LongCat-AudioDiT | Qwen3-TTS | Fish Audio S2 Pro |
|----------|------------------|-----------|-------------------|
| **生成速度** | 🥇 6秒 | 🥈 33秒 | 🥉 >1分钟 |
| **音质** | 清晰 | 稳定 | 最好 |
| **音色迁移** | 效果好 | 适中 | 优秀 |
| **零样本克隆 SIM** | **0.818** (中文) | 0.770 | - |
| **中文 WER** | 1.09 | 1.22 | **0.54** |
| **英文 WER** | 1.50 | 1.23 | **0.99** |

### 6.2 主观评价（来源：电磁波Studio 评测）

> - **LongCat-AudioDiT**：生成速度仅需6秒，音质清晰，音色迁移效果好，**完胜**
> - **Qwen3-TTS**：生成速度33秒，轻量适中，音质稳定
> - **Fish Audio S2 Pro**：音质非常好，但生成速度很慢，资源占用率高

---

## 7. 适用场景分析

### 7.1 LongCat-AudioDiT

**优势场景**：
- ✅ 需要快速生成的实时语音对话
- ✅ 零样本语音克隆
- ✅ 对音色迁移效果要求高的应用
- ✅ OpenClaw 类语音助手项目

**推荐模型**：LongCat-AudioDiT-1B（速度与效果平衡）

### 7.2 Qwen3-TTS

**优势场景**：
- ✅ 需要流式输出的交互式应用
- ✅ 低延迟实时对话（97ms 首包）
- ✅ 多语言支持（10种语言）
- ✅ 语音设计和风格控制
- ✅ 快速语音克隆（3秒音频）

**推荐模型**：Qwen3-TTS-1.7B-CustomVoice（功能最全面）

### 7.3 Fish Audio S2 Pro

**优势场景**：
- ✅ 对音质有最高要求的场景
- ✅ 需要细粒度情感控制的创意内容
- ✅ 多说话人多轮对话
- ✅ 批量高质量语音内容生产

**注意**：生成速度较慢，不适合实时性要求高的场景

---

## 8. 结论与建议

### 8.1 综合评价

| 维度 | LongCat-AudioDiT | Qwen3-TTS | Fish Audio S2 Pro |
|------|------------------|-----------|-------------------|
| 🎯 **精准度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| ⚡ **速度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| 🎭 **音色** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 🌍 **多语言** | 中英 | 10种语言 | 80+语言 |
| 💰 **资源需求** | 中等 | 较低 | 较高 |

### 8.2 OpenClaw 语音对话选型建议

对于 OpenClaw 这类需要**实时语音对话**的项目：

1. **首选：LongCat-AudioDiT**
   - 生成速度快（6秒）
   - 音色迁移效果好
   - 零样本克隆能力强

2. **备选：Qwen3-TTS**
   - 流式输出支持（97ms）
   - 多语言支持
   - Apache 2.0 许可证（更宽松）

3. **音质优先：Fish Audio S2 Pro**
   - 音质最佳
   - 情感控制强大
   - 适合非实时的高质量内容生成

### 8.3 许可证说明

| 模型 | 许可证 | 商业可用 | 注意事项 |
|------|--------|----------|----------|
| LongCat-AudioDiT | MIT | ✅ | 需遵守 MIT 条款 |
| Qwen3-TTS | Apache 2.0 | ✅ | 可商用，需保留版权声明 |
| Fish Audio S2 Pro | Fish Audio Research | ⚠️ | 需查看具体许可证条款 |

---

## 📚 参考链接

### GitHub 仓库
- LongCat-AudioDiT: https://github.com/meituan-longcat/LongCat-AudioDiT
- Qwen3-TTS: https://github.com/QwenLM/Qwen3-TTS
- Fish Speech: https://github.com/fishaudio/fish-speech

### HuggingFace
- LongCat-AudioDiT: https://huggingface.co/meituan-longcat/LongCat-AudioDiT-1B
- Qwen3-TTS: https://huggingface.co/collections/Qwen/qwen3-tts
- Fish Audio S2 Pro: https://huggingface.co/fishaudio/s2-pro

### 论文
- LongCat-AudioDiT: [arXiv:2603.29339](https://arxiv.org/abs/2603.29339)
- Qwen3-TTS: [arXiv:2601.15621](https://arxiv.org/abs/2601.15621)
- Fish Audio S2: [arXiv:2603.08823](https://arxiv.org/abs/2603.08823)

---

*报告生成时间：2026-04-07*  
*数据来源：GitHub 项目页面及官方文档*
