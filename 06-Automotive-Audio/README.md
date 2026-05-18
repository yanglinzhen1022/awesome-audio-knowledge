# 06. 车载音频系统 (Automotive Audio)

本模块深入探讨智能座舱环境下的系统级音频设计。不同于手机的垂直架构，车载音频是一个高度分布式、强调安全优先的复杂系统。

## 📖 章节导航

1.  **[车载音频系统概览 (Overview)](./01-Overview.md)**
    *   **仲裁矩阵 (Arbitration Matrix)**：P0 到 P4 的优先级定义与处理策略。
    *   **VHAL 边界**：连接 Android 与底层 MCU/DSP 的桥梁。
    *   域控架构与集中式架构的演进。

2.  **[车载多音区、动态路由与 Bus 绑定](./02-Multi-zone-Routing.md)**
    *   **Audio Zone (音区)**：物理隔离、Occupant Zone 乘员映射。
    *   **Bus 路由机制**：为何弃用设备类型改用逻辑总线地址。
    *   `car_audio_configuration.xml` 核心配置源码级解析。
    *   动态切换路由的代码级实现路径。

3.  **[车载音频焦点策略与音量组管理](./03-Focus-Volume-Management.md)**
    *   **车载自定义焦点 (CarAudioFocus)**：冲突矩阵与处理模型。
    *   **音量组 (Volume Groups)**：Context 到 Group 的映射逻辑。
    *   音量调节的底层“穿透”过程。
    *   **声场中心调节**：Fade (前后) 与 Balance (左右) 的计算逻辑。

4.  **[主动降噪与路噪消除 (ANC & RNC)](./04-ANC-RNC.md)**
    *   反相干涉基本原理与 FxLMS 自适应算法。
    *   发动机阶次 ANC 系统架构与次级通路建模。
    *   路噪消除 (RNC) MIMO 多通道架构与计算复杂度。
    *   ESE 引擎声浪模拟与 ANC 协同。

5.  **[AudioControl HAL 与 AAOS 架构演进](./05-AudioControl-HAL.md)**
    *   AudioControl HAL AIDL 接口详解（Ducking / 焦点 / Fade&Balance）。
    *   外部音源焦点管理 (IFocusListener) 机制。
    *   AAOS 架构演进：分布式 → 域控集中 → 中央计算。
    *   Hypervisor 多 OS 音频隔离与 Ethernet AVB/TSN。

---
[返回主目录](../README.md)
