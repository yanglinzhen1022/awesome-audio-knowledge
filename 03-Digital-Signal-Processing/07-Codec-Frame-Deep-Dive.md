# 音频编解码帧结构深度解析 (Codec Frame Deep Dive)

本章深入 AAC、Opus、LC3 三种核心编解码器的**比特流帧格式**，覆盖帧头解析、编码工具链、RTP 打包，为协议分析和底层调试提供参考。

> **前置阅读**：[05-Audio-Codec-Formats.md](./05-Audio-Codec-Formats.md) 覆盖编解码格式概览、选型指南和 Android MediaCodec 集成。

---

## 📑 目录

- [1. AAC 帧结构](#1-aac-帧结构)
- [2. Opus 帧结构](#2-opus-帧结构)
- [3. LC3 帧结构](#3-lc3-帧结构)
- [4. RTP 音频打包](#4-rtp-音频打包)
- [5. 实战：帧解析与调试](#5-实战帧解析与调试)

---

## 1. AAC 帧结构

### 1.1 封装格式对比

AAC 比特流有两种主要封装格式：

```
┌─────────────────────────────────────────────────────────────┐
│ ADTS (Audio Data Transport Stream)                          │
│ - 每帧独立可解码 (自包含 header)                              │
│ - 用于流媒体/广播 (DAB+/HLS)                                │
│ - 每帧 = ADTS Header (7/9 bytes) + Raw AAC Frame            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ LATM/LOAS (Low Overhead Audio Stream)                       │
│ - 更低开销的封装                                             │
│ - 用于 DVB/MPEG-TS 广播                                     │
│ - AudioSpecificConfig 可不重复                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Raw AAC in MP4/M4A (AudioSpecificConfig in esds box)         │
│ - 最紧凑，无冗余 header                                      │
│ - 用于文件存储 (MP4/M4A/MKV)                                 │
│ - 依赖容器提供 config                                        │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 ADTS 帧头详解 (7 bytes fixed + 2 bytes optional CRC)

```
ADTS Header (56 bits / 72 bits with CRC):

Byte 0-1: Sync Word + Flags
┌──────────────────────────────────────────────────────────────────┐
│ 1111 1111 │ 1111 │ X │ XX │ X │ XXXX XX │ X │ X │ XX │ XXXX     │
│ syncword  │      │ID │Lyr │CRC│ Profile  │SF │Prv│Ch  │FrameLen │
│ 0xFFF     │      │   │    │   │          │Idx│   │Cfg │          │
└──────────────────────────────────────────────────────────────────┘

字段详解:
  [12 bits] Sync Word         = 0xFFF (固定)
  [1 bit]   ID                = 0: MPEG-4, 1: MPEG-2
  [2 bits]  Layer             = 00 (固定)
  [1 bit]   Protection Absent = 1: 无 CRC (7 byte header)
                                0: 有 CRC (9 byte header)
  [2 bits]  Profile           = 00: Main, 01: LC, 10: SSR, 11: LTP
  [4 bits]  Sampling Freq Idx = 0x3: 48000Hz
                                0x4: 44100Hz
                                0x5: 32000Hz
                                0xB: 8000Hz
  [1 bit]   Private           = 0
  [3 bits]  Channel Config    = 1: Mono, 2: Stereo, 6: 5.1
  [1 bit]   Originality       = 0
  [1 bit]   Home              = 0
  [13 bits] Frame Length       = Header + Raw Frame 总字节数
  [11 bits] Buffer Fullness    = 0x7FF 表示 VBR
  [2 bits]  Num Raw Blocks     = 通常 0 (即 1 个 raw block)
```

### 1.3 AAC-LC 编码工具链

```
PCM 输入 (1024 samples/frame @ 48kHz = 21.3ms)
    │
    ▼
┌────────────────────────────────┐
│ 1. MDCT (Modified DCT)        │
│    1024 时域样本 → 1024 频域系数│
│    50% overlap → 无块间缝隙    │
│    窗函数: Long(1024) / Short(128)×8│
└────────────────┬───────────────┘
                 │
                 ▼
┌────────────────────────────────┐
│ 2. 心理声学模型 (Psychoacoustic)│
│    计算掩蔽阈值 (masking curve) │
│    确定每个频带可容忍的量化噪声 │
│    → 指导比特分配               │
└────────────────┬───────────────┘
                 │
                 ▼
┌────────────────────────────────┐
│ 3. 量化 & Scale Factor         │
│    根据掩蔽阈值量化 MDCT 系数    │
│    每个 Scale Factor Band 独立  │
│    → 低掩蔽区: 精细量化 (多比特) │
│    → 高掩蔽区: 粗量化 (少比特)  │
└────────────────┬───────────────┘
                 │
                 ▼
┌────────────────────────────────┐
│ 4. 无噪声编码 (Huffman)        │
│    对量化后的系数做熵编码        │
│    11 张 Huffman 码表           │
│    Sectioning: 相邻 SFB 共享码表│
└────────────────┬───────────────┘
                 │
                 ▼
┌────────────────────────────────┐
│ 5. 比特流组装                   │
│    ics_info + scale_factors     │
│    + spectral_data + fill_elements│
│    → 一个完整 raw_data_block    │
└────────────────────────────────┘
```

### 1.4 AAC Raw Frame 内部结构

```
raw_data_block() {
    id_syn_ele (3 bits)     // 元素类型: SCE=单声道, CPE=双声道, LFE, DSE, FIL
    
    single_channel_element() / channel_pair_element() {
        element_instance_tag (4 bits)
        
        individual_channel_stream() {
            ics_info() {
                window_sequence (2 bits)   // ONLY_LONG / LONG_START / EIGHT_SHORT / LONG_STOP
                window_shape (1 bit)       // KBD / Sine
                max_sfb (6 bits)           // 使用的 Scale Factor Band 数量
                predictor_data()           // Main Profile only
            }
            
            section_data() {              // Huffman 码表分配
                sect_cb[]                  // 每段使用哪张码表
                sect_len[]                 // 每段覆盖多少 SFB
            }
            
            scale_factor_data() {         // 量化步长 (DPCM 编码)
                差分编码的 scale factors
                // 解码: sf[i] = sf[i-1] + delta
            }
            
            spectral_data() {             // 核心频谱数据 (Huffman 编码)
                按 section 依次解码 MDCT 系数
            }
        }
    }
}
```

### 1.5 HE-AAC SBR 扩展

```
HE-AAC v1 = AAC-LC (低频) + SBR (高频重建):

  编码端:
    PCM (48kHz) → 下采样到 24kHz → AAC-LC 编码 (核心)
                → QMF 分析滤波器 → 提取高频包络 → SBR 参数 (很少比特)
    
  解码端:
    AAC-LC 解码 → 24kHz PCM
      + SBR 参数 → QMF 合成 → 利用低频谐波重建高频
      → 48kHz 全频带输出

  帧结构:
    [ADTS Header][AAC-LC raw frame][FIL element → SBR extension payload]
    
  SBR payload:
    bs_header_flag         → 是否包含 SBR header
    bs_freq_scale          → 频带划分方式
    bs_limiter_bands       → 限制器频带数
    envelope_data[]        → 高频包络 (时间/频率分辨率可选)
    noise_floor_data[]     → 噪声底数据
    
  码率节省: 48kbps HE-AAC ≈ 96kbps AAC-LC 音质
```

---

## 2. Opus 帧结构

### 2.1 Opus 比特流格式 (RFC 6716)

```
Opus Packet (一个 UDP/RTP payload):

┌─────────┬───────────────────────────────────────────────┐
│ TOC Byte│ Frame Data (可变长度)                          │
│ (1 byte)│                                               │
└─────────┴───────────────────────────────────────────────┘

TOC Byte (Table of Contents):
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ c │ c │ c │ c │ c │ s │ f │ f │
  │(config)     │(stereo)│(frames/pkt)│
  └───┴───┴───┴───┴───┴───┴───┴───┘
  
  config (5 bits): 编码配置
    [0-3]   SILK-only NB     (10/20/40/60ms)
    [4-7]   SILK-only MB     (10/20/40/60ms)
    [8-11]  SILK-only WB     (10/20/40/60ms)
    [12-13] Hybrid SWB       (10/20ms)
    [14-15] Hybrid FB        (10/20ms)
    [16-19] CELT-only NB     (2.5/5/10/20ms)
    [20-23] CELT-only WB     (2.5/5/10/20ms)
    [24-27] CELT-only SWB    (2.5/5/10/20ms)
    [28-31] CELT-only FB     (2.5/5/10/20ms)
    
  s (1 bit): 0=Mono, 1=Stereo
  
  ff (2 bits): 帧打包模式
    00: 1 frame per packet
    01: 2 equal-size frames
    10: 2 different-size frames
    11: 任意数量 frames (VBR, 带 frame lengths)
```

### 2.2 SILK 核心 (语音模式)

```
SILK 编码器内部 (基于 Skype SILK):

  输入: 8/12/16kHz 语音 PCM (10-60ms 帧)
    │
    ▼
  ┌──────────────────────────────────┐
  │ 1. 线性预测 (LP) 分析            │
  │    LPC 阶数: 10 (NB) / 16 (WB)  │
  │    使用 Burg 算法估计 LP 系数     │
  │    LP 系数 → LSF 量化             │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 2. 长时预测 (LTP / Pitch)        │
  │    基音周期检测 (Pitch lag)       │
  │    5 阶 LTP 滤波器               │
  │    → 去除基音周期性 (激励白化)   │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 3. 激励信号量化                   │
  │    残差信号 (excitation) 量化     │
  │    Shell coding + Range coding   │
  │    自适应码本 + 随机码本          │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 4. Range Encoder (算术编码)       │
  │    所有参数统一用 Range coding    │
  │    比 Huffman 更高效率            │
  └──────────────────────────────────┘

SILK 帧 bit allocation (20ms, WB, 20kbps):
  LSF 系数:         ~40 bits
  Pitch lag:        ~12 bits
  LTP 系数:         ~30 bits
  激励量化:         ~300 bits
  VAD/DTX flag:     ~2 bits
  ─────────────────────────
  总计:             ~384 bits = 19.2 kbps
```

### 2.3 CELT 核心 (音乐模式)

```
CELT 编码器内部 (基于 Xiph.org CELT):

  输入: 48kHz 音乐 PCM (2.5-20ms 帧)
    │
    ▼
  ┌──────────────────────────────────┐
  │ 1. MDCT 变换                      │
  │    帧长 2.5ms: 120 samples MDCT  │
  │    帧长 20ms:  960 samples MDCT  │
  │    窗函数: 正弦窗 + 50% overlap  │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 2. 频带能量 (Band Energy)         │
  │    21 个 Bark 频带               │
  │    编码每个频带的对数能量          │
  │    Laplace 分布 + Range coding   │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 3. 频谱形状编码 (PVQ)             │
  │    Pyramid Vector Quantization   │
  │    将归一化频谱视为单位球上的点    │
  │    N 维球面上的最优量化           │
  │    比特在频带间动态分配           │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 4. 时频决策 (tf_select)           │
  │    每个频带独立选择:              │
  │    - 时间分辨率优先 (瞬态信号)    │
  │    - 频率分辨率优先 (稳态信号)    │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 5. Range Encoder                  │
  │    band_energy + PVQ index        │
  │    → 紧凑比特流                   │
  └──────────────────────────────────┘

CELT 关键特性:
  - 无 LP 模型 → 对非语音信号无偏见
  - PVQ: 码率越高, 球面上分配点越密 → 音质线性提升
  - 最低 2.5ms 帧 → 算法延迟仅 2.5ms
```

### 2.4 Hybrid 模式

```
Opus Hybrid = SILK (低频 ≤ 8kHz) + CELT (高频 > 8kHz):

  适用: 16-64 kbps, SWB/FB
  
  编码:
    PCM 48kHz
      ├── 下采样到 16kHz → SILK 编码 (0-8kHz 语音结构)
      └── 全频带 → CELT 编码 (8-20kHz 高频细节)
    
  解码:
    SILK 解码 → 16kHz LP 合成 → 上采样到 48kHz
    CELT 解码 → 高频 MDCT 合成
    两者叠加 → 全频带 48kHz 输出

  比特分配 (32kbps Hybrid SWB):
    SILK: ~20 kbps (语音基础)
    CELT: ~12 kbps (高频补充)
```

---

## 3. LC3 帧结构

### 3.1 LC3 编码框架 (ETSI TS 103 634)

```
LC3 (Low Complexity Communication Codec):
  - 蓝牙 LE Audio 强制编解码器
  - 帧长: 7.5ms / 10ms
  - 采样率: 8/16/24/32/44.1/48 kHz
  - 码率: 16-320 kbps

LC3 编码器:

  输入: PCM (N samples per frame)
    │     N = Fs × frame_duration
    │     48kHz × 10ms = 480 samples
    │     48kHz × 7.5ms = 360 samples
    ▼
  ┌──────────────────────────────────┐
  │ 1. MDCT 变换                      │
  │    窗长 = 2N (50% overlap)        │
  │    → N 个频域系数                 │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 2. 频谱整形 (Spectral Shaping)    │
  │    a) 带宽检测 (Bandwidth Det.)  │
  │    b) 时域攻击检测 (Attack Det.) │
  │    c) SNS (Spectral Noise Shaping)│
  │       - 16 个 scale factors      │
  │       - LPC-like 包络建模         │
  │       - 矢量量化 (Split VQ)       │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 3. 时域噪声整形 (TNS)             │
  │    - 预测滤波, 减少时域包络振荡   │
  │    - 最多 2 阶 LPC               │
  │    - 类似 AAC 的 TNS 工具        │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 4. 频谱量化 (Spectral Quantizer)  │
  │    - 全局增益 (global_gain)       │
  │    - 死区量化 (dead-zone)         │
  │    - 算术编码 (Arithmetic Coding) │
  │    - 残余比特 → LSB 补充          │
  └────────────────┬─────────────────┘
                   │
                   ▼
  ┌──────────────────────────────────┐
  │ 5. 比特流组装                     │
  │    side_info + SNS + TNS          │
  │    + spectral_data + noise_fill  │
  │    → 固定长度帧 (CBR)             │
  └──────────────────────────────────┘
```

### 3.2 LC3 帧比特流格式

```
LC3 Frame (固定长度, 由码率决定):

  frame_bytes = bitrate × frame_duration / 8
  例: 160kbps × 10ms = 200 bytes/frame

帧内结构 (比特流从 MSB 到 LSB):
┌─────────────────────────────────────────────────────────┐
│ bandwidth (4 bits)     带宽标志 (NB/WB/SSWB/SWB/FB)     │
│ lastnz (log2)         最后一个非零系数位置              │
│ lsb_mode (1 bit)      LSB 模式标志                      │
├─────────────────────────────────────────────────────────┤
│ SNS data              频谱噪声整形参数                   │
│   - ind_LF (32 bits)   低频 VQ index                    │
│   - ind_HF (32 bits)   高频 VQ index                    │
│   - shape_j            子带增益形状                      │
├─────────────────────────────────────────────────────────┤
│ TNS data              时域噪声整形 (可选)                │
│   - order (0-2)                                         │
│   - coef_idx[]         滤波器系数索引                    │
├─────────────────────────────────────────────────────────┤
│ Spectral data         频谱系数 (Arithmetic coded)        │
│   - 从高频到低频逐个编码                                 │
│   - 使用概率模型 + range coder                           │
├─────────────────────────────────────────────────────────┤
│ Residual / Noise Fill 残余比特 + 噪声填充                │
│   - 未被量化的频带用伪随机噪声填充                       │
│   - LSB bits: 利用剩余比特精修重要系数                   │
└─────────────────────────────────────────────────────────┘
```

### 3.3 LC3 vs SBC vs AAC-LC 帧级对比

```
同等 48kHz Stereo 160kbps:

┌──────────────┬──────────┬──────────┬──────────┐
│              │ LC3      │ SBC      │ AAC-LC   │
├──────────────┼──────────┼──────────┼──────────┤
│ 帧长         │ 10ms     │ ~12ms    │ 21.3ms   │
│ 帧大小       │ 200 B    │ ~246 B   │ ~426 B   │
│ 算法延迟     │ 10ms     │ ~12ms    │ 21.3ms   │
│ 变换         │ MDCT     │ 子带滤波 │ MDCT     │
│ 频域粒度     │ 480 bins │ 8 subbands│ 1024 bins│
│ 熵编码       │ 算术编码 │ 无       │ Huffman  │
│ 量化         │ Dead-zone│ Bit alloc│ NL 量化  │
│ 音质 (MUSHRA)│ ~85      │ ~52      │ ~72      │
│ 复杂度(MIPS) │ ~15      │ ~5       │ ~25      │
└──────────────┴──────────┴──────────┴──────────┘

结论: LC3 在低复杂度下实现了接近 AAC 的音质,
      且帧长短一半 → 延迟低一半, 非常适合实时蓝牙传输
```

### 3.4 LC3plus 扩展

```
LC3plus (Fraunhofer 扩展):
  - 帧长: 2.5ms / 5ms / 10ms
  - 新增: 高分辨率模式 (96kHz/24bit)
  - 新增: 多通道扩展
  - 2.5ms 帧: 端到端延迟 < 5ms (适合无线监听耳返)
  
  与 LC3 的关系:
    LC3:     蓝牙 LE Audio 标准 (免费)
    LC3plus: 超集, 更多帧长和采样率 (需授权)
    LC3plus 向后兼容 LC3 10ms 帧
```

---

## 4. RTP 音频打包

### 4.1 RTP Header 结构 (RFC 3550)

```
RTP Packet:
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       Sequence Number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             SSRC                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Payload ...                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

关键字段:
  V (2):  版本 = 2
  P (1):  填充位
  X (1):  扩展头标志
  CC (4): CSRC 计数
  M (1):  标记位 (通常帧边界/静音后第一帧)
  PT (7): Payload Type (动态分配, 如 Opus=111)
  Seq:    序列号 (检测丢包)
  TS:     时间戳 (音频采样时钟)
  SSRC:   同步源标识符
```

### 4.2 Opus RTP 打包 (RFC 7587)

```
Opus RTP Payload:
  PT = 动态 (通常 111)
  Clock Rate = 48000 (固定, 不管实际采样率)
  Channels: 通过 SDP 'sprop-maxcapturerate' 声明

  简单模式 (1 frame per packet):
  ┌──────────────┬─────────────────────────────┐
  │ RTP Header   │ [TOC byte][Opus Frame Data] │
  │ (12 bytes)   │                             │
  └──────────────┴─────────────────────────────┘
  
  Timestamp 增量:
    20ms 帧: TS += 960 (48000 × 0.020)
    10ms 帧: TS += 480

  典型 SDP:
    m=audio 49170 RTP/AVP 111
    a=rtpmap:111 opus/48000/2
    a=fmtp:111 minptime=20; useinbandfec=1; stereo=1
```

### 4.3 AAC RTP 打包 (RFC 3640)

```
AAC RTP Payload (mpeg4-generic mode):
  
  ┌──────────────┬──────────────────────────────────────┐
  │ RTP Header   │ AU Header Section │ AU Data Section  │
  │              │ (可选)            │ (AAC frames)     │
  └──────────────┴──────────────────────────────────────┘
  
  AU Header Section:
    AU-headers-length (16 bits): AU header 区域总 bit 数
    AU-header[0]: AU-size (13 bits) + AU-Index (3 bits)
    AU-header[1]: AU-size (13 bits) + AU-Index-delta (3 bits)
    ...
  
  允许多个 AAC Access Unit 打入一个 RTP 包:
    → 减少网络开销 (适合低码率场景)
    → 增加延迟 (需凑齐多帧才发)
    
  典型 SDP:
    m=audio 49230 RTP/AVP 96
    a=rtpmap:96 mpeg4-generic/44100/2
    a=fmtp:96 streamtype=5; profile-level-id=2;
             mode=AAC-hbr; sizeLength=13; indexLength=3;
             config=1210   (AudioSpecificConfig hex)
```

---

## 5. 实战：帧解析与调试

### 5.1 使用 ffprobe 分析帧信息

```bash
# AAC 帧信息
ffprobe -show_frames -select_streams a -print_format json input.aac 2>/dev/null | head -60

# 输出示例:
# "codec_type": "audio",
# "pkt_size": "423",            ← 帧大小 (bytes)
# "nb_samples": "1024",         ← 每帧样本数
# "pkt_duration_time": "0.021333" ← 21.3ms

# Opus 帧信息 (OGG 封装)
ffprobe -show_frames -select_streams a input.opus 2>/dev/null | grep -E "pkt_size|nb_samples|duration"

# 获取 AAC AudioSpecificConfig
ffprobe -show_streams -select_streams a input.m4a 2>/dev/null | grep extradata
```

### 5.2 手动解析 ADTS 帧头 (Python)

```python
import struct

def parse_adts_header(data):
    """解析 ADTS 帧头"""
    if len(data) < 7:
        return None
    
    # Byte 0-1: sync word
    sync = (data[0] << 4) | (data[1] >> 4)
    if sync != 0xFFF:
        return None
    
    header = {}
    header['id'] = (data[1] >> 3) & 0x01           # MPEG-2 or MPEG-4
    header['protection_absent'] = data[1] & 0x01
    header['profile'] = (data[2] >> 6) & 0x03       # 0=Main, 1=LC, 2=SSR
    
    sf_index = (data[2] >> 2) & 0x0F
    sf_table = [96000,88200,64000,48000,44100,32000,24000,22050,
                16000,12000,11025,8000,7350]
    header['sample_rate'] = sf_table[sf_index] if sf_index < 13 else 0
    
    header['channel_config'] = ((data[2] & 0x01) << 2) | ((data[3] >> 6) & 0x03)
    header['frame_length'] = ((data[3] & 0x03) << 11) | (data[4] << 3) | ((data[5] >> 5) & 0x07)
    header['buffer_fullness'] = ((data[5] & 0x1F) << 6) | ((data[6] >> 2) & 0x3F)
    header['num_raw_blocks'] = data[6] & 0x03
    
    return header

# 使用
with open('audio.aac', 'rb') as f:
    data = f.read(9)
    info = parse_adts_header(data)
    print(f"Profile: AAC-{['Main','LC','SSR','LTP'][info['profile']]}")
    print(f"Sample Rate: {info['sample_rate']} Hz")
    print(f"Channels: {info['channel_config']}")
    print(f"Frame Length: {info['frame_length']} bytes")
```

### 5.3 Opus TOC 解析

```python
def parse_opus_toc(toc_byte):
    """解析 Opus TOC byte"""
    config = (toc_byte >> 3) & 0x1F
    stereo = (toc_byte >> 2) & 0x01
    frame_code = toc_byte & 0x03
    
    # 确定模式和帧长
    if config <= 11:
        mode = 'SILK'
        frame_ms = [10, 20, 40, 60][config % 4]
        if config <= 3: bandwidth = 'NB (8kHz)'
        elif config <= 7: bandwidth = 'MB (12kHz)'
        else: bandwidth = 'WB (16kHz)'
    elif config <= 15:
        mode = 'Hybrid'
        frame_ms = [10, 20][(config - 12) % 2]
        bandwidth = 'SWB (24kHz)' if config <= 13 else 'FB (48kHz)'
    else:
        mode = 'CELT'
        frame_ms = [2.5, 5, 10, 20][(config - 16) % 4]
        if config <= 19: bandwidth = 'NB (8kHz)'
        elif config <= 23: bandwidth = 'WB (16kHz)'
        elif config <= 27: bandwidth = 'SWB (24kHz)'
        else: bandwidth = 'FB (48kHz)'
    
    frames_per_packet = [1, 2, 2, 'arbitrary'][frame_code]
    
    return {
        'mode': mode,
        'bandwidth': bandwidth,
        'frame_ms': frame_ms,
        'stereo': bool(stereo),
        'frames_per_packet': frames_per_packet
    }

# 示例: TOC = 0xFC → config=31, stereo=1, code=0
info = parse_opus_toc(0xFC)
# {'mode': 'CELT', 'bandwidth': 'FB (48kHz)', 'frame_ms': 20, 'stereo': True, ...}
```

### 5.4 蓝牙 LC3 帧抓取 (btsnoop)

```bash
# 1. 开启 btsnoop HCI log
adb shell setprop persist.bluetooth.btsnooplogmode full
adb shell svc bluetooth disable && adb shell svc bluetooth enable

# 2. 播放音乐通过 LE Audio 耳机

# 3. 导出 btsnoop log
adb pull /data/misc/bluetooth/logs/btsnoop_hci.log

# 4. Wireshark 打开, 过滤 LE Audio ISO data:
#    Filter: btle.data_header.llid == 0x02
#    或: btatt.opcode == 0x1b (ATT Handle Value Notification)
#    
#    LE Audio ISO Data Packet:
#    ┌────────────────────────────────────────────┐
#    │ HCI ISO Data Header                        │
#    │ ├── Connection Handle                      │
#    │ ├── Timestamp (optional)                   │
#    │ ├── Sequence Number                        │
#    │ └── ISO Data Load Length                   │
#    ├────────────────────────────────────────────┤
#    │ SDU (= LC3 Frame)                          │
#    │ frame_bytes = bitrate × frame_duration / 8 │
#    └────────────────────────────────────────────┘

# 5. 验证 LC3 帧参数
#    160kbps × 10ms = 200 bytes/frame
#    如果看到 200 字节 SDU → 确认 LC3 @ 160kbps/10ms
```

---

## 6. 关键参考 (References)

1. [IETF RFC 6716 - Opus Codec](https://datatracker.ietf.org/doc/html/rfc6716)
2. [IETF RFC 7587 - Opus RTP](https://datatracker.ietf.org/doc/html/rfc7587)
3. [IETF RFC 3640 - AAC RTP](https://datatracker.ietf.org/doc/html/rfc3640)
4. [ISO 14496-3 - AAC Specification](https://www.iso.org/standard/53943.html)
5. [ETSI TS 103 634 - LC3 Specification](https://www.etsi.org/deliver/etsi_ts/103600_103699/103634/)
6. [Bluetooth Core Spec 5.3 - LE Audio](https://www.bluetooth.com/specifications/specs/core-specification-5-3/)
7. [Opus Codec Implementation (libopus)](https://opus-codec.org/docs/)
8. [Fraunhofer LC3plus](https://www.iis.fraunhofer.de/en/ff/amm/communication/lc3.html)
