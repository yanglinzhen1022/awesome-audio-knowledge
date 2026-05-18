# 声波物理特性 (Sound Wave Physics)

声音是音频工程的逻辑起点。理解声音如何产生、传播以及其物理参数，是掌握后续数字信号处理 (DSP) 和声学设计的前提。

---

## 1. 什么是声音 (Definition of Sound)

**声音 (Sound)** 是由物体振动产生的**机械波 (Mechanical Wave)**。
*   **传播介质**：声音必须依靠介质（固体、液体、气体）传播，真空中无法传声。
*   **波的类型**：在空气中，声音以**纵波 (Longitudinal Wave)** 的形式传播，即介质质点的振动方向与波的传播方向平行，形成交替的**疏部 (Rarefaction)** 和 **密部 (Compression)**。

---

## 2. 核心物理参数 (Core Physical Parameters)

### 2.1 频率 (Frequency, $f$) 与 周期 (Period, $T$)
*   **定义**：物体每秒振动的次数称为频率，单位为 **赫兹 (Hz)**。振动一次所需的时间称为周期。
*   **关系**：$f = 1 / T$
*   **听觉范围**：人耳能听到的频率范围约为 **20Hz - 20,000Hz (20kHz)**。
    *   < 20Hz: 次声波 (Infrasound)
    *   > 20kHz: 超声波 (Ultrasound)

### 2.2 波长 (Wavelength, $\lambda$)
*   **定义**：声波在传播方向上，相邻两个相同相位点（如两个波峰）之间的距离。
*   **计算公式**：
    $$\lambda = \frac{v}{f}$$
    *(其中 $v$ 为声速)*
*   **专业应用**：在声学设计中，如果房间的尺寸与声波波长成整数倍关系，会产生**驻波 (Standing Wave)**，导致某些频率音量异常放大，产生严重的染色。

### 2.3 声速 (Speed of Sound, $v$)
*   声速取决于介质的弹性系数和密度。
*   **空气中的声速**：在 20°C (68°F) 的干燥空气中，声速约为 **343 m/s**。
*   **温度影响**：温度越高，声速越快。近似公式：$v \approx 331.5 + 0.6T$ (其中 $T$ 为摄氏度)。

### 2.4 相位 (Phase, $\phi$)
*   **定义**：描述声波在特定时间点处于振动周期中哪个位置的物理量，通常用角度 (0° - 360°) 表示。
*   **相位差 (Phase Difference)**：两个频率相同的声波，如果起始时间不同，就会产生相位差。这是 **回声消除 (AEC)** 和 **主动降噪 (ANC)** 的核心物理基础。

---

## 3. 声学现象解析 (Acoustic Phenomena)

### 3.1 干涉 (Interference)
当两列或多列频率相同的声波在空间相遇时，会产生叠加效果：
*   **相长干涉 (Constructive Interference)**：两波相位相同，叠加后振幅增大。
*   **相消干涉 (Destructive Interference)**：两波相位相差 180°，叠加后振幅减小甚至抵消（主动降噪原理）。

```mermaid
graph TD
    subgraph "相消干涉 (Destructive Interference)"
        A1[原始声波 Phase: 0°]
        A2[反相声波 Phase: 180°]
        A3[叠加结果: 静音/衰减]
        A1 + A2 --> A3
    end
```

### 3.2 衍射与反射 (Diffraction & Reflection)
*   **反射**：声波遇到大于其波长的障碍物时会发生反射。
*   **RT60 (混响时间)**：衡量空间特性的重要指标。定义为声场停止发声后，声压级衰减 60dB 所需的时间。
    *   **赛宾公式 (Sabine Formula)**：$RT60 = 0.161 \cdot \frac{V}{A}$
    *   $V$：房间体积；$A$：总吸声量。

---

## 4. 分贝系统 (The Decibel System)

音频领域有多种分贝基准，混淆它们是专业工作中的大忌。

### 4.1 dB SPL (声压级)
用于衡量空气中的声音强度。参考基准 $P_0 = 20 \mu Pa$（人耳听阈）。
$$L_p = 20 \log_{10} \left( \frac{P}{P_0} \right)$$

### 4.2 dBFS (Full Scale)
**数字音频标准**。0 dBFS 代表数字系统能表达的最大电平（所有位全是 1）。
*   在数字系统中，所有的电平通常都是负值（如 -6 dBFS）。
*   **警告**：超过 0 dBFS 会导致数字斩波（Clipping），产生极难听的失真。

### 4.3 dBu & dBV (电压电平)
*   **dBu**：参考基准 0.775 Vrms（常用于专业音频设备，+4 dBu 为标准电平）。
*   **dBV**：参考基准 1.0 Vrms（常用于民用消费电子，-10 dBV 为标准电平）。

---

## 5. 关键参考 (References)

1.  *Principles of Vibration and Sound* - Thomas D. Rossing
2.  [Sound - Wikipedia](https://en.wikipedia.org/wiki/Sound)
3.  [The Speed of Sound - Engineering Toolbox](https://www.engineeringtoolbox.com/speed-sound-d_82.html)

---
*Next Topic: [心理声学 (Psychoacoustics)](./02-Psychoacoustics.md)*
