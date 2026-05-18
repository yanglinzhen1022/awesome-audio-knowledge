# 语音交互算法 (Voice Interaction: KWS, ASR, NLU, TTS)

语音交互是音频、语言学与深度学习的交叉学科。本章解析将模拟语音转化为机器指令的核心算法流程。

---

## 1. 特征提取：从波形到频谱 (MFCC)

机器无法直接处理原始 PCM。MFCC (梅尔频率倒谱系数) 是目前最常用的语音特征。

### 1.1 提取步骤
1.  **预加重 (Pre-emphasis)**：高通滤波，平衡频谱，提升高频能量。
2.  **分帧与加窗 (Framing & Windowing)**：通常 20-30ms 一帧，加汉明窗防止频谱泄露。
3.  **FFT**：计算每帧的功率谱。
4.  **Mel 滤波器组**：模拟人耳对频率的对数感知。
5.  **离散余弦变换 (DCT)**：去除各频带间的相关性。

---

## 2. 语音前端的关键差异：VAD vs KWS

这是最容易混淆的两个概念：

| 特性 | VAD (语音活动检测) | KWS (唤醒词识别) |
| :--- | :--- | :--- |
| **主要目标** | 区分“人声”还是“环境噪声” | 识别特定的词（如“嘿 Siri”） |
| **计算复杂度** | 极低，通常是基于能量或轻量级 CNN | 中等，需要比对特定声学特征 |
| **运行位置** | 常驻硬件 (Always-on) 或 DSP 边缘 | 常驻 DSP |

---

## 3. 语音识别 (ASR) 的主流架构

### 3.1 传统架构：HMM-DNN
*   **声学模型**：DNN/CNN。
*   **语言模型**：N-gram。
*   **解码器**：加权有限状态转换器 (WFST)。

### 3.2 现代架构：End-to-End (E2E)
*   **CTC (Connectionist Temporal Classification)**：无需对齐。
*   **LAS (Listen, Attend and Spell)**：基于注意力机制。
*   **Transformer / Conformer**：目前工业界的 SOTA (State-of-the-art) 方案。

---

## 4. 自然语言理解 (NLU) 的核心：Slot Filling

NLU 的目标是将文本结构化。
*   **意图识别 (Intent)**：查询、控制、闲聊。
*   **槽位填充 (Slot Filling)**：
    *   *输入*：“帮我把 **空调** 调到 **26度**”
    *   *输出*：Device="Air-conditioner", Value="26", Unit="Celsius"。

---

## 5. 关键参考 (References)

1.  *Speech and Language Processing* - Jurafsky & Martin
2.  [Mel-frequency cepstrum - Wikipedia](https://en.wikipedia.org/wiki/Mel-frequency_cepstrum)
3.  [OpenAI Whisper Architecture Blog](https://openai.com/research/whisper)
