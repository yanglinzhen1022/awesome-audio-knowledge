# Android 音频版本新特性 (Audio Version Changelog)

本章按 Android 版本汇总音频子系统的重要变更，聚焦架构演进和新增 API，方便开发者快速了解升级影响。

---

## 📑 目录

- [Android 14 (API 34)](#android-14-api-34)
- [Android 15 (API 35)](#android-15-api-35)
- [历史关键版本回顾](#历史关键版本回顾)

---

## Android 14 (API 34)

### 1. Loudness 响度控制

Android 14 引入 CTA-2075 响度标准支持，允许系统对不同内容自动应用响度补偿。

```java
// 获取 Loudness 控制器
AudioManager am = context.getSystemService(AudioManager.class);
LoudnessCodecController controller = am.createLoudnessCodecController(
    sessionId,
    Executors.newSingleThreadExecutor(),
    (mediaCodec, loudnessParams) -> {
        // 回调中应用响度参数到 MediaCodec
        // loudnessParams 包含: targetLoudness, kneePoint, limiterRelease
        mediaCodec.setParameters(loudnessParams);
    }
);
controller.start();
```

**架构影响：**
```
App (MediaCodec) ← LoudnessCodecController ← AudioService
                                              ↕
                              CTA-2075 Metadata (从 Extractor 提取)
                                              ↕
                              audio_policy_configuration.xml
                              (loudness_control_enabled = true)
```

**关键点：**
- 需要内容本身携带响度元数据（如 AC-4/Dolby 的 `dialnorm`）
- 系统级自动补偿，无需 App 手动处理
- 厂商可通过 HAL 侧 `ILoudnessEnhancerEffect` 实现硬件加速

### 2. 空间音频 (Spatializer) AIDL 稳定化

```
Android 12: Spatializer 引入 (实验性, HIDL)
Android 13: Head Tracking API 开放
Android 14: ISpatializer AIDL 接口稳定化 + 第三方耳机支持

AIDL 接口:
  ISpatializer.aidl
    ├── setHeadTrackingMode(HeadTrackingMode mode)
    ├── getSupportedHeadTrackingModes() → HeadTrackingMode[]
    ├── setDesiredHeadTrackingMode(mode)
    ├── recenterHeadTracker()
    └── setSensorPose(float[] quaternion)

  IHeadTrackingCallback.aidl
    └── onHeadTrackingChanged(HeadTrackingMode mode)
```

**第三方耳机适配：**
- Android 14 开放 `SpatializerHeadTrackingCallback` 给蓝牙耳机厂商
- 耳机通过 LE Audio 的 Head Tracking Profile 上报 IMU 数据
- 系统 Spatializer 引擎 (通常在 ADSP) 接收数据做 HRTF 渲染

### 3. Ultra HDR Audio 路径优化

```
Android 14 对 24-bit / 32-bit 高精度音频的改进:

  之前 (Android 13-):
    App (32-bit float) → AudioFlinger 混音 (内部 float)
      → 截断为 16-bit/24-bit → HAL

  之后 (Android 14+):
    App (32-bit float) → AudioFlinger 混音 (内部 float)
      → 保持 32-bit float → HAL (如果 HAL 声明支持)
      → Codec 内部 DAC 处理全精度

  条件:
    - audio_policy_configuration.xml mixPort format 包含 AUDIO_FORMAT_PCM_FLOAT
    - HAL 实现 IStreamOut::prepareForWriting() 支持 float buffer
    - 仅 DirectOutput / MMAP 路径可全链路保持
```

### 4. 预测性返回手势音频反馈

- 新增 `HapticGenerator` 效果增强，支持导航返回手势的触觉-音频联合反馈
- `AudioAttributes.USAGE_ASSISTANCE_NAVIGATION_GUIDANCE` 新增低延迟路径

---

## Android 15 (API 35)

### 1. Audio Sharing (LE Audio Auracast 系统集成)

```
Android 15 将 Auracast 广播音频提升为系统级功能:

  ┌─────────────────────────────────────────────────────┐
  │ Settings UI                                         │
  │   "Share Audio" → 选择音源 → 生成广播              │
  └──────────────────────┬──────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────┐
  │ BluetoothLeBroadcast API (App 层)                   │
  │   startBroadcast(BroadcastSettings)                 │
  │   stopBroadcast(broadcastId)                        │
  │   updateBroadcast(broadcastId, settings)            │
  └──────────────────────┬──────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────┐
  │ LE Audio Stack                                      │
  │   BAP Broadcast Source → BIS (Broadcast ISO Stream) │
  │   Codec: LC3 (mandatory) / LC3plus (optional)       │
  │   Encryption: Broadcast Code (16-byte)              │
  └─────────────────────────────────────────────────────┘

使用场景:
  - 机场/健身房公共广播
  - 课堂分享音频给多人
  - 家庭多房间同步播放
```

**开发者 API：**
```java
// Android 15: 系统级广播音频
BluetoothLeBroadcast broadcast = 
    BluetoothAdapter.getDefaultAdapter()
        .getProfileProxy(context, listener, BluetoothProfile.LE_AUDIO_BROADCAST);

BluetoothLeBroadcastSettings settings = new BluetoothLeBroadcastSettings.Builder()
    .setBroadcastName("My Music Share")
    .setBroadcastCode(new byte[]{...})  // 加密码 (可选)
    .addSubgroupSettings(
        new BluetoothLeBroadcastSubgroupSettings.Builder()
            .setContentMetadata(metadata)
            .setPreferredQuality(BluetoothLeBroadcastSubgroupSettings.QUALITY_HIGH)
            .build())
    .build();

broadcast.startBroadcast(settings);
```

### 2. Per-App 响度独立控制

```
Android 15 扩展了响度控制:

  Android 14: 全局响度补偿 (基于内容元数据)
  Android 15: 每个 App 独立响度曲线

  AudioManager API:
    setAppVolumeLoudness(int uid, LoudnessParams params)
    getAppVolumeLoudness(int uid) → LoudnessParams

  内部实现:
    AudioService → AudioPolicyManager
      → 为每个 uid 维护独立的 volume curve offset
      → 混音前在 AudioFlinger 的 Track 级别应用 gain 补偿
```

### 3. MMAP 共享模式增强

```
Android 15 对 AAudio MMAP 共享模式 (SHARED) 的改进:

  之前:
    MMAP EXCLUSIVE: 单 App 独占 → 最低延迟 (1-2ms)
    MMAP SHARED: 多 App 共享 → 走 MixerThread → 延迟回退到 ~10ms

  之后 (Android 15):
    MMAP SHARED 优化:
    - MmapPlaybackThread 内部支持多 client 直接混音
    - 减少一次 buffer copy
    - 延迟从 ~10ms 降至 ~3-5ms (取决于 HAL 实现)

  HAL 变更:
    IStreamOut.aidl 新增:
      createMmapBuffer(MmapBufferInfo info, int clientCount)
      // clientCount > 1 表示 shared 模式请求
```

### 4. 虚拟化音频设备 (Virtual Audio Device)

```
Android 15 完善了虚拟音频设备框架:

  场景: 虚拟机 / 远程桌面 / 多用户隔离
  
  VirtualAudioDevice API:
    VirtualDeviceManager vdm = context.getSystemService(VirtualDeviceManager.class);
    VirtualDevice device = vdm.createVirtualDevice(associationId, params);
    
    VirtualAudioDevice audioDevice = device.createVirtualAudioDevice(
        new VirtualAudioDeviceConfig.Builder()
            .setInputConfig(inputConfig)   // 虚拟 MIC
            .setOutputConfig(outputConfig) // 虚拟 Speaker
            .build()
    );
    
    // 从虚拟设备读取音频 (该 VirtualDevice 上 App 的播放音频)
    audioDevice.getAudioOutputStream().read(buffer);
    
    // 向虚拟设备注入音频 (作为该设备上 App 的录音输入)
    audioDevice.getAudioInputStream().write(buffer);
```

### 5. 音频权限细化

| 权限 | Android 14 | Android 15 |
|:---|:---|:---|
| 前台录音 | `RECORD_AUDIO` | `RECORD_AUDIO` |
| 后台录音 | `RECORD_AUDIO` + 前台服务 | 新增 `RECORD_AUDIO_BACKGROUND` (更严格) |
| 热词检测 | `CAPTURE_AUDIO_HOTWORD` | 保持 (系统 App only) |
| 音频路由 | `MODIFY_AUDIO_ROUTING` | 保持 (系统签名) |

---

## 历史关键版本回顾

| 版本 | 里程碑 |
|:---|:---|
| **Android 5.0** | AudioPolicy 重构为 Engine 架构; OpenSL ES 低延迟改进 |
| **Android 6.0** | AudioPatch 机制引入; MIDI API |
| **Android 7.0** | A2DP Offload; HW Volume 精细控制 |
| **Android 8.0** | AAudio API 引入; Audio HAL HIDL 化; Volume Group |
| **Android 9.0** | Oboe 库发布; MMAP 路径成熟; 动态 AudioPolicy 配置 |
| **Android 10.0** | 并发录音策略; AudioAttributes 完善; Audio Capture Policy |
| **Android 11.0** | AIDL HAL 开始迁移; Capture Playback (系统级回放捕获) |
| **Android 12.0** | Spatializer 引入; LE Audio 初始支持; Haptic coupled playback |
| **Android 13.0** | LE Audio 完善 (BAP/CAP); Head Tracking; SpatializerEffect |
| **Android 14.0** | Loudness CTA-2075; Spatializer AIDL 稳定; Ultra HDR Audio |
| **Android 15.0** | Audio Sharing/Auracast; MMAP Shared 增强; Virtual Audio Device |

---

## 参考

- Android Audio 版本说明: https://developer.android.com/about/versions
- AOSP Audio HAL AIDL: `hardware/interfaces/audio/aidl/`
- Spatializer AIDL: `frameworks/av/services/audiopolicy/service/Spatializer.h`
- LE Audio Broadcast: `packages/modules/Bluetooth/system/bta/le_audio/broadcaster/`
