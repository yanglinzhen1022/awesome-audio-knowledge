# 04. Android 音频架构 (Android Audio Stack)

本模块深度拆解 Android 系统中的音频全链路。

## 📖 章节导航

1.  **[Android 音频系统概览 (Overview)](./01-Overview.md)**
    *   全景架构图：App → Framework → AudioFlinger → HAL → Kernel 五层模型。
    *   各层级核心组件与职责详解。
    *   进程隔离模型：App / SystemServer / audioserver / HAL 进程边界与崩溃影响。
    *   音频流类型与 AudioAttributes 对比。
    *   数据路径对比：Normal Path / Fast Path / MMAP 延迟与特性。
    *   关键配置文件速查（audio_policy_configuration.xml 等）。
    *   版本演进：Android 5.0 ~ 15 音频关键特性时间线。
    *   AAOS 车载场景扩展概览与调试入口速查。
2.  **[AudioService 系统管理中心](./02-AudioService.md)**
    *   AudioService 在系统中的位置与启动流程。
    *   音量管理体系：StreamVolumeState、音量曲线、安全音量 (CSD)。
    *   设备管理：AudioDeviceInventory、AudioDeviceBroker、连接/断开事件处理。
    *   Ringer Mode 与 Do Not Disturb 联动机制。
    *   媒体按键分发：MediaSession → AudioService → AudioPolicy 链路。
3.  **[AudioTrack 播放流程解析](./03-AudioTrack.md)**
    *   Java API 配置要点：AudioAttributes / AudioFormat / BufferSize / PerformanceMode。
    *   工作模式对比：MODE_STREAM vs MODE_STATIC 内存策略与适用场景。
    *   JNI 桥接：Java AudioTrack → Native AudioTrack (libaudioclient) 完整调用链。
    *   Native 初始化调用栈：`createTrack_l()` → Binder → AudioFlinger `createTrack()`。
    *   共享内存机制：Ashmem 分配、Control Block 结构、环形缓冲区读写同步。
    *   数据写入路径：`write()` → obtainBuffer / releaseBuffer → MixerThread 消费。
    *   Underrun 处理策略与 getMinBufferSize 计算原理。
4.  **[AudioRecord 录音流程解析](./04-AudioRecord.md)**
    *   AudioSource 选型指南：MIC / VOICE_COMMUNICATION / UNPROCESSED / VOICE_RECOGNITION 等场景差异。
    *   AudioSource 与预处理算法绑定：VOICE_COMMUNICATION 自动加载 AEC/NS/AGC 的底层机制。
    *   JNI 桥接与 Native AudioRecord 初始化调用栈。
    *   RecordThread 数据流：HAL read() → ResamplerBufferProvider → RecordTrack → SharedMem → App。
    *   多客户端并发录音：RecordThread 多 RecordTrack 分发、权限控制与优先级抢占。
    *   权限与隐私：RECORD_AUDIO 权限、前台服务要求 (Android 9+)、录音指示器 (Android 12+)。
    *   Overrun 诊断与 Buffer 容量调优。
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
    *   三代 HAL 接口演进：Legacy C → HIDL → AIDL，传输机制与核心文件对比。
    *   AIDL HAL 核心接口：IModule / IStreamOut / IStreamIn / IConfig 层次关系。
    *   FMQ (Fast Message Queue) 数据传输机制与零拷贝设计。
    *   HAL 实现与 ALSA/TinyALSA 驱动对接：open/write/read 调用流程。
    *   Audio HAL 调试：VTS 测试、HAL dump、常见兼容性问题。
8.  **[AudioEffect 音效框架深度解析](./08-AudioEffect.md)**
    *   架构分层：App API → EffectsFactory → EffectChain → EffectModule → HAL/DSP。
    *   音效链挂载点：Session / PerStream / Global，Insert vs Auxiliary 效果类型。
    *   Buffer 传递与同步：EffectChain 在 MixerThread 中的处理时序。
    *   AIDL Effect HAL 迁移与 HW Offload 音效。
    *   自定义音效开发流程与 audio_effects.xml 配置。
9.  **[AudioFocus 音频焦点机制](./09-AudioFocus.md)**
    *   焦点类型：GAIN / GAIN_TRANSIENT / GAIN_TRANSIENT_MAY_DUCK / GAIN_TRANSIENT_EXCLUSIVE。
    *   MediaFocusControl 仲裁逻辑与 FocusStack 栈管理。
    *   焦点丢失响应：LOSS / LOSS_TRANSIENT / LOSS_TRANSIENT_CAN_DUCK 处理策略。
    *   FadeManager (Android 14+) 自动淡入淡出与延迟焦点 (Delayed Focus)。
    *   AAOS CarAudioFocus 车载定制：交互矩阵与多区域焦点独立管理。
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
12. **[Android 音频版本新特性 (Version Changelog)](./12-Android-New-Features.md)**
    *   Android 14：Loudness CTA-2075 响度控制、Spatializer AIDL 稳定化、Ultra HDR Audio 路径。
    *   Android 15：Audio Sharing (Auracast)、Per-App 响度、MMAP Shared 增强、Virtual Audio Device。
    *   历史关键版本里程碑回顾（Android 5.0 ~ 15）。

---
[返回主目录](../README.md)
