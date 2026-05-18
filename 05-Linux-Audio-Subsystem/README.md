# 05. Linux 音频子系统 (Linux Audio Subsystem)

本模块介绍 Linux 内核及桌面系统的音频架构，这是 Android 底层驱动及嵌入式音频系统的技术支撑。

## 📖 章节导航

1.  **[ALSA 核心架构 (ALSA Architecture)](./01-ALSA-Architecture.md)**
    *   ALSA 系统分层：alsa-lib, 内核中间层与驱动。
    *   设备节点解析：pcm, control, timer。
    *   alsa-lib 与 tinyalsa 的区别与应用场景。

2.  **[ASoC 驱动模型 (ASoC Driver Model)](./02-ASoC-Driver-Model.md)**
    *   三大核心组件：Codec, Platform, Machine 驱动。
    *   DAPM 深度解析：Widget 完整分类、有向图寻路、Bias Level 状态机、电源序列、代码示例。
    *   FE/BE 与 DPCM：Dynamic PCM 动态路由机制。
    *   DMA 传输机制：环形缓冲区、Period/Buffer/延迟关系、驱动实现框架。
    *   Device Tree 音频节点：simple-audio-card / 高通车载多链路 DTS 示例。

3.  **[现代 Linux 音频服务 (PulseAudio & PipeWire)](./03-PulseAudio-PipeWire.md)**
    *   PulseAudio：桌面音频混音与网络透明性。
    *   PipeWire：下一代统一音视频处理标准。
    *   低延迟图模型架构与 Jack 兼容性。

4.  **[ALSA 声卡注册与实例化](./04-ALSA-Card-Registration.md)**
    *   Machine Driver 的匹配与绑定流程。
    *   `snd_soc_bind_card` 核心函数解析。
    *   DAI Link 的 FE 与 BE 概念。

---
[返回主目录](../README.md)
