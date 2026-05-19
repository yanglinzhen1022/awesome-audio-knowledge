# 数字音频接口与总线 (Digital Audio Interfaces & Bus)

在音频 SoC 设计中，芯片间的数据交换（Inter-IC Communication）主要依赖以下几种串行总线。本章涵盖 I2S、TDM、PDM、SoundWire 四种核心接口。

---

## 接口选型速查

| 场景 | 推荐接口 | 说明 |
|:---|:---|:---|
| SoC ↔ Codec (2ch) | I2S | 最简单、最通用 |
| SoC ↔ 外部 DSP (多ch) | TDM | 单线多通道 |
| SoC ↔ DMIC | PDM | 2 线支持双麦 |
| SoC ↔ Codec + SmartPA | SoundWire | 数据+控制+电源管理一线搞定 |
| 车载 SoC ↔ 远端节点 | A2B | 长距离、菊花链 (见 [车载硬件](./04-Automotive-Hardware.md)) |

---

# Part 1: I2S 总线 (Inter-IC Sound)

I2S 是飞利浦 (Philips) 于 1986 年提出的芯片间数字音频传输标准，至今仍是音频硬件最基础、最广泛使用的接口。

## 1.1 I2S 信号线定义

```
I2S 标准使用 3 根信号线:

  ┌─────────────────────────────────────────────────────────┐
  │ 信号线              说明                                │
  ├─────────────────────────────────────────────────────────┤
  │ BCLK (Bit Clock)    位时钟, 每个边沿传输 1 bit         │
  │ LRCLK / WS          帧同步 (左/右声道选择)             │
  │   (Word Select)     0 = Left, 1 = Right (标准 I2S)     │
  │ SD (Serial Data)    串行数据线, MSB first              │
  └─────────────────────────────────────────────────────────┘
  
  可选信号:
    MCLK (Master Clock): 主时钟, 通常 = 256 × Fs 或 512 × Fs
      → 部分 Codec 需要 MCLK 才能锁相产生内部时钟
      → 现代 Codec (如 WCD938x) 可用 PLL 自恢复, 不需要 MCLK
      
  时钟关系:
    BCLK = Fs × Channels × Bit_Depth
    例: 48kHz × 2ch × 32bit = 3.072 MHz
    
    MCLK = N × Fs (N = 256/384/512)
    例: 256 × 48000 = 12.288 MHz
```

## 1.2 四种数据对齐格式

```
以 16-bit 采样、Stereo 为例:

  1. I2S Standard (Philips):
     MSB 在 WS 翻转后延迟 1 个 BCLK
     
     WS:   _____|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|_______________|‾‾‾‾
     BCLK: _|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_
     DATA: ----[   Left Channel   ][  Right Channel  ]
               ↑ MSB 在此 (WS翻转后第2个BCLK)
     
  2. Left Justified (左对齐):
     MSB 与 WS 翻转同时开始
     
     WS:   _____|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|_______________|‾‾‾‾
     DATA: -----[MSB         LSB]-[MSB         LSB]---
                ↑ MSB 在此 (WS翻转后第1个BCLK)
     
  3. Right Justified (右对齐 / EIAJ):
     LSB 与 WS 翻转对齐 (数据右对齐)
     → 常见于旧式日系 DAC
     
  4. DSP/PCM Mode (TDM Short Frame):
     WS 变成单 BCLK 宽度的脉冲
     → 用于连接多声道, 与 TDM 兼容
     
     WS:   _|‾|________________________________|‾|____
     DATA: ---[Ch0][Ch1][Ch2]...[ChN]---[Ch0][Ch1]...
```

### Linux ASoC 中的格式配置

```c
// 在 Machine Driver 的 DAI Link 中指定格式:
static struct snd_soc_dai_link my_dai_link = {
    .name = "HiFi",
    .dai_fmt = SND_SOC_DAIFMT_I2S        // 格式: I2S Standard
             | SND_SOC_DAIFMT_NB_NF      // 时钟极性: Normal BCLK, Normal Frame
             | SND_SOC_DAIFMT_CBP_CFP,   // 主从: Codec 提供 BCLK 和 Frame
    // ...
};

// 格式选项:
//   SND_SOC_DAIFMT_I2S          I2S Standard
//   SND_SOC_DAIFMT_LEFT_J       Left Justified
//   SND_SOC_DAIFMT_RIGHT_J      Right Justified
//   SND_SOC_DAIFMT_DSP_A        DSP Mode A (数据在脉冲后1 BCLK)
//   SND_SOC_DAIFMT_DSP_B        DSP Mode B (数据在脉冲后立即开始)

// 主从模式:
//   SND_SOC_DAIFMT_CBP_CFP      Codec is BCLK Provider & Frame Provider (旧: CBS_CFS)
//   SND_SOC_DAIFMT_CBC_CFC      Codec is BCLK Consumer & Frame Consumer (旧: CBM_CFM)
```

## 1.3 主从模式 (Master / Slave)

```
谁提供 BCLK 和 LRCLK, 谁就是 Master:

  模式 1: SoC = Master, Codec = Slave (最常见)
    SoC ──BCLK──→ Codec
    SoC ──LRCLK──→ Codec
    SoC ←──SD──── Codec (录音)
    SoC ──SD──→ Codec (播放)
    
  模式 2: Codec = Master, SoC = Slave
    适用于 Codec 有高精度时钟 (如 Hi-Fi DAC)
    
  模式 3: 外部时钟发生器 (如 Si5351) 同时驱动两者
    适用于需要严格同步的多设备系统
    
  注意事项:
    - 系统中 BCLK 只能有一个 Master!
    - 如果两端都配成 Master → 时钟冲突 → 无声或杂音
    - 常见 Bug: DTS 中 SoC 和 Codec 都配了 Master
```

## 1.4 常见 I2S 问题与调试

```bash
# === 检查 I2S 时钟是否正确 ===
# 用示波器/逻辑分析仪测量:
#   BCLK 频率是否 = Fs × Channels × BitWidth
#   LRCLK 频率是否 = Fs
#   MCLK 频率是否 = 256 × Fs (如需要)

# === 常见问题 ===
# 1. 无声:
#   → BCLK/LRCLK 无输出 → 检查 SoC I2S 时钟配置
#   → MCLK 缺失 → Codec 无法锁定 PLL
#   → 主从配置冲突

# 2. 声音变调 (音高不对):
#   → 采样率不匹配 (SoC 48kHz vs Codec 44.1kHz)
#   → MCLK/BCLK 比例错误

# 3. 左右声道反转:
#   → WS 极性配置反了 (NB_NF vs NB_IF)

# 4. 声音断续/杂音:
#   → BCLK Jitter 过大 → 检查时钟源质量
#   → DMA 配置错误 → period_size 不匹配
```

---

# Part 2: TDM 时分复用 (Time Division Multiplexing)

当需要在一条物理 I2S 数据线上传输 2 个以上声道时（如车载 8ch、专业音频 16ch），TDM 是标准方案。TDM 本质上是 I2S DSP/PCM Mode 的多通道扩展。

## 2.1 TDM 基本原理

```
TDM 将每个采样周期 (1/Fs) 划分为多个时间槽 (Slot):

  一帧 = N 个 Slot, 每个 Slot 传输一个声道的数据
  
  Slot 0    Slot 1    Slot 2    Slot 3    Slot 4 ...
  ┌────────┬────────┬────────┬────────┬────────┐
  │  Ch 0  │  Ch 1  │  Ch 2  │  Ch 3  │  Ch 4  │ ...
  └────────┴────────┴────────┴────────┴────────┘
  ←──────────── 1 Frame (1/Fs) ────────────────→
  
  帧同步信号 (FSYNC):
    长帧模式 (Long Frame): FSYNC = Slot 0 持续时间 (类似 LRCLK)
    短帧模式 (Short Frame): FSYNC = 1 BCLK 宽度的脉冲 → 更常用
```

## 2.2 关键参数计算

```
TDM Bitclock 计算:

  BCLK = Sample_Rate × Num_Slots × Slot_Width
  
  示例:
  ┌──────────────┬────────┬────────────┬───────────────┐
  │ 场景         │ Fs     │ Slots×Width│ BCLK          │
  ├──────────────┼────────┼────────────┼───────────────┤
  │ Stereo I2S   │ 48kHz  │ 2 × 32bit │ 3.072 MHz     │
  │ 车载 4ch     │ 48kHz  │ 4 × 32bit │ 6.144 MHz     │
  │ 车载 8ch     │ 48kHz  │ 8 × 32bit │ 12.288 MHz    │
  │ 车载 16ch    │ 48kHz  │ 16 × 32bit│ 24.576 MHz    │
  │ 高保真 8ch   │ 96kHz  │ 8 × 32bit │ 24.576 MHz    │
  │ 专业音频 32ch│ 48kHz  │ 32 × 32bit│ 49.152 MHz    │
  └──────────────┴────────┴────────────┴───────────────┘
  
  Slot Width vs Sample Width:
    Slot Width = 32-bit (固定槽宽, 含 padding)
    Sample Width = 16/24-bit (实际有效数据)
    → 16-bit 数据放在 32-bit Slot 中, 高位对齐, 低位补 0
```

## 2.3 TDM Slot Mapping (通道映射)

```
Slot Mapping = 哪个声道放在哪个 Slot 位置:

  高通平台 TDM 配置示例:
    TDM RX (播放):
      Slot 0 → Front Left
      Slot 1 → Front Right  
      Slot 2 → Center
      Slot 3 → LFE (低音炮)
      Slot 4 → Rear Left
      Slot 5 → Rear Right
      Slot 6 → Side Left
      Slot 7 → Side Right
      
    TDM TX (录音):
      Slot 0 → Mic 1
      Slot 1 → Mic 2
      Slot 2 → Reference (AEC 参考)
      Slot 3 → IV-Sense Left
      
  Slot Mask 表示法 (bitmask):
    使用哪些 Slot: slot_mask = 0xFF (8 个 Slot 全用)
    只用 Slot 0,1: slot_mask = 0x03
    只用 Slot 0,2: slot_mask = 0x05
```

### 高通平台 TDM 配置

```c
// 高通 Machine Driver 中的 TDM 配置:
// techpack/audio/asoc/msm-dai-q6-v2.c

// 设置 TDM 参数
static int msm_tdm_snd_hw_params(struct snd_pcm_substream *substream,
                                  struct snd_pcm_hw_params *params) {
    struct snd_soc_pcm_runtime *rtd = substream->private_data;
    struct snd_soc_dai *cpu_dai = asoc_rtd_to_cpu(rtd, 0);
    
    int slot_width = 32;
    int num_slots = 8;
    unsigned int slot_mask = 0xFF;  // 使用全部 8 个 slot
    unsigned int slot_offset[8] = {0, 4, 8, 12, 16, 20, 24, 28}; // 字节偏移
    
    ret = snd_soc_dai_set_tdm_slot(cpu_dai, 
            slot_mask,      // TX slot mask
            slot_mask,      // RX slot mask
            num_slots,      // 总 slot 数
            slot_width);    // 每 slot 位宽
    
    return ret;
}
```

### Device Tree TDM 配置

```dts
// 高通平台 DTS 中的 TDM 端口定义:
&q6core {
    pri_tdm: primary-tdm {
        compatible = "qcom,msm-dai-tdm";
        qcom,msm-cpudai-tdm-group-id = <0x3700>;  // Primary TDM RX
        qcom,msm-cpudai-tdm-group-num-ports = <1>;
        qcom,msm-cpudai-tdm-clk-rate = <12288000>; // BCLK = 12.288MHz
        qcom,msm-cpudai-tdm-sync-mode = <0>;       // Short Sync
        qcom,msm-cpudai-tdm-sync-src = <1>;         // Internal
        qcom,msm-cpudai-tdm-data-out = <0>;         // Data Out
        qcom,msm-cpudai-tdm-invert-sync = <0>;
        qcom,msm-cpudai-tdm-data-delay = <1>;       // 1 BCLK delay
        
        dai_pri_tdm_rx_0: qcom,msm-dai-q6-tdm-pri-rx-0 {
            compatible = "qcom,msm-dai-q6-tdm";
            qcom,msm-cpudai-tdm-dev-id = <0x3700>;
            qcom,msm-cpudai-tdm-data-align = <0>; // MSB aligned
        };
    };
};
```

## 2.4 TDM 与 I2S 的关系

```
I2S 是 TDM 的特殊情况 (2 Slot TDM):

  I2S Standard = TDM Long Frame, 2 slots, 1 BCLK delay
  I2S Left-J   = TDM Long Frame, 2 slots, 0 delay
  DSP/PCM Mode = TDM Short Frame, N slots
  
  在 Linux ASoC 中:
    2ch I2S  → snd_soc_dai_set_fmt(SND_SOC_DAIFMT_I2S)
    多ch TDM → snd_soc_dai_set_fmt(SND_SOC_DAIFMT_DSP_B)
              + snd_soc_dai_set_tdm_slot()
```

## 2.5 常见 TDM 问题

| # | 问题 | 原因 | 解决 |
|:---|:---|:---|:---|
| 1 | 只有部分声道有声 | Slot mask 配置不全 | 检查 slot_mask 是否覆盖所有声道 |
| 2 | 声道顺序错乱 | Slot offset 配错 | 核对 slot_offset 数组 |
| 3 | 声音变调/异常 | BCLK 频率不对 | 确认 BCLK = Fs × Slots × SlotWidth |
| 4 | 杂音/底噪 | 数据对齐问题 | 检查 data-delay (0 或 1 BCLK) |
| 5 | 多设备不同步 | 时钟域不一致 | 确保所有设备使用同一 BCLK 源 |

```bash
# TDM 调试命令 (高通平台)
adb shell cat /proc/asound/card0/pcm*p/sub0/hw_params  # 查看 PCM 参数
adb shell tinymix | grep -i tdm    # 查看 TDM 相关 mixer controls
adb shell cat /proc/asound/card0/pcm*p/sub0/status     # 运行状态
```

---

# Part 3: PDM 脉冲密度调制 (Pulse Density Modulation)

PDM 是数字 MEMS 麦克风与 SoC 之间的标准接口。与 I2S 传输多位 PCM 数据不同，PDM 仅用 1 bit 表示音频——靠"1"的密度编码振幅。

## 3.1 PDM 调制原理

```
PDM 信号特征:

  采样率: 极高 (1.024 MHz ~ 4.8 MHz, 典型 3.072 MHz)
  位宽:   1 bit (只有 0 和 1)
  编码:   1 的密度 ∝ 信号振幅
  
  波形示意:
    模拟信号:   ╱‾‾‾‾‾╲            ╱‾╲
               ╱       ╲          ╱   ╲
    ──────────╱         ╲────────╱     ╲───────
    
    PDM 码流: 1110111101111110 0001000010000001 0110111011101110
              ← 高密度(正峰值) → ← 低密度(负峰值) → ← 中等密度 →
              
  vs PCM:
    PCM: 每个采样 = 16/24 bit 的精确数值
    PDM: 每个采样 = 1 bit, 但采样率高 64-256 倍
    → 信息量等效: PDM @3.072MHz ≈ PCM @48kHz/16bit
    
  Sigma-Delta (ΣΔ) 调制器:
    MEMS 麦克风内部使用 ΣΔ ADC 将模拟信号转换为 PDM
    → 量化噪声被推到高频 (Noise Shaping)
    → 低频信号保持高 SNR
```

## 3.2 PDM 信号线

```
PDM 接口只需 2 根线:

  ┌──────────────────────────────────────────────┐
  │ 信号线       说明                            │
  ├──────────────────────────────────────────────┤
  │ PDM_CLK      时钟 (SoC 提供, 1~5 MHz)       │
  │ PDM_DATA     数据 (MEMS 麦克风输出)          │
  └──────────────────────────────────────────────┘
  
  一根 DATA 线可传输 2 个麦克风信号!
    → 麦克风 A: 在 CLK 上升沿输出数据
    → 麦克风 B: 在 CLK 下降沿输出数据
    → 通过 L/R Select 引脚配置 (接 GND 或 VDD)
    
  典型连接:
    SoC PDM_CLK ──→ DMIC0_CLK, DMIC1_CLK
    SoC PDM_DATA0 ←── DMIC0 (上升沿) + DMIC1 (下降沿)
    SoC PDM_DATA1 ←── DMIC2 (上升沿) + DMIC3 (下降沿)
    → 2 根 DATA 线 = 4 个麦克风
```

## 3.3 SoC 侧抽取处理 (Decimation)

```
PDM → PCM 转换链 (在 SoC 的数字音频前端完成):

  PDM 码流 (1-bit @ 3.072 MHz)
    │
    ▼
  ┌──────────────────────────────────┐
  │ CIC 滤波器 (Cascaded            │
  │ Integrator-Comb)                 │
  │   → 抽取 (Decimation) 降低采样率│
  │   → 3.072MHz ÷ 64 = 48kHz       │
  │   → 输出: 多位 PCM              │
  │   → 但频响有 droop (衰减)       │
  └──────────────────────────────────┘
    │
    ▼
  ┌──────────────────────────────────┐
  │ 补偿 FIR 滤波器                 │
  │   → 修正 CIC 的频响 droop       │
  │   → 恢复平坦频响               │
  └──────────────────────────────────┘
    │
    ▼
  ┌──────────────────────────────────┐
  │ HPF (高通滤波器)                 │
  │   → 滤除 DC Offset              │
  │   → 截止频率: ~20 Hz            │
  └──────────────────────────────────┘
    │
    ▼
  PCM 输出 (16/24-bit @ 48kHz)
  → 送入 ADSP / AudioFlinger 处理
  
  WCD938x Codec 中的 PDM 路径:
    DMIC → PDM Interface → Decimation Filter → TX DMA → SoundWire
```

## 3.4 DMIC 配置实战

### Device Tree 配置 (高通平台)

```dts
// DMIC 在 Device Tree 中的定义:
&wcd938x_codec {
    // DMIC 时钟频率配置
    qcom,cdc-dmic-clk-drv-strength = <2>;  // 驱动强度
    
    // DMIC GPIO 配置
    qcom,cdc-dmic01-gpios = <&wcd_dmic01_gpios 0 0>;
    qcom,cdc-dmic23-gpios = <&wcd_dmic23_gpios 0 0>;
};

// DMIC GPIO pinctrl
wcd_dmic01_gpios: wcd-dmic01-gpios {
    compatible = "qcom,msm-cdc-pinctrl";
    pinctrl-names = "aud_active", "aud_sleep";
    pinctrl-0 = <&wcd_dmic01_clk_active &wcd_dmic01_data_active>;
    pinctrl-1 = <&wcd_dmic01_clk_sleep &wcd_dmic01_data_sleep>;
};
```

### DMIC 调试

```bash
# 检查 DMIC 是否被识别
adb shell tinymix | grep -i dmic
# "DMIC0" → 0=Off, 1=DMIC0, 2=DMIC1 ...

# 测试录音
adb shell tinycap /data/local/tmp/test.wav -D 0 -d 6 -c 1 -r 48000 -b 16
# -d 6: PCM device ID (DMIC)

# 常见问题:
# 1. 录音无声 → 检查 PDM_CLK 是否输出 (示波器)
# 2. 录音噪声大 → 检查 DMIC 供电 (1.8V), 检查 PCB 走线
# 3. 左右声道反 → L/R Select 引脚接错
# 4. 底噪偏高 → 增大 CIC 滤波器阶数, 或调整抽取比
```

## 3.5 PDM vs I2S 数字麦克风对比

| 维度 | PDM (DMIC) | I2S 数字麦克风 |
|:---|:---|:---|
| 信号线数 | 2 (CLK + DATA) | 3-4 (BCLK + LRCLK + SD) |
| 每线麦克风数 | 2 个 (上升/下降沿) | 1 个 |
| SoC 处理 | 需要 Decimation 滤波 | 直接 PCM, 无需滤波 |
| 功耗 | 较低 (~0.5mW) | 较高 (~1.5mW) |
| 布线复杂度 | 简单 (2 线) | 较复杂 |
| 音质上限 | 受 Decimation 滤波器影响 | 取决于内部 ADC |
| 主流应用 | 手机/笔电/IoT | 专业设备/车载 |

---

# Part 4: SoundWire 总线

SoundWire 是 MIPI 联盟于 2014 年发布的新一代音频接口标准，旨在取代 I2S/PDM/SLIMbus，实现单总线上的音频数据传输 + 设备控制 + 动态电源管理。高通 WCD93xx 系列 Codec 和 WSA88xx SmartPA 已全面采用 SoundWire。

## 4.1 SoundWire vs 传统接口

```
SoundWire 解决了传统接口的痛点:

  ┌────────────────┬──────────┬──────────┬──────────┬──────────────┐
  │ 维度           │ I2S      │ SLIMbus  │ PDM      │ SoundWire    │
  ├────────────────┼──────────┼──────────┼──────────┼──────────────┤
  │ 信号线数       │ 3-4      │ 2        │ 2        │ 2            │
  │ 数据+控制      │ 仅数据   │ 数据+控制│ 仅数据   │ 数据+控制    │
  │ 多设备挂载     │ ❌       │ ✅       │ ❌       │ ✅           │
  │ 动态带宽分配   │ ❌       │ ✅       │ ❌       │ ✅           │
  │ 电源管理       │ 无       │ 基本     │ 无       │ 完善 (Clock  │
  │                │          │          │          │ Stop/Pause)  │
  │ 最大带宽       │ ~50Mbps  │ ~28Mbps  │ ~5Mbps   │ ~100Mbps     │
  │ 标准化组织     │ Philips  │ MIPI     │ 无       │ MIPI         │
  │ 目前状态       │ 广泛使用 │ 被取代中 │ DMIC 用  │ 高通主推     │
  └────────────────┴──────────┴──────────┴──────────┴──────────────┘
```

## 4.2 SoundWire 物理层

```
SoundWire 仅使用 2 根信号线:

  ┌───────────────────────────────────────────────────┐
  │ 信号线           说明                             │
  ├───────────────────────────────────────────────────┤
  │ SWR_CLK (Clock)  主时钟, Master 提供              │
  │                  频率: 0.5 ~ 12.288 MHz           │
  │ SWR_DATA (Data)  双向数据线 (DDR)                 │
  │                  在 CLK 上升沿和下降沿都传数据     │
  └───────────────────────────────────────────────────┘
  
  拓扑结构:
    一条 SoundWire Bus 上可挂载:
      1 个 Master (SoC / Codec)
      最多 11 个 Slave 设备
      
    高通手机典型拓扑:
      SoC LPAIF ── SWR Master 0 ── WCD9385 (Codec, Slave)
                 ── SWR Master 1 ── WSA8845 #1 (SmartPA, Slave)
                                  ── WSA8845 #2 (SmartPA, Slave)
                                  
  帧结构 (Frame):
    每帧 = Control Word + Data Slots
    Control Word: 用于寄存器读写、设备枚举、中断
    Data Slots: 用于 PCM 音频数据传输
    → 控制和数据共用同一总线, 无需额外 I2C/SPI
```

## 4.3 SoundWire 协议层

### 设备枚举

```
SoundWire 设备自动枚举流程:

  1. Master 发送 PING 命令
     → 检测总线上有哪些 Slave 响应
     
  2. Slave 响应自己的 Device ID
     → Device ID = 48-bit 唯一标识
     → 包含: Manufacturer ID + Part ID + Version
     
  3. Master 分配 Device Number (1-11)
     → 后续通信用 Device Number 寻址
     
  4. Master 读取 Slave 的 Data Port 能力
     → 支持多少 Data Port
     → 每个 Port 的采样率/位宽/声道数
     
  5. Master 配置 Data Port
     → 分配 Stream (Playback/Capture)
     → 配置 Transport 参数 (Block Offset, Sample Interval)
     
  vs I2C 设备地址: 
    I2C = 7-bit 固定地址, 容易冲突
    SoundWire = 48-bit 唯一 ID, 自动枚举, 不冲突
```

### 寄存器访问

```
SoundWire 内嵌寄存器读写 (替代 I2C):

  写寄存器:
    Master → Control Word: [DevNum][Write][RegAddr][Data]
    → 在音频传输间隙插入, 不影响音频数据
    
  读寄存器:
    Master → Control Word: [DevNum][Read][RegAddr]
    Slave  → 下一帧返回: [Data]
    
  优势:
    - 无需额外 I2C 总线
    - 寄存器访问与音频传输共享同一物理层
    - 支持批量读写 (Bulk Transfer)
    
  Linux 中的实现:
    SoundWire Slave 注册为 regmap 设备
    → Codec 驱动直接用 snd_soc_component_read/write()
    → 底层通过 SoundWire Control Word 传输
```

## 4.4 SoundWire 电源管理

```
SoundWire Clock Stop 机制:

  ┌────────────────────────────────────────────────────┐
  │ 状态            说明                  功耗         │
  ├────────────────────────────────────────────────────┤
  │ Active          正常传输              全速         │
  │ Clock Stop 0    时钟暂停, Slave 保持  极低 (~µW)   │
  │   (De0)         寄存器状态, 快速恢复  恢复 <100µs  │
  │ Clock Stop 1    时钟停止, Slave 可    最低         │
  │   (De1)         进入深度睡眠          恢复 ~1ms    │
  └────────────────────────────────────────────────────┘
  
  使用场景:
    播放音乐时 → Active
    暂停播放 → Clock Stop 0 (快速恢复, 用户感知不到)
    息屏待机 → Clock Stop 1 (最省电)
    语音唤醒 → Codec 保持低功耗监听, SWR 进入 Clock Stop
    
  DAPM 集成:
    ASoC DAPM Widget 状态变化时自动触发 Clock Stop/Resume
    → 用户无感知的动态功耗管理
```

## 4.5 Linux SoundWire 驱动架构

```
Linux SoundWire 驱动栈:

  用户空间
    ├── ALSA 应用 / AudioFlinger
    └── libasound / TinyALSA
  ────────────────────────────────────────
  内核空间
    ├── ALSA Core / ASoC Framework
    ├── Codec 驱动 (如 wcd938x.c)
    │     → 使用 regmap-soundwire 访问寄存器
    │     → 注册 DAPM Widget / Route
    ├── SoundWire Bus Driver (drivers/soundwire/)
    │     ├── bus.c          → 总线注册/设备枚举
    │     ├── slave.c        → Slave 设备管理
    │     ├── stream.c       → Stream 配置
    │     ├── master.c       → Master 控制器接口
    │     └── sysfs.c        → debugfs 接口
    └── SoundWire Master Controller Driver
          → 高通: qcom-soundwire.c
          → Intel: intel-soundwire.c
  ────────────────────────────────────────
  硬件
    SoC SWR Master ──── SWR Bus ──── WCD9385 / WSA8845
    
  关键源码:
    drivers/soundwire/           ← SoundWire 核心框架
    techpack/audio/asoc/codecs/  ← 高通 Codec SWR 驱动
```

### SoundWire 调试命令

```bash
# 查看 SoundWire 设备
adb shell ls /sys/bus/soundwire/devices/
# sdw:0:0217:0110:00:0   → Master 0, Slave WCD9385
# sdw:1:0217:0204:00:0   → Master 1, Slave WSA8845

# 查看 SoundWire Master 状态
adb shell cat /sys/kernel/debug/soundwire/master-0/status
# State: active / clock_stop

# 查看 Slave 设备信息
adb shell cat /sys/bus/soundwire/devices/sdw:0:0217:0110:00:0/device_id
# Manufacturer: 0x0217 (Qualcomm)
# Part: 0x0110 (WCD9385)

# SoundWire 传输错误统计
adb shell cat /sys/kernel/debug/soundwire/master-0/statistics
# Bus clash count, Parity errors, etc.

# 常见问题:
# 1. Slave 未枚举 → SWR_CLK/SWR_DATA 信号线问题
# 2. 寄存器读写失败 → 总线忙或 Slave 未响应
# 3. Clock Stop 恢复慢 → 检查 Slave 唤醒时间配置
```

---

## 关键参考 (References)

1. *I2S Bus Specification* - Philips Semiconductors, 1986 (Rev 1996)
2. [Understanding Digital Audio Interfaces - Texas Instruments](https://www.ti.com/lit/an/slaa701/slaa701.pdf)
3. [PDM Microphones and Sigma-Delta ADC - Analog Devices](https://www.analog.com/en/technical-articles/pdm-microphones.html)
4. [MIPI SoundWire Specification v1.2](https://www.mipi.org/specifications/soundwire)
5. Linux Kernel: `sound/soc/` — DAI format definitions, `drivers/soundwire/` — SoundWire core
6. [Intel SoundWire Documentation](https://www.kernel.org/doc/html/latest/driver-api/soundwire/)

---
*Next: [移动端音频硬件架构](./03-Mobile-Hardware.md)* | [返回硬件系统](./README.md)
