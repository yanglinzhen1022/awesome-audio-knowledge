# 04. Android 音频架构 (Android Audio Stack)

本模块深度拆解 Android 系统中的音频全链路。

## 📖 章节导航

1.  **[Android 音频系统概览 (Overview)](./01-Overview.md)**
    *   全景架构、进程隔离模型与 AAOS 特殊集成。
2.  **[AudioService 系统管理中心](./02-AudioService.md)**
    *   Java 层管理总管，负责音量、焦点与设备连接状态。
3.  **[AudioTrack 播放流程解析](./03-AudioTrack.md)**
    *   Native 初始化调用栈与共享内存同步机制。
4.  **[AudioRecord 录音流程解析](./04-AudioRecord.md)**
    *   音频源选择与录音数据流向。
5.  **[AudioFlinger 混音引擎深度解析](./05-AudioFlinger.md)**
    *   线程模型全景：MixerThread / DirectOutput / OffloadThread / MMAP。
    *   Track 生命周期与状态机（异步状态转换机制）。
    *   **FastTrack vs NormalTrack 深度对比**：准入条件、FastMixer 独立线程架构。
    *   重采样器 (Resampler) 多相滤波器原理与质量等级。
    *   Buffer 链路：Ashmem 共享内存、环形缓冲区、Underrun 处理。
    *   Dump 实战分析：关键字段解读与常见问题定位清单。
6.  **[AudioPolicy 策略管理深度解析](./06-AudioPolicy.md)**
    *   `audio_policy_configuration.xml` 完整结构解析（mixPort/devicePort/route）。
    *   **路由决策全链路**：Usage → Strategy → Device → Output 选择。
    *   AudioPolicyEngine 可替换架构（Default vs Configurable/PFW）。
    *   音量控制体系：曲线配置、Index→dB 转换、下发路径源码跟踪。
    *   设备连接/断开处理全链路（耳机、蓝牙 A2DP 跨模块路由）。
    *   AudioPatch 硬件直连机制。
    *   AAOS 车载特殊策略（Bus 路由、CarAudioFocus）。
7.  **[Audio HAL 接口规范](./07-AudioHAL.md)**
    *   从 HIDL 到 AIDL 的演进与 ALSA 驱动对接。
8.  **[AudioEffect 音效框架深度解析](./08-AudioEffect.md)**
    *   音效链加载与 Buffer 传递同步。
9.  **[AudioFocus 音频焦点机制](./09-AudioFocus.md)**
    *   基于协作的多应用焦点竞争管理。
10. **[Oboe 与 AAudio：低延迟音频 API](./10-Oboe-AAudio.md)**
    *   AAudio MMAP 独占路径与数据回调模型。
    *   Oboe 跨版本兼容方案与自动重连。
    *   延迟优化 Checklist 与测量方法。
11. **[VoIP 与通话音频链路 (VoIP & Voice Call Chain)](./11-VoIP-Call-Chain.md)**
    *   CS Voice / VoLTE / VoNR / VoIP 通话类型全景。
    *   高通平台 VoLTE 音频路径（ADSP ↔ Modem 直连）。
    *   VoIP App 架构与 AudioAttributes 配置。
    *   通话 3A 处理（AEC/NS）与 Jitter Buffer。
    *   通话质量评估指标与调试命令。

---
[返回主目录](../README.md)
