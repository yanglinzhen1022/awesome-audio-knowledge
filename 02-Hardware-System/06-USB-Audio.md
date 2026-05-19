# USB 音频 (USB Audio)

USB Audio 是连接外部 DAC、耳机、声卡、麦克风的主要数字接口。Android 手机接 USB DAC、车载系统接 USB 麦克风阵列、专业录音棚用 USB 声卡——都离不开 USB Audio Class (UAC) 协议。本章从协议规范到 Android/Linux 驱动全链路深入讲解。

---

## 1. USB Audio 协议基础

### 1.1 USB Audio Class (UAC) 版本对比

| 特性 | UAC 1.0 | UAC 2.0 | UAC 3.0 |
|:---|:---|:---|:---|
| **USB 规范** | USB 1.1+ | USB 2.0+ | USB 3.x |
| **最高采样率** | 96 kHz (实际受限带宽) | 384 kHz | 384 kHz+ |
| **最大位深** | 24-bit | 32-bit | 32-bit |
| **最大声道数** | ~8 ch | ~32 ch | ~128 ch |
| **异步模式** | 支持 | 支持 (推荐) | 支持 |
| **Type-C** | 不涉及 | 兼容 | 原生支持 |
| **功耗管理** | 无 | 基本 | 完善 (LPM) |
| **Android 支持** | ✅ 原生 | ✅ 原生 (5.0+) | ⚠️ 部分 |
| **Linux 支持** | ✅ `snd-usb-audio` | ✅ `snd-usb-audio` | ⚠️ 进行中 |

### 1.2 USB Audio 描述符结构

```
USB Audio 设备的描述符层级:

  Device Descriptor
    └── Configuration Descriptor
          ├── Interface 0: AudioControl (AC)  ← 控制接口
          │     ├── Header Descriptor
          │     ├── Input Terminal (IT)   ← 音频输入端点 (如麦克风)
          │     ├── Output Terminal (OT)  ← 音频输出端点 (如扬声器)
          │     ├── Feature Unit (FU)     ← 音量/静音/EQ 控制
          │     ├── Mixer Unit            ← 混音 (可选)
          │     ├── Selector Unit         ← 输入选择 (可选)
          │     └── Clock Source (UAC2)   ← 时钟描述 (UAC2 新增)
          │
          ├── Interface 1: AudioStreaming (AS) — Playback  ← 播放
          │     ├── Alt Setting 0: Zero-bandwidth (idle)
          │     ├── Alt Setting 1: S16_LE 2ch 48kHz
          │     ├── Alt Setting 2: S24_LE 2ch 96kHz
          │     └── Endpoint: Isochronous OUT
          │           ├── Sync Type: Async / Adaptive / Sync
          │           └── Feedback Endpoint (Async 模式)
          │
          └── Interface 2: AudioStreaming (AS) — Capture   ← 录音
                ├── Alt Setting 0: Zero-bandwidth
                ├── Alt Setting 1: S16_LE 1ch 48kHz
                └── Endpoint: Isochronous IN
```

### 1.3 USB 传输模式

```
USB Audio 使用 Isochronous (等时) 传输:

  特点:
    - 保证带宽, 固定间隔传输 (每 1ms 或 125µs)
    - 无重传: 数据丢了就丢了 (实时性优先)
    - 无握手确认
    
  三种同步模式:
  
  ┌──────────────────────────────────────────────────────────┐
  │ Synchronous (同步)                                      │
  │   主机和设备使用同一时钟 (USB SOF)                       │
  │   问题: USB 时钟精度不够, 长期有漂移                    │
  │   用途: 低端设备                                        │
  ├──────────────────────────────────────────────────────────┤
  │ Adaptive (自适应)                                       │
  │   设备跟随主机时钟, 动态调整自己的采样率                │
  │   问题: 设备被迫锁定到非理想频率 → Jitter 较大          │
  │   用途: 中端设备                                        │
  ├──────────────────────────────────────────────────────────┤
  │ Asynchronous (异步) — 推荐!                             │
  │   设备使用自己的高精度时钟, 通过 Feedback Endpoint       │
  │   告诉主机当前实际消耗速率                              │
  │   主机动态调整发送速率                                  │
  │   优势: 最低 Jitter, 音质最好                           │
  │   用途: Hi-Fi DAC, 专业声卡                             │
  └──────────────────────────────────────────────────────────┘
  
  Async 模式的 Feedback 机制:
    设备 → 主机:  "我当前实际消耗速率是 47999.8 frames/s"
    主机调整: 下一个 USB 帧少发 1 个 sample
    → 长期精确同步, 无 underrun/overrun
```

### 1.4 USB Audio 带宽计算

```
USB 2.0 High-Speed 带宽: 480 Mbps (理论), 实际可用 ~300 Mbps

  所需带宽 = Sample Rate × Bit Depth × Channels × 1.1 (协议开销)
  
  示例:
    S16 Stereo 48kHz:   48000 × 16 × 2 × 1.1 = ~1.7 Mbps   ✅ 轻松
    S24 Stereo 192kHz:  192000 × 24 × 2 × 1.1 = ~10.1 Mbps  ✅ 没问题
    S32 32ch 96kHz:     96000 × 32 × 32 × 1.1 = ~108 Mbps   ⚠️ 接近极限
    
  USB 1.1 Full-Speed: 12 Mbps
    S16 Stereo 48kHz: ~1.7 Mbps  ✅
    S24 Stereo 96kHz: ~5.1 Mbps  ✅
    S24 8ch 48kHz:    ~10.2 Mbps ❌ 超出!
```

---

## 2. Linux USB Audio 驱动

### 2.1 驱动架构

```
Linux USB Audio 驱动栈:

  用户空间
    ├── aplay / arecord / PulseAudio / PipeWire
    └── ALSA userspace API (libasound)
  ────────────────────────────────────────────
  内核空间
    ├── ALSA Core (snd_pcm, snd_ctl)
    ├── snd-usb-audio 模块  ← USB Audio 核心驱动
    │   ├── card.c          → 声卡注册
    │   ├── pcm.c           → PCM 流管理
    │   ├── stream.c        → 解析 AS 描述符
    │   ├── endpoint.c      → URB 提交与回调
    │   ├── clock.c         → 时钟源管理 (UAC2)
    │   ├── mixer.c         → Feature Unit → ALSA controls
    │   └── quirks.c        → 设备兼容性修补
    └── USB Core (usb-core, xhci-hcd)
  ────────────────────────────────────────────
  硬件
    └── USB Host Controller (xHCI / EHCI)
        └── USB Cable → USB Audio Device
        
  关键源码位置:
    sound/usb/         ← snd-usb-audio 驱动
    sound/usb/card.c   ← probe() 入口
    sound/usb/quirks.c ← 各种设备 quirk 修补
```

### 2.2 URB (USB Request Block) 数据流

```
USB Audio 数据传输:

  播放 (Playback):
    AudioFlinger → ALSA write → snd-usb-audio
      → 填充 URB buffer (ISO OUT)
      → 提交 URB 给 USB HC
      → USB HC 按 1ms/125µs 间隔发送到设备
      → 设备 DAC 播放
      
  录音 (Capture):
    设备 ADC 采样
      → USB HC 按间隔接收 ISO IN 数据
      → URB 完成回调 → 拷贝到 ALSA ring buffer
      → ALSA read → AudioFlinger
      
  URB 双缓冲:
    ┌──────┐     ┌──────┐
    │URB 0 │ ──→ │USB HC│ ──→ 设备
    └──────┘     └──────┘
    ┌──────┐        ↑
    │URB 1 │ ──→ ───┘  (URB 0 完成后提交 URB 1)
    └──────┘
    CPU 填写 URB 0 ← 同时 USB HC 传输 URB 1
```

### 2.3 常用调试命令

```bash
# ==================== USB Audio 设备识别 ====================
# 查看已连接的 USB Audio 设备
lsusb | grep -i audio
# 或
cat /proc/asound/cards
#  2 [USB_DAC        ]: USB-Audio - USB DAC
#                       FiiO K5 Pro at usb-0000:00:14.0-1

# 查看 USB Audio 描述符 (极其详细)
cat /proc/asound/card2/usbid       # USB VID:PID
cat /proc/asound/card2/stream0     # 支持的格式/采样率

# ==================== Android USB Audio ====================
adb shell cat /proc/asound/cards
adb shell dumpsys media.audio_flinger | grep -i usb
adb shell dumpsys usb

# 查看 USB Audio 路由
adb shell dumpsys media.audio_policy | grep -i usb

# ==================== 兼容性问题排查 ====================
# 内核日志
dmesg | grep -iE "usb|snd-usb|audio"

# 常见错误:
#   "cannot set freq 48000 to ep 0x01" → 采样率不支持
#   "no valid URB" → 带宽不足或端点异常
#   "clock source xx is not valid" → UAC2 时钟问题

# ==================== USB Audio quirks ====================
# 某些设备需要特殊处理 (quirk):
# sound/usb/quirks-table.h 中硬编码
# 或通过模块参数: 
#   modprobe snd-usb-audio vid=0x1234 pid=0x5678 quirk_alias=...
```

---

## 3. Android USB Audio

### 3.1 Android USB Audio 架构

```
Android USB Audio 全链路:

  App (AudioTrack / MediaPlayer)
    │
    ▼
  AudioFlinger
    │
    ▼
  AudioPolicyService
    │  → 检测到 USB Audio 设备连接
    │  → 将 USAGE_MEDIA 路由到 USB 输出
    │
    ▼
  USB Audio HAL (audio.usb.default.so)
    │  → 打开 ALSA PCM 设备 (TinyALSA)
    │  → 配置采样率/格式/声道
    │  → 处理格式转换 (如需要)
    │
    ▼
  Kernel: snd-usb-audio
    │  → URB → USB HC → USB Device
    │
    ▼
  USB DAC / Headphone / Mic

  关键 HAL 源码:
    hardware/libhardware/modules/usbaudio/  (Legacy)
    frameworks/av/media/libaudiohal/impl/   (AIDL)
```

### 3.2 USB Audio 设备连接流程

```
USB Audio 设备插入 → Android 响应流程:

  1. USB HC 检测到设备连接
     └── Kernel: usb_new_device() → 枚举描述符
     
  2. snd-usb-audio 驱动 probe
     └── 创建 ALSA 声卡 → /dev/snd/pcmCxDxp, controlCx
     
  3. AudioPolicyService 收到 ALSA 声卡添加事件
     └── DeviceDescriptor: AUDIO_DEVICE_OUT_USB_DEVICE
     
  4. AudioPolicyService 执行路由策略
     └── 如果是输出设备: 将媒体/通话路由切换到 USB
     └── 如果是输入设备: 将录音路由切换到 USB Mic
     
  5. USB Audio HAL 打开 PCM 设备
     └── 格式协商: 查询设备支持的 format/rate/channels
     └── 选择最佳匹配 (优先: 设备原生格式, 避免 SRC)
     
  6. AudioFlinger 开始向 USB HAL 写数据
     └── 音频从扬声器切换到 USB DAC 输出
     
  总延迟变化:
    内置 Speaker: ~30-50ms
    USB DAC:      ~50-100ms (多了 USB 传输延迟)
```

### 3.3 常见 USB Audio 问题

| # | 问题 | 原因 | 解决方案 |
|:---|:---|:---|:---|
| 1 | 插入 USB DAC 无声 | HAL 未识别 / 路由未切换 | 检查 `dumpsys media.audio_policy` |
| 2 | 爆音/卡顿 | Buffer 太小 / USB 带宽不足 | 增大 period_size, 降低采样率 |
| 3 | 采样率被降级 | 设备不支持 48kHz | 查 `stream0` 描述符, 添加 SRC |
| 4 | 音量无法调节 | Feature Unit 缺失 | HAL 层软件音量补偿 |
| 5 | 拔出后无声 | 路由未切回 Speaker | 检查 AudioPolicy 设备断开处理 |
| 6 | 声道映射错误 | UAC Channel Cluster 不标准 | 添加设备 quirk |
| 7 | 延迟过高 | USB 固有延迟 + 大 buffer | 使用异步模式 DAC, 减小 buffer |
| 8 | 部分设备不识别 | USB 描述符不合规 | 内核 quirks-table.h 修补 |

---

## 4. USB Type-C 音频

### 4.1 Type-C 音频模式

```
USB Type-C 音频的两种模式:

  ┌──────────────────────────────────────────────────────────┐
  │ 模式 1: 模拟音频 (Analog Audio Accessory Mode)         │
  │                                                          │
  │   Type-C 口复用 2 个 pin 输出模拟信号:                  │
  │     SBU1/SBU2 → 左/右声道模拟信号                       │
  │     CC pin → 检测耳机插入                               │
  │                                                          │
  │   特点:                                                  │
  │     - 手机内部 DAC 输出 → Type-C → 耳机                 │
  │     - 不需要 USB Audio 协议                             │
  │     - Android 11+ 已废弃此模式                          │
  ├──────────────────────────────────────────────────────────┤
  │ 模式 2: 数字音频 (USB Audio over Type-C) — 主流         │
  │                                                          │
  │   标准 USB Audio Class 协议:                            │
  │     手机 → USB 数字数据 → Type-C 线 → 耳机内置 DAC     │
  │                                                          │
  │   特点:                                                  │
  │     - 耳机内置 DAC + Amp (如 Apple EarPods USB-C)       │
  │     - 支持高品质音频 (24-bit/96kHz)                     │
  │     - 需要 USB Audio HAL 支持                           │
  │     - 可能有兼容性问题                                  │
  └──────────────────────────────────────────────────────────┘
```

### 4.2 Type-C 耳机检测流程

```
Type-C 数字耳机插入检测:

  1. CC pin 检测到 UFP (Upstream Facing Port)
  2. USB 枚举 → 识别为 USB Audio Class 设备
  3. snd-usb-audio probe → 创建声卡
  4. AudioPolicy: AUDIO_DEVICE_OUT_USB_HEADSET
  5. 路由切换: Speaker → USB Headset
  6. 通知 App: AudioManager.ACTION_HEADSET_PLUG

  vs 3.5mm 耳机:
    3.5mm: 纯硬件检测 (MBHC / GPIO) → 内置 Codec 路径
    USB-C: USB 枚举 → USB Audio HAL → snd-usb-audio
```

---

## 5. 专业 USB 声卡与多通道

```
专业 USB 声卡 (如 Focusrite Scarlett, RME Babyface):

  特点:
    - 多通道 I/O (4in/4out, 8in/8out)
    - 低延迟驱动 (ASIO on Windows, JACK on Linux)
    - 支持 24-bit/192kHz
    - 异步模式 + 高精度时钟
    
  Linux 专业音频配置:
    # 使用 JACK 低延迟音频服务器
    jackd -d alsa -d hw:USB_DAC -r 48000 -p 64 -n 2
    # -p 64: 64 frames/period ≈ 1.33ms
    # -n 2:  2 periods ≈ 2.67ms 往返延迟
    
  多通道映射:
    USB Audio 的 Channel Cluster:
      Front Left, Front Right, Front Center, LFE,
      Back Left, Back Right, Side Left, Side Right
    
    需要在 ALSA / PipeWire 中正确映射:
      aplay -D hw:2,0 --channels=8 multi_channel.wav
```

---

## 6. 关键参考 (References)

1.  [USB Audio Class 2.0 Specification](https://www.usb.org/document-library/audio-devices-rev-20-and-adopters-agreement)
2.  [Linux snd-usb-audio 源码](https://github.com/torvalds/linux/tree/master/sound/usb)
3.  [Android USB Audio HAL](https://source.android.com/docs/core/audio/usb)
4.  [USB Type-C Audio - Android](https://source.android.com/docs/core/audio/usb-headset-spec)
5.  [USB in a NutShell - Isochronous Transfers](https://www.beyondlogic.org/usbnutshell/usb4.shtml)

---
*返回：[Codec 与 SmartPA](./05-Codec-SmartPA.md) | [模块目录](./README.md)*
