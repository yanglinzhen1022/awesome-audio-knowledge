# VoIP 与通话音频链路 (VoIP & Voice Call Chain)

语音通话是音频系统中最复杂、最苛刻的场景——低延迟、全双工、实时 3A 处理、多网络制式并存。本章从 Android 应用层到 Modem/网络层，全链路拆解语音通话的音频数据流。

---

## 1. 通话类型全景

```
Android 设备上的语音通话方式:

  ┌──────────────────────────────────────────────────────────┐
  │ 电路域通话 (CS Voice) — 传统蜂窝                        │
  │   2G GSM: AMR-NB (8kHz, 12.2kbps)                       │
  │   3G WCDMA: AMR-WB (16kHz, 23.85kbps)                   │
  │   路径: AP ←→ Modem DSP (音频不经过 AP 处理!)           │
  ├──────────────────────────────────────────────────────────┤
  │ VoLTE (Voice over LTE) — 4G 高清通话                    │
  │   Codec: EVS / AMR-WB (16kHz) / AMR-WB+ (32kHz)        │
  │   路径: AP ←→ IMS ←→ Modem (可能经过 AP 3A)            │
  ├──────────────────────────────────────────────────────────┤
  │ VoNR (Voice over NR) — 5G 通话                          │
  │   Codec: EVS (最高 48kHz SWB/FB)                        │
  │   路径: 类似 VoLTE, 基于 IMS                            │
  ├──────────────────────────────────────────────────────────┤
  │ VoIP (Voice over IP) — 网络通话                         │
  │   App: WeChat / Teams / Zoom / WhatsApp                  │
  │   Codec: Opus (推荐) / SILK / iLBC / G.711              │
  │   路径: App → AudioTrack/Record → AudioFlinger → HAL    │
  │   3A: App 自带 或 使用系统 WebRTC                       │
  ├──────────────────────────────────────────────────────────┤
  │ Wi-Fi Calling (VoWiFi)                                  │
  │   IMS over WiFi, 体验同 VoLTE                           │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. VoLTE/VoNR 通话链路

### 2.1 端到端架构

```
VoLTE 通话全链路:

  本端                                              对端
  ────                                              ────
  麦克风                                            扬声器
    │                                                 ↑
    ▼                                                 │
  Codec ADC                                       Codec DAC
    │                                                 ↑
    ▼                                                 │
  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐
  │ ADSP (Audio DSP)                │    │ ADSP                            │
  │  ├── NS  (降噪)                │    │  ├── EC 参考信号                │
  │  ├── AEC (回声消除)            │    │  └── DRC / Volume              │
  │  ├── AGC (增益控制)            │    │                                 │
  │  └── 上行增强                  │    │                                 │
  └─────────────────────────────────┘    └─────────────────────────────────┘
    │ (PCM 16kHz)                             ↑ (PCM 16kHz)
    ▼                                         │
  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐
  │ Modem DSP                       │    │ Modem DSP                       │
  │  ├── 语音编码 (AMR-WB/EVS)     │    │  ├── 语音解码 (AMR-WB/EVS)     │
  │  ├── Jitter Buffer              │    │  ├── PLC (丢包补偿)            │
  │  └── DTMF 生成                  │    │  └── DTMF 检测                  │
  └─────────────────────────────────┘    └─────────────────────────────────┘
    │ (RTP/RTCP over IP)                      ↑
    ▼                                         │
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                        LTE / 5G NR 空口                                │
  │  UE → eNodeB/gNB → S-GW → P-GW → IMS Core → P-GW → eNodeB → UE     │
  └─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 高通平台 VoLTE 音频路径

```
高通平台 (SM8550 / SA8295) VoLTE 音频路径:

  上行 (Tx / 录音方向):
    麦克风 → Codec ADC → LPAIF TDM/MI2S
      → ADSP SPF Graph [Tx Path]:
          EC Ref (从 Rx 路径回环)
            → AEC (回声消除)
            → NS (降噪)
            → AGC (增益控制)
      → CS-Voice Tx (PCM 16kHz)
      → 送给 Modem DSP 编码
      → RTP 打包 → 发送

  下行 (Rx / 播放方向):
    接收 RTP → Modem DSP 解码
      → CS-Voice Rx (PCM 16kHz)
      → ADSP SPF Graph [Rx Path]:
          → DRC (动态范围控制)
          → EQ (听觉补偿)
          → Volume
          → EC Ref 分出 (送给 Tx AEC)
      → LPAIF → Codec DAC → 听筒/扬声器

  音频不经过 AP (audioserver):
    VoLTE 通话时, PCM 数据在 ADSP ↔ Modem 之间直连
    AP 只负责: 建立/拆除 Graph, 设置音量, 切换设备
    → 低延迟, 低功耗
```

### 2.3 通话 AudioPolicy 路由

```
通话时 AudioPolicy 的路由决策:

  正常通话 (手持):
    输出: AUDIO_DEVICE_OUT_EARPIECE (听筒)
    输入: AUDIO_DEVICE_IN_BUILTIN_MIC (底部主麦)
    
  免提 (Speaker):
    输出: AUDIO_DEVICE_OUT_SPEAKER
    输入: AUDIO_DEVICE_IN_BACK_MIC (顶部副麦, 用于 AEC 参考)
    
  有线耳机:
    输出: AUDIO_DEVICE_OUT_WIRED_HEADSET
    输入: AUDIO_DEVICE_IN_WIRED_HEADSET (耳机麦)
    
  蓝牙 SCO:
    输出: AUDIO_DEVICE_OUT_BLUETOOTH_SCO_HEADSET
    输入: AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET
    编码: mSBC (16kHz) 或 CVSD (8kHz)
    
  路由切换命令:
    通话中插入耳机 → AudioPolicy 自动切换
    用户按免提按钮 → InCallService → AudioManager.setSpeakerphoneOn()
    → AudioPolicy 切换输出设备 + 重配 ADSP Graph
```

---

## 3. VoIP 应用通话链路

### 3.1 VoIP 与 VoLTE 的关键差异

```
VoIP vs VoLTE 音频链路对比:

  ┌──────────────────────┬────────────────────┬──────────────────────┐
  │                      │ VoLTE              │ VoIP (微信等)        │
  ├──────────────────────┼────────────────────┼──────────────────────┤
  │ 3A 处理位置          │ ADSP (硬件)       │ App 内 (软件)        │
  │ 编解码位置           │ Modem DSP          │ App CPU              │
  │ 音频经过 AP?        │ 否 (ADSP↔Modem)   │ 是 (AudioTrack/Record)│
  │ 延迟                 │ ~80-150ms          │ ~150-400ms           │
  │ 编解码器             │ AMR-WB / EVS       │ Opus / SILK / iLBC   │
  │ 网络                 │ 运营商 QoS 保证    │ 无保证 (Best Effort) │
  │ 丢包处理             │ Modem PLC          │ App 自带 FEC/PLC     │
  │ Jitter Buffer        │ Modem 管理         │ App 自适应           │
  │ 功耗                 │ 低 (DSP offload)   │ 高 (AP 持续运行)     │
  └──────────────────────┴────────────────────┴──────────────────────┘
```

### 3.2 VoIP App 典型架构

```
VoIP App 通话音频模块架构:

  ┌────────────────────────────────────────────────────────┐
  │ VoIP App (微信 / Zoom / Teams)                        │
  │                                                        │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │ Audio Engine (通常基于 WebRTC)                   │  │
  │  │                                                  │  │
  │  │  录音线程:                                       │  │
  │  │    AudioRecord (16kHz, Mono, S16)                │  │
  │  │      → 3A 处理:                                 │  │
  │  │        ├── AEC (回声消除, 需要远端参考)          │  │
  │  │        ├── NS  (降噪, RNNoise 或 传统谱减法)    │  │
  │  │        └── AGC (自动增益)                        │  │
  │  │      → Encoder (Opus @ 24kbps)                   │  │
  │  │      → RTP 打包 → 网络发送                      │  │
  │  │                                                  │  │
  │  │  播放线程:                                       │  │
  │  │    网络接收 → RTP 解包                           │  │
  │  │      → Jitter Buffer (自适应缓冲)               │  │
  │  │      → Decoder (Opus)                            │  │
  │  │      → PLC (丢包隐藏)                           │  │
  │  │      → AudioTrack (16kHz, Mono, S16)            │  │
  │  │                                                  │  │
  │  │  AEC 参考:                                       │  │
  │  │    AudioTrack 播放数据 → 复制一份给 AEC 参考端  │  │
  │  └──────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────┘
```

### 3.3 VoIP AudioAttributes 配置

```java
// === VoIP App 录音配置 ===
AudioRecord recorder = new AudioRecord.Builder()
    .setAudioSource(MediaRecorder.AudioSource.VOICE_COMMUNICATION)
    // ↑ 关键! 启用系统 AEC/NS (如果可用)
    .setAudioFormat(new AudioFormat.Builder()
        .setSampleRate(16000)          // 通话标配 16kHz
        .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
        .setChannelMask(AudioFormat.CHANNEL_IN_MONO)
        .build())
    .setBufferSizeInBytes(bufferSize)
    .build();

// === VoIP App 播放配置 ===
AudioTrack player = new AudioTrack.Builder()
    .setAudioAttributes(new AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_VOICE_COMMUNICATION)
        // ↑ 告诉系统这是通话音频
        .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
        .build())
    .setAudioFormat(new AudioFormat.Builder()
        .setSampleRate(16000)
        .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
        .setChannelMask(AudioFormat.CHANNEL_OUT_MONO)
        .build())
    .setBufferSizeInBytes(bufferSize)
    .setPerformanceMode(AudioTrack.PERFORMANCE_MODE_LOW_LATENCY)
    .build();

// === 使用系统内置 AEC (推荐, 避免自研) ===
if (AcousticEchoCanceler.isAvailable()) {
    AcousticEchoCanceler aec = AcousticEchoCanceler.create(
        recorder.getAudioSessionId());
    aec.setEnabled(true);
}

if (NoiseSuppressor.isAvailable()) {
    NoiseSuppressor ns = NoiseSuppressor.create(
        recorder.getAudioSessionId());
    ns.setEnabled(true);
}
```

---

## 4. 通话音频 3A 处理

### 4.1 回声消除 (AEC) 在通话中的位置

```
为什么通话需要 AEC:

  场景: A 和 B 通话, A 说话
  
    A 说话 → 网络 → B 的扬声器播出
                        │
                        ▼ (声学耦合)
                     B 的麦克风拾取 A 的声音 + B 的声音
                        │
                        ▼
    如果不做 AEC: A 听到自己的回声! (延迟 150-400ms)
    
  AEC 在 B 端的处理:
    远端参考 (A 的声音, 从 AudioTrack 获取)
      ↓
    自适应滤波器 W(z)
      ↓ 估计出的回声信号 ŷ(n)
      ↓
    麦克风信号 d(n) - ŷ(n) = 近端语音 (B 说话)
      ↓
    发送给 A
    
  关键指标:
    ERLE (Echo Return Loss Enhancement): > 35dB
    收敛时间: < 500ms
    双讲 (Double-talk) 处理: 不能把 B 的声音当回声消掉
```

### 4.2 通话降噪 vs 媒体降噪

```
通话降噪 (NS) 特殊要求:

  通话 NS:                          媒体 NS:
    实时性: < 10ms 延迟算法延迟       可以有 20-40ms
    单声道: 通常 1ch 16kHz            可能 Stereo 48kHz
    保护语音: 不能损伤语音音质         可以更激进
    低复杂度: 需要 < 5% CPU           可以用大模型
    
  典型算法:
    传统: 谱减法 / Wiener Filter / MMSE
    现代: RNNoise (GRU) / DTLN (双阶段) / 高通 Fluence
    
  高通 Fluence:
    Single-Mic (FV5): 基于 ADSP, 单麦降噪
    Dual-Mic (FV5-Pro): 双麦波束成形 + 后置滤波
    Tri-Mic: 三麦
    → 全部在 ADSP 运行, 不占 AP CPU
```

---

## 5. Jitter Buffer 与网络对抗

### 5.1 概念

```
网络语音的核心挑战 — Jitter (抖动):

  发送端: 每 20ms 发一个 RTP 包 (均匀)
  网络中: 有的包快有的慢 (Jitter)
  接收端: 包到达时间不均匀
  
  发送:  |──20ms──|──20ms──|──20ms──|──20ms──|
  接收:  |──15ms──|──30ms──|──10ms──|──25ms──|  ← Jitter!
  
  如果直接播放: 有的包还没到就该播了 → 丢包 → 断续
  
  解决: Jitter Buffer (自适应缓冲区)
    ┌─────────────────────────────────────────┐
    │ 接收 → [Buffer 50-200ms] → 均匀播放    │
    │                                         │
    │ Buffer 太小: 来不及缓冲 → 丢包多        │
    │ Buffer 太大: 延迟太高 → 通话不自然      │
    │                                         │
    │ 自适应: 根据网络状况动态调整 buffer 大小 │
    │ 好网络: buffer 小 → 低延迟              │
    │ 差网络: buffer 大 → 容忍抖动            │
    └─────────────────────────────────────────┘
```

### 5.2 丢包隐藏 (PLC)

```
网络丢包时的处理策略:

  PLC (Packet Loss Concealment):
    丢包率 < 5%:  插值/重复上一帧 → 几乎不可闻
    丢包率 5-15%: 衰减重复 + 噪声注入 → 有感知但可接受
    丢包率 > 15%: 严重影响通话质量
    
  Opus 的前向纠错 (In-band FEC):
    每个包中嵌入上一个包的低码率冗余
    丢包后用冗余信息恢复 (质量略低但比 PLC 好)
    
  WebRTC 的 NetEQ:
    集成了 Jitter Buffer + PLC + Accelerate/Decelerate
    Accelerate: 网络恢复后, 加速播放消化 buffer 积压
    Decelerate: buffer 快空了, 减慢播放速度争取时间
```

---

## 6. 通话音频调试

### 6.1 关键调试命令

```bash
# ==================== 通话状态检查 ====================
# 当前通话模式
adb shell dumpsys audio | grep -i "mode"
# mode = MODE_IN_CALL (VoLTE) / MODE_IN_COMMUNICATION (VoIP)

# 通话路由
adb shell dumpsys media.audio_policy | grep -iE "active.*call|voice"
# 应该看到 EARPIECE / SPEAKER / BT_SCO 为活跃输出

# 通话音量
adb shell dumpsys audio | grep -i "voice_call"

# ==================== VoLTE/Modem 音频 ====================
# 检查 Voice Graph 是否建立 (高通)
adb shell cat /proc/asound/card0/agm_dump | grep -i voice
# 应看到: CS-Voice-Tx / CS-Voice-Rx graph STARTED

# 检查 Modem 通话状态
adb shell dumpsys telephony.registry | grep -i "call\|voice"

# ==================== VoIP 链路 ====================
# 检查 AudioSource = VOICE_COMMUNICATION 的录音流
adb shell dumpsys media.audio_flinger | grep -iA5 "VOICE_COMMUNICATION"

# 检查系统 AEC/NS 是否启用
adb shell dumpsys media.audio_flinger | grep -iE "AEC|NS|effect" 

# ==================== 通话质量问题排查 ====================
# 回声: 对端反馈听到自己的声音
#   → 检查 AEC 是否启用, ERLE 是否达标
#   → adb shell dumpsys media.audio_flinger | grep -i echo

# 对方听不清:
#   → 检查麦克风路由是否正确
#   → 检查 NS 是否过度降噪
#   → 抓取上行 PCM: 高通 QXDM 或 adb shell tinycap

# 断续/卡顿:
#   → VoIP: 检查 App 的 Jitter Buffer 统计
#   → VoLTE: 检查 RTP 丢包率 (AT 命令或 IMS log)

# ==================== 录制通话音频 (调试用) ====================
# 高通 ADSP loopback 抓取:
adb shell "echo 1 > /data/vendor/audio/pcm_dump_enable"
# PCM dump 文件: /data/vendor/audio/pcm_dump_*.pcm
```

### 6.2 通话质量评估指标

| 指标 | 全称 | 范围 | 说明 |
|:---|:---|:---|:---|
| **MOS** | Mean Opinion Score | 1-5 | 主观评分 (5=优秀, 3=可接受) |
| **PESQ** | Perceptual Evaluation of Speech Quality | -0.5~4.5 | ITU-T P.862, 窄带 |
| **POLQA** | Perceptual Objective Listening Quality | 1-5 | ITU-T P.863, 宽带/超宽带 |
| **E-Model (R值)** | E-Model Rating | 0-100 | ITU-T G.107, 综合评分 |
| **ERLE** | Echo Return Loss Enhancement | dB | > 35dB 为合格 |
| **单向延迟** | One-way Delay | ms | < 150ms 优秀, < 400ms 可接受 |
| **RTT** | Round-Trip Time | ms | < 300ms (端到端) |
| **丢包率** | Packet Loss Rate | % | < 3% 影响小, > 10% 严重 |

---

## 7. 通话编解码器对比

| 编解码器 | 采样率 | 码率 | 延迟 | MOS | 使用场景 |
|:---|:---|:---|:---|:---|:---|
| **G.711 µ-law** | 8 kHz | 64 kbps | 0.125ms | 4.1 | 固话/PSTN |
| **G.729** | 8 kHz | 8 kbps | 15ms | 3.9 | 低带宽 VoIP |
| **AMR-NB** | 8 kHz | 4.75-12.2 kbps | 20ms | 4.0 | 2G/3G |
| **AMR-WB** | 16 kHz | 6.6-23.85 kbps | 20ms | 4.5 | VoLTE/3G HD |
| **EVS** | 8-48 kHz | 5.9-128 kbps | 20ms | 4.7 | VoLTE/VoNR |
| **Opus** | 8-48 kHz | 6-510 kbps | 2.5-60ms | 4.6 | WebRTC/VoIP |
| **SILK** | 8-24 kHz | 6-40 kbps | 25ms | 4.3 | Skype/微信 |
| **iLBC** | 8 kHz | 13.3/15.2 kbps | 20/30ms | 3.9 | 旧版 WebRTC |
| **Lyra** | 16 kHz | 3.2 kbps | 20ms | 4.0 | Google 神经编码 |

---

## 8. 关键参考 (References)

1.  [Android Voice Call Audio - AOSP](https://source.android.com/docs/core/audio/voice-call)
2.  [WebRTC Native Code](https://webrtc.googlesource.com/src/)
3.  [RFC 7587 - RTP Payload Format for Opus](https://tools.ietf.org/html/rfc7587)
4.  [ITU-T P.863 - POLQA](https://www.itu.int/rec/T-REC-P.863)
5.  [3GPP TS 26.114 - IMS Media Handling](https://www.3gpp.org/DynaReport/26114.htm)
6.  [Qualcomm Voice Call Architecture](https://docs.qualcomm.com/bundle/80-PM164-51)

---
*返回：[Oboe/AAudio](./10-Oboe-AAudio.md) | [模块目录](./README.md)*
