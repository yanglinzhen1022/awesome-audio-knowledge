# 09. 蓝牙音频 (Bluetooth Audio)

本模块系统梳理蓝牙音频技术栈，从经典蓝牙 A2DP/HFP 到最新的 LE Audio，覆盖协议原理、编解码、Android 集成架构与调试方法。

## 📖 章节导航

1.  **[蓝牙音频协议栈与编解码](./01-BT-Audio-Protocols.md)**
    *   经典蓝牙音频协议：A2DP (SBC/AAC/aptX/LDAC) 与 HFP (mSBC/LC3-SWB)。
    *   LE Audio 协议栈：BAP、CAP、TMAP、Isochronous Channels。
    *   LC3 编解码深度解析：帧结构、码率、延迟与音质对比。
    *   Auracast 广播音频架构。

2.  **[Android 蓝牙音频架构与调试](./02-Android-BT-Audio.md)**
    *   Android BT Audio HAL 架构演进（HIDL → AIDL）。
    *   Bluetooth Audio AIDL HAL 接口详解。
    *   A2DP / LE Audio 数据通路与 Session 管理。
    *   蓝牙音频调试：btsnoop、dumpsys、HCI log 分析。

---
[返回主目录](../README.md)
