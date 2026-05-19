# 02. 硬件系统 (Hardware System)

本模块详细介绍音频硬件的物理实现、电路架构以及在不同设备（手机、汽车）中的差异化设计。

## 📖 章节导航

1.  **[换能器：麦克风与扬声器 (Transducers)](./01-Transducers.md)**
    *   动圈式与电容式麦克风原理。
    *   MEMS 麦克风在移动设备中的应用。
    *   扬声器构造、安培力原理与核心性能指标。

2.  **[数字音频接口与总线 (Interface & Bus)](./02-Interface-Bus.md)**
    *   I2S 信号线、四种格式、主从模式与 Linux ASoC 配置。
    *   TDM 时分复用：Slot 映射、高通 DTS 配置。
    *   PDM 脉冲密度调制：ΣΔ 原理、Decimation 链、DMIC 调试。
    *   SoundWire：设备枚举、寄存器访问、Clock Stop 电源管理。

3.  **[移动端音频硬件架构 (Mobile Audio)](./03-Mobile-Hardware.md)**
    *   手机音频典型拓扑与 ADSP。
    *   SmartPA 及其保护算法 (IV-Sense)。
    *   从 3.5mm 到 USB-C/蓝牙的硬件演进。

4.  **[车载音频硬件架构 (Automotive Audio)](./04-Automotive-Hardware.md)**
    *   车载音频总线：A2B 技术深度解析。
    *   外部 DSP 功放与多通道系统设计。
    *   主动降噪 (ANC) 与路噪消除 (RNC) 的硬件要求。

5.  **[编解码器与智能功放 (Audio Codec & SmartPA)](./05-Codec-SmartPA.md)**
    *   Codec 功能模块与主流芯片（WCD938x、CS系列、ALC系列）。
    *   SmartPA IV-Sense 闭环保护原理与调参流程。
    *   WCD938x 寄存器操作实战与调试命令。
    *   高通平台 IV-Sense 数据通路。

6.  **[USB 音频 (USB Audio)](./06-USB-Audio.md)**
    *   USB Audio Class 协议（UAC 1.0/2.0/3.0）与描述符结构。
    *   等时传输模式：同步/自适应/异步（Feedback）。
    *   Linux snd-usb-audio 驱动架构与 URB 数据流。
    *   Android USB Audio HAL 与设备连接流程。
    *   USB Type-C 音频（模拟/数字模式）。

---
[返回主目录](../README.md)
