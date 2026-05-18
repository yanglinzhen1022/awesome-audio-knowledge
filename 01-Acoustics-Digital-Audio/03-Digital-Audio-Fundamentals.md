# 数字音频基础 (Digital Audio Fundamentals)

数字音频是将连续的模拟声波信号通过采样、量化和编码，转化为计算机可处理的离散二进制数据的过程。本章涵盖音频数字化过程中的数学边界与工程实践。

---

## 1. 采样 (Sampling) 与 奈奎斯特定理

采样是在时间轴上对信号进行离散化。

### 1.1 奈奎斯特-香农采样定理 (Nyquist-Shannon Theorem)
*   **定理**：为了无失真地重建最高频率为 $f_{max}$ 的连续信号，采样率 $f_s$ 必须满足 $f_s > 2 f_{max}$。
*   **混叠 (Aliasing)**：如果 $f_s \le 2 f_{max}$，高于奈奎斯特频率（$f_s/2$）的信号会“折返”回低频段，形成虚假信号。
*   **工程实践**：CD 采样率选择 44.1kHz 是因为人耳上限为 20kHz，留出 4.1kHz 的**过渡带 (Guard Band)** 以便设计低成本的抗混叠低通滤波器。

---

## 2. 量化 (Quantization) 与 位深

量化是在幅度轴上对信号进行离散化。

### 2.1 SQNR (信号量化噪声比)
量化过程必然产生误差（量化噪声）。对于 $N$ 位线性 PCM，其理论最大信噪比推导公式为：
$$\text{SQNR} \approx 6.02N + 1.76 \text{ dB}$$
*   **16-bit**：约 98 dB。
*   **24-bit**：约 146 dB（已超过大多数模拟电路的动态范围）。

### 2.2 抖动 (Dithering)：音频中的“玄学”数学
*   **问题**：当信号幅度极小时（低位跳变），量化误差会产生与信号相关的“谐波失真”。
*   **方案**：在量化前引入微量的随机白噪声。
*   **效果**：虽然略微降低了 SNR，但将难听的“数码味”失真转化为了背景底噪，使听感更自然，能听到掩埋在量化层级以下的微弱细节。

---

## 3. PCM 数据结构与码率

### 3.1 原始 PCM 存储格式
PCM (Pulse Code Modulation) 是音频数据的“生肉”。
*   **对齐方式**：
    *   **Little-endian**：Intel/Android/Windows 常用，低位在前。
    *   **Big-endian**：部分网络协议使用。
*   **布局**：
    *   `Interleaved` (交织)：L R L R L R...
    *   `Non-interleaved` (非交织)：LLLL... RRRR... (DSP 算法处理通常更喜欢这种模式，方便 SIMD 优化)。

### 3.2 码率计算公式
$$\text{Bitrate (bps)} = \text{SampleRate} \times \text{BitDepth} \times \text{Channels}$$
*   *例*：双声道 48kHz, 24bit PCM -> $48000 \times 24 \times 2 = 2.304 \text{ Mbps}$。

---

## 4. 常见音频术语解析

*   **帧 (Frame)**：包含所有声道在同一个采样时刻的数据。例如，16bit 立体声，1 帧 = 4 bytes。
*   **周期 (Period/Fragment)**：HAL 层一次处理的数据量。它是导致“音频延迟”的物理根源。
*   **Jitter (时钟抖动)**：采样时间点的不稳定，会导致高频失真。

---

## 5. 关键参考 (References)

1.  *Digital Audio Signal Processing* - Udo Zölzer
2.  [The Science of Sample Rates - Xiph.org](https://xiph.org/video/vid2.shtml)
3.  [Dither and Noise Shaping - Wikipedia](https://en.wikipedia.org/wiki/Dither)

---
*Next Module: [02. 硬件系统 (Hardware System)](../02-Hardware-System/README.md)*
