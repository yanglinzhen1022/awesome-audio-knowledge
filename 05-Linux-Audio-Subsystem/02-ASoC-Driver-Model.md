# ASoC 驱动模型 (ALSA System on Chip)

ASoC 层的目标是将音频驱动标准化。本章解析 ASoC 如何通过声明式的方式管理复杂的 SoC 音频系统。

---

## 1. DAPM：动态音频电源管理

DAPM 是 ASoC 最精妙的设计。它将声卡中的每个电路块抽象为 **Widget**。

### 1.1 Widget 类型
*   **snd_soc_dapm_adc/dac**：数模转换器。
*   **snd_soc_dapm_mixer**：多个输入混合。
*   **snd_soc_dapm_pga**：可编程增益放大器。
*   **snd_soc_dapm_output/input**：物理引脚（Mic/Speaker）。

### 1.2 自动寻路逻辑
DAPM 内部维护了一张**有向图**。
1.  当应用层打开 PCM 流时，DAPM 会从 `SND_SOC_DAPM_DAC` 开始，寻找一条通往 `SND_SOC_DAPM_OUTPUT` 的路径。
2.  **只有路径上被选中的 Widget 才会上电**，其余所有电路保持关断。
3.  这实现了“零代码参与”的深度节能。

---

## 2. DAI Link：数据链路的定义

DAI Link 是 Machine 驱动的核心，它定义了数据从哪里流向哪里。

```c
// Machine 驱动示例代码 (C)
static struct snd_soc_dai_link my_machine_dai[] = {
    {
        .name = "HiFi-Audio",
        .stream_name = "HiFi-Playback",
        .cpu_dai_name = "soc-i2s.0", // SoC 侧 I2S
        .codec_dai_name = "rt5640-hifi", // Codec 侧 I2S
        .platform_name = "soc-audio.0", // DMA 驱动
        .codec_name = "rt5640.0-001c", // Codec I2C 地址
        .dai_fmt = SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_NB_NF | SND_SOC_DAIFMT_CBM_CFM,
    },
};
```

---

## 3. ASoC 调试神器：DebugFS

ASoC 将其所有状态暴露在 Linux 的 DebugFS 中：
*   **查看 DAPM 状态**：`cat /sys/kernel/debug/asoc/<card>/dapm/bias_level`。
*   **查看 Widget 电源**：进入 `widgets` 目录查看 `[on]` 或 `[off]` 状态。
*   **查看通路 (Routes)**：通过 `cat paths` 查看当前的音频路由图。

---

## 4. 关键参考 (References)

1.  [Linux Kernel: ASoC Design Overview](https://www.kernel.org/doc/html/latest/sound/soc/overview.html)
2.  [DAPM internals - Wolfson Microelectronics](https://www.alsa-project.org/wiki/DAPM)
3.  *Audio Driver Development in Linux* - Specialized Tutorials
