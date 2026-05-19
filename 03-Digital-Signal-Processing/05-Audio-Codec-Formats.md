# 音频编解码格式 (Audio Codec Formats)

音频编解码 (Audio Codec) 是对原始 PCM 数据进行压缩编码以减少存储和传输带宽的技术。不同格式在压缩率、音质、延迟和计算复杂度之间做出不同权衡。

---

## 1. 基本概念

### 1.1 无损 vs 有损

| 类型 | 原理 | 典型格式 | 压缩率 |
|:---|:---|:---|:---|
| **无损 (Lossless)** | 信息熵编码，可完美还原 | FLAC, ALAC, APE | 50%-70% |
| **有损 (Lossy)** | 利用心理声学模型丢弃不可闻信息 | AAC, MP3, Opus | 5%-20% |

### 1.2 关键指标

*   **比特率 (Bitrate)**：kbps，直接影响音质和带宽占用
*   **采样率 (Sample Rate)**：44.1kHz (CD) / 48kHz (影视) / 96kHz+ (Hi-Res)
*   **编解码延迟 (Codec Latency)**：帧长 + Look-ahead，实时通话要求 < 40ms
*   **算法复杂度 (Complexity)**：影响 CPU/DSP 功耗

---

## 2. 通用音频编解码格式

### 2.1 AAC (Advanced Audio Coding)

业界最广泛的有损格式，MPEG 标准家族成员。

| Profile | 比特率 | 特性 | 应用场景 |
|:---|:---|:---|:---|
| **AAC-LC** | 128-256 kbps | 低复杂度、高兼容性 | 音乐流媒体、视频 |
| **HE-AAC v1** | 48-64 kbps | 加入 SBR (频谱带复制) | 数字广播 (DAB+) |
| **HE-AAC v2** | 24-48 kbps | 加入 PS (参数立体声) | 低带宽流媒体 |
| **AAC-ELD** | 24-64 kbps | 增强低延迟 (< 20ms) | FaceTime 通话 |
| **xHE-AAC** | 12-256 kbps | 统一低码率到高码率 | 自适应流媒体 |

### 2.2 Opus

IETF 开放标准，目前公认最先进的通用音频编解码器。

```
Opus 内部架构:

  输入 PCM
    │
    ▼
  ┌────────────────────────────────────────────┐
  │ Opus Encoder                               │
  │                                            │
  │  ┌─────────┐  ┌──────────┐  ┌──────────┐  │
  │  │  SILK   │  │  Hybrid  │  │   CELT   │  │
  │  │(语音核心)│  │(混合模式)│  │(音乐核心)│  │
  │  │ 6-20kbps│  │ 20-64kbps│  │ 64-510kbps│ │
  │  │ NB/WB   │  │ SWB      │  │ FB       │  │
  │  └─────────┘  └──────────┘  └──────────┘  │
  │       ↑              ↑            ↑        │
  │       └──── 自动切换 (基于内容检测) ────┘   │
  └────────────────────────────────────────────┘
    │
    ▼
  Opus Bitstream (OGG/WebM/RTP 封装)

  帧长选择:
    2.5ms / 5ms / 10ms / 20ms / 40ms / 60ms
    通话场景: 20ms (低延迟)
    音乐场景: 60ms (高压缩效率)
```

*   **延迟**：最低 2.5ms（业界最低之一）
*   **采样率**：8kHz - 48kHz 全覆盖
*   **应用**：WebRTC 默认编解码器、Discord、微信通话

### 2.3 MP3 (MPEG-1 Audio Layer III)

*   历史最悠久的有损格式，专利已过期 (2017)
*   128kbps 已成为"可接受音质"的基线
*   **局限**：不支持多声道、编码效率低于 AAC/Opus

### 2.4 FLAC (Free Lossless Audio Codec)

*   最流行的开源无损格式
*   **压缩率**：典型约 50%-60%
*   **解码复杂度**：极低，适合嵌入式设备
*   Android 原生支持

### 2.5 ALAC (Apple Lossless Audio Codec)

*   Apple 生态的无损格式
*   性能与 FLAC 接近，Apple 设备原生支持
*   2011 年已开源

---

## 3. 蓝牙音频编解码格式

蓝牙音频受限于无线带宽，编解码格式直接决定了音质上限。

### 3.1 格式对比

| 格式 | 标准 | 最高码率 | 采样率 | 延迟 | 授权 |
|:---|:---|:---|:---|:---|:---|
| **SBC** | A2DP 必选 | 328 kbps | 48kHz/16bit | ~150ms | 免费 |
| **AAC** | A2DP 可选 | 256 kbps | 44.1kHz | ~100ms | 许可 |
| **aptX** | Qualcomm | 384 kbps | 48kHz/16bit | ~70ms | 许可 |
| **aptX HD** | Qualcomm | 576 kbps | 48kHz/24bit | ~100ms | 许可 |
| **aptX Adaptive** | Qualcomm | 420 kbps | 96kHz/24bit | ~50ms | 许可 |
| **LDAC** | Sony | 990 kbps | 96kHz/24bit | ~100ms | 开放 |
| **LC3** | Bluetooth LE Audio | 160 kbps | 48kHz | ~20ms | 免版税 |
| **LC3plus** | Fraunhofer | 256+ kbps | 96kHz | ~5ms | 许可 |

### 3.2 LC3：LE Audio 的核心编解码器

LC3 (Low Complexity Communication Codec) 是蓝牙 5.2 LE Audio 的强制编解码器，代表蓝牙音频的未来方向：

```mermaid
graph TD
    subgraph Classic_BT ["经典蓝牙 A2DP"]
        SBC["SBC (必选)"]
        APTX["aptX / LDAC (可选)"]
    end
    
    subgraph LE_Audio ["LE Audio (蓝牙 5.2+)"]
        LC3_CODEC["LC3 (必选)"]
    end
    
    Classic_BT -->|"演进"| LE_Audio
```

**LC3 优势**：
*   同等码率下音质显著优于 SBC
*   极低延迟 (帧长 7.5ms / 10ms)
*   支持多流 (Multi-Stream)：真正的独立左右耳传输
*   支持广播音频 (Auracast)

### 3.3 Android 蓝牙音频编解码路径

```mermaid
graph LR
    APP[应用层] --> AT[AudioTrack]
    AT --> AF[AudioFlinger]
    AF --> A2DP["A2DP HAL<br/>(编码器选择)"]
    A2DP --> |"SBC/AAC/LDAC/LC3"| BT_STACK[Bluetooth Stack]
    BT_STACK --> |"HCI"| BT_HW[蓝牙芯片]
    BT_HW --> |"无线"| HEADPHONE[蓝牙耳机]
```

---

## 4. 语音通信编解码格式

### 4.1 窄带/宽带语音编解码

| 格式 | 带宽 | 采样率 | 码率 | 场景 |
|:---|:---|:---|:---|:---|
| **G.711** | 窄带 | 8kHz | 64 kbps | PSTN 电话 |
| **G.729** | 窄带 | 8kHz | 8 kbps | VoIP |
| **AMR-NB** | 窄带 | 8kHz | 4.75-12.2 kbps | 2G/3G 通话 |
| **AMR-WB** | 宽带 | 16kHz | 6.6-23.85 kbps | VoLTE HD Voice |
| **EVS** | 超宽带 | 8-48kHz | 5.9-128 kbps | VoLTE 演进 |
| **Opus** | 全频段 | 8-48kHz | 6-510 kbps | WebRTC/OTT |

### 4.2 编解码器选择链路 (Android VoIP)

```mermaid
graph TD
    CALL[通话建立] --> CODEC_NEG["编解码器协商<br/>(SDP Offer/Answer)"]
    CODEC_NEG --> |"VoLTE"| EVS_AMR["EVS > AMR-WB > AMR-NB"]
    CODEC_NEG --> |"WiFi 通话"| OPUS_AAC["Opus > AAC-ELD"]
    CODEC_NEG --> |"蓝牙 HFP"| CVSD_MSBC["mSBC (宽带) > CVSD (窄带)"]
```

---

## 5. 沉浸式音频编解码格式

| 格式 | 标准组织 | 核心特性 | 应用 |
|:---|:---|:---|:---|
| **Dolby AC-4** | Dolby | 对象音频 + 对话增强 | Atmos 流媒体 |
| **MPEG-H 3D Audio** | MPEG | 场景 + 对象 + HOA | Sony 360RA、广播 |
| **DTS:X** | DTS | 对象音频 + 无损 | 蓝光碟、家庭影院 |
| **IAMF** | Alliance for Open Media | 开放标准、免版税 | 流媒体、XR |

---

## 6. 编解码格式选择指南

```mermaid
graph TD
    START{应用场景?} --> MUSIC[音乐流媒体]
    START --> VOICE[语音通话]
    START --> BT[蓝牙音频]
    START --> ARCHIVE[无损存档]
    START --> IMMERSIVE[沉浸式音频]
    
    MUSIC --> |"通用"| AAC_LC["AAC-LC 256kbps"]
    MUSIC --> |"低带宽"| HE_AAC["HE-AAC v2"]
    MUSIC --> |"Hi-Res"| FLAC_OUT["FLAC / ALAC"]
    
    VOICE --> |"WebRTC"| OPUS_OUT["Opus"]
    VOICE --> |"VoLTE"| EVS_OUT["EVS / AMR-WB"]
    
    BT --> |"LE Audio"| LC3_OUT["LC3"]
    BT --> |"经典蓝牙高音质"| LDAC_OUT["LDAC / aptX Adaptive"]
    
    ARCHIVE --> FLAC_ARC["FLAC"]
    
    IMMERSIVE --> |"Dolby"| AC4["AC-4 / DD+JOC"]
    IMMERSIVE --> |"开放"| IAMF_OUT["IAMF"]
```

---

## 7. Android MediaCodec 编解码集成

### 7.1 Android 编解码框架架构

```
Android 音频编解码栈:

  App (MediaPlayer / ExoPlayer / AudioTrack)
    │
    ▼
  MediaCodec API (Java / NDK)
    │
    ▼
  ACodec / CCodec (Codec2)    ← Android 10+ 推荐 Codec2
    │
    ├── Software Codec (CPU):
    │     libopus.so / libFLAC.so / libAAC.so
    │     → 通用兼容，CPU 开销大
    │
    └── Hardware Codec (DSP/HW):
          vendor 提供 (如高通 DSP AAC Offload)
          → 低功耗，但支持格式有限
          
  选择逻辑:
    MediaCodec.createDecoderByType("audio/opus")
      → 系统查 media_codecs.xml
      → 优先选择 HW codec, 若不可用 fallback 到 SW
```

### 7.2 MediaCodec 编解码代码示例

```java
// === 使用 MediaCodec 解码 AAC ===
MediaExtractor extractor = new MediaExtractor();
extractor.setDataSource(audioFilePath);

// 找到音频 track
int audioTrack = -1;
for (int i = 0; i < extractor.getTrackCount(); i++) {
    MediaFormat format = extractor.getTrackFormat(i);
    String mime = format.getString(MediaFormat.KEY_MIME);
    if (mime.startsWith("audio/")) {
        audioTrack = i;
        break;
    }
}
extractor.selectTrack(audioTrack);

// 创建解码器
MediaFormat format = extractor.getTrackFormat(audioTrack);
MediaCodec decoder = MediaCodec.createDecoderByType(
    format.getString(MediaFormat.KEY_MIME));
decoder.configure(format, null, null, 0);
decoder.start();

// 异步解码循环 (简化)
while (!eos) {
    // 送入编码数据
    int inIdx = decoder.dequeueInputBuffer(10000);
    if (inIdx >= 0) {
        ByteBuffer inBuf = decoder.getInputBuffer(inIdx);
        int sampleSize = extractor.readSampleData(inBuf, 0);
        if (sampleSize < 0) {
            decoder.queueInputBuffer(inIdx, 0, 0, 0,
                MediaCodec.BUFFER_FLAG_END_OF_STREAM);
            eos = true;
        } else {
            decoder.queueInputBuffer(inIdx, 0, sampleSize,
                extractor.getSampleTime(), 0);
            extractor.advance();
        }
    }
    
    // 取出 PCM 数据
    MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();
    int outIdx = decoder.dequeueOutputBuffer(info, 10000);
    if (outIdx >= 0) {
        ByteBuffer outBuf = decoder.getOutputBuffer(outIdx);
        // outBuf 中是解码后的 PCM 数据 → 写入 AudioTrack
        audioTrack.write(outBuf, info.size, AudioTrack.WRITE_BLOCKING);
        decoder.releaseOutputBuffer(outIdx, false);
    }
}
```

```java
// === 使用 MediaCodec 编码 Opus (录音压缩) ===
MediaFormat encFormat = MediaFormat.createAudioFormat(
    MediaFormat.MIMETYPE_AUDIO_OPUS, 48000, 1);
encFormat.setInteger(MediaFormat.KEY_BIT_RATE, 64000);
encFormat.setInteger(MediaFormat.KEY_COMPLEXITY, 5);

MediaCodec encoder = MediaCodec.createEncoderByType(
    MediaFormat.MIMETYPE_AUDIO_OPUS);
encoder.configure(encFormat, null, null,
    MediaCodec.CONFIGURE_FLAG_ENCODE);
encoder.start();
// 然后将 AudioRecord 的 PCM 数据送入 encoder...
```

---

## 8. 硬件编解码 Offload

### 8.1 Offload 播放原理

```
Offload 播放 = 将编解码从 AP CPU 卸载到 DSP:

  普通播放:
    App → MediaCodec (CPU 解码) → PCM → AudioFlinger → HAL → DAC
    CPU 持续工作, 功耗高
    
  Offload 播放:
    App → 压缩数据 → AudioFlinger → Compress Offload HAL → DSP 解码 → DAC
    AP 可以 suspend! 功耗极低
    
  ┌─────────────────────────────────────────────┐
  │ Normal Path (CPU decode)                    │
  │ App → [CPU: AAC→PCM] → AF → HAL → DAC      │
  │ CPU: 活跃, 功耗 ~50mW+                     │
  ├─────────────────────────────────────────────┤
  │ Offload Path (DSP decode)                   │
  │ App → [压缩数据] → AF → DSP → DAC           │
  │ CPU: suspend, 功耗 ~5mW                    │
  └─────────────────────────────────────────────┘
```

### 8.2 Android Offload 支持的格式

| 格式 | 高通 ADSP | MTK DSP | Exynos | 备注 |
|:---|:---|:---|:---|:---|
| **AAC-LC** | ✅ | ✅ | ✅ | 最常见 Offload 格式 |
| **MP3** | ✅ | ✅ | ✅ | |
| **FLAC** | ✅ | ✅ | ⚠️ | 无损也可 Offload |
| **ALAC** | ✅ | ⚠️ | ⚠️ | Apple 格式 |
| **WMA** | ✅ | ⚠️ | ❌ | |
| **Opus** | ⚠️ 部分 | ❌ | ❌ | 较新, 支持有限 |
| **Vorbis** | ✅ | ⚠️ | ❌ | OGG 容器 |
| **DSD** | ✅ (DoP) | ❌ | ❌ | Hi-Fi 玩家 |

```bash
# 检查设备支持的 Offload 格式
adb shell dumpsys media.audio_policy | grep -i offload
# 或
adb shell cat /vendor/etc/audio_policy_configuration.xml | grep -i compress
```

### 8.3 Offload 播放代码路径

```java
// App 不需要特殊处理, 系统自动选择 Offload:
// 条件:
//   1. 单独的音乐播放 (非混音场景)
//   2. 格式在 Offload 支持列表中
//   3. 没有绑定需要 PCM 的音效 (如 Visualizer)
//
// AudioFlinger 判断逻辑:
//   → 检查 format 是否在 offload 列表
//   → 检查 AudioPolicy 是否允许 offload
//   → 创建 OffloadThread (而非 MixerThread)
//   → 压缩数据直接送 Compress Offload HAL
//
// 禁用 Offload (如需要 PCM 做可视化):
AudioTrack track = new AudioTrack.Builder()
    .setOffloadedPlayback(false)  // Android 11+
    .build();
```

---

## 9. 编解码质量基准对比

```
主观音质评测 (MUSHRA / ABX 测试) 近似排名:

  码率: 64kbps Stereo 48kHz
  ─────────────────────────────────────
  Opus          ████████████████░ 88/100
  xHE-AAC       ███████████████░░ 85/100
  AAC-LC        ███████████░░░░░░ 72/100
  MP3           ██████████░░░░░░░ 68/100
  SBC           ██████░░░░░░░░░░░ 52/100
  
  码率: 128kbps Stereo 48kHz
  ─────────────────────────────────────
  Opus          ██████████████████ 95/100
  AAC-LC        █████████████████░ 93/100
  MP3           ███████████████░░░ 88/100
  Vorbis        ██████████████░░░░ 86/100
  
  透明码率 (无法分辨与原始PCM):
    Opus:   ~160 kbps
    AAC-LC: ~192 kbps
    MP3:    ~256 kbps
    
  注: 以上数据为典型测试汇总参考, 实际因内容而异
```

---

## 10. 关键参考 (References)

1.  [Opus Codec Official](https://opus-codec.org/)
2.  [Bluetooth LE Audio Specification](https://www.bluetooth.com/learn-about-bluetooth/recent-enhancements/le-audio/)
3.  [Fraunhofer - AAC](https://www.iis.fraunhofer.de/en/ff/amm/consumer-electronics/aac.html)
4.  [FLAC - Free Lossless Audio Codec](https://xiph.org/flac/)
5.  [Dolby AC-4 Technical Overview](https://professional.dolby.com/tv/dolby-ac-4/)
6.  [IETF RFC 6716 - Opus](https://datatracker.ietf.org/doc/html/rfc6716)
7.  [Android MediaCodec API](https://developer.android.com/reference/android/media/MediaCodec)
8.  [Android Compress Offload](https://source.android.com/docs/core/audio/offload)
