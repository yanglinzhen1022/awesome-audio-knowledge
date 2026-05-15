# 04. Android 音频架构 (Android Audio Stack)

本模块深度拆解 Android 系统中的音频全链路，从应用层 API 到 Native 服务，再到硬件抽象层。

## 📖 章节导航

1.  **[Android 音频系统概览 (Overview)](./01-Overview.md)**
    *   分层架构图：App -> Framework -> Native -> HAL -> Kernel。
    *   各层级职责与数据流/控制流路径。

2.  **[AudioTrack 深度解析](./02-AudioTrack-Deep-Dive.md)**
    *   静态模式 (Static) 与流模式 (Stream)。
    *   JNI 层与 Native 层 AudioTrack 的对应关系。
    *   共享内存 (Shared Memory) 传输机制。

3.  **[AudioRecord 录音流程解析](./03-AudioRecord.md)**
    *   音频源 (AudioSource) 的选择与预处理。
    *   录音数据流：从 HAL 到应用层的传递。

4.  **[AudioFlinger 混音引擎详解](./04-AudioFlinger/README.md)**
    *   播放线程模型：MixerThread, DirectThread, FastMixer。
    *   混音、重采样与音效链处理。

5.  **[AudioPolicy 策略管理详解](./05-AudioPolicy/README.md)**
    *   路由决策逻辑：Usage -> Strategy -> Device。
    *   配置文件 `audio_policy_configuration.xml` 详解。

6.  **[Audio HAL 接口规范](./06-AudioHAL.md)**
    *   HAL 演进：Legacy -> HIDL -> AIDL。
    *   核心接口：IDevice 与 IStream。
    *   与 AudioFlinger 的跨进程通信链路。

---
[返回主目录](../README.md)
