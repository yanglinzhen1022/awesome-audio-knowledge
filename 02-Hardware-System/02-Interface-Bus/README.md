# 数字音频接口与总线 (Digital Audio Interfaces & Bus)

在音频 SoC 设计中，芯片间的数据交换（Inter-IC Communication）主要依赖以下几种串行总线。

---

## 1. I2S (Inter-IC Sound) 详解

I2S 是音频领域的行业标准。除了飞利浦定义的标准协议，实际工程中还有多种对齐格式。

### 1.1 常见的四种格式
1.  **I2S Phillips Standard**：MSB 在 LRCK 翻转后的第 2 个 BCLK 上升沿开始。
2.  **Left Justified (左对齐)**：MSB 在 LRCK 翻转后的第 1 个 BCLK 上升沿立即开始。
3.  **Right Justified (右对齐/EIAJ)**：数据在 LRCK 翻转前对齐，常用于部分旧式 DAC。
4.  **DSP / PCM Mode**：LRCK 变成一个单周期的脉冲，后面连续跟多个声道的数据。

### 1.2 主从模式 (Master vs Slave)
*   **Master**：提供 BCLK 和 LRCK 的芯片。
*   **Slave**：接收时钟并根据其节拍发送/接收数据。
*   *经验*：系统设计中通常由 SoC 作为 Master，Codec 作为 Slave。

---

## 2. TDM (Time Division Multiplexing)

当需要在一个物理引脚上传输多于 2 个声道（如车机 8 声道、12 声道）时，TDM 是唯一选择。

### 2.1 槽位 (Slot) 与宽度
*   一个 TDM 帧包含多个 Slot。
*   **Slot Width**：通常为 32-bit，但实际 Data 可能是 16-bit 或 24-bit。
*   **Bitclock 计算**：$BCLK = \text{SampleRate} \times \text{Slots} \times \text{SlotWidth}$。
    *   *例*：48kHz, 8ch, 32bit -> $48000 \times 8 \times 32 = 12.288 \text{ MHz}$。

---

## 3. PDM (Pulse Density Modulation)

PDM 是数字 MEMS 麦克风的标配接口。

### 3.1 调制原理
PDM 不传量化值，而是以极高的采样率（如 3.072MHz）传输 1 位码流。
*   **密度 -> 振幅**：1 的密度越高，代表模拟波形幅度越大。

### 3.2 SoC 侧的“抽取”处理 (Decimation)
SoC 接收到 PDM 信号后不能直接混音，必须经过：
1.  **CIC 滤波器**：降低采样率（抽取）。
2.  **补偿滤波器**：修正频响。
3.  **高通滤波器**：滤除直流分量。
*最终转化为 16/24-bit 的 PCM 信号。*

---

## 4. 新一代总线：SoundWire

MIPI 联盟推出的 SoundWire 旨在替代 I2S 和 PDM。
*   **优势**：支持在同一总线上挂载多个麦克风和扬声器，支持控制命令与数据并发，支持动态电源管理。

---

## 5. 关键参考 (References)

1.  *I2S Bus Specification* - Philips
2.  [Understanding Digital Audio Interfaces - TI](https://www.ti.com/lit/an/slaa701/slaa701.pdf)
3.  [MIPI SoundWire Specification](https://www.mipi.org/specifications/soundwire)

---
*Next Topic: [移动端与车载音频硬件架构 (Mobile & Automotive Audio Hardware)](../03-Mobile-Hardware.md)*
