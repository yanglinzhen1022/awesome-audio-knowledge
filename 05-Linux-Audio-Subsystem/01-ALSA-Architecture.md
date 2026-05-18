# ALSA 核心架构 (Advanced Linux Sound Architecture)

ALSA 是 Linux 内核中音频处理的核心子系统。本章深入探讨其内存管理与底层同步机制。

---

## 1. 核心环形缓冲区机制 (Ring Buffer)

ALSA 播放和录音的核心是维护在内核中的环形缓冲区。数据的平衡由两个指针控制：

*   **sw_ptr (软件指针)**：应用层调用 `write()` 写入数据后，该指针向前移动。
*   **hw_ptr (硬件指针)**：DMA 硬件搬运完数据发送到 Codec 后，由中断触发，该指针向前移动。

### 🚀 专家点：Underrun 与 Overrun
1.  **Xrun (Underrun)**：播放时，hw_ptr 追上了 sw_ptr。说明应用层送数据太慢，硬件没得播，会产生“咔哒”爆音。
2.  **Overrun**：录音时，sw_ptr 没赶上 hw_ptr。说明应用层读太慢，内核缓冲区被硬件塞满，导致数据丢失。

---

## 2. 状态机：PCM State

每一个 ALSA 句柄都有一个严格的状态机转换：
*   **OPEN** -> **SETUP** (设置参数) -> **PREPARED** (准备就绪) -> **RUNNING** (正在播放) -> **XRUN** (错误)。

---

## 3. 命令行实战：tinyalsa vs alsa-utils

在嵌入式开发中，我们常使用 tinyalsa 进行底层验证：

```bash
# 播放测试
tinyplay /sdcard/test.wav -D 0 -d 0 -p 1024 -n 4

# 录音测试
tinycap /sdcard/rec.wav -D 0 -d 0 -r 44100 -b 16 -c 2

# 查看 Mixer 控制项 (Kcontrol)
tinymix -D 0
```
*参数解释：-p 代表 Period Size，-n 代表 Period Count。缓冲区总大小 = p * n。*

---

## 4. 关键数据结构 (Kernel 源码)

```c
// include/sound/pcm.h 简化版
struct snd_pcm_substream {
    struct snd_pcm *pcm;
    struct snd_pcm_runtime *runtime; // 核心运行时数据，包含缓冲区指针
    int stream; // PLAYBACK 或 CAPTURE
    // ...
};
```

---

## 5. 关键参考 (References)

1.  [ALSA Project - PCM Ring Buffer Detail](https://www.alsa-project.org/alsa-doc/alsa-lib/pcm_ring_buffer.html)
2.  *Linux Sound Subsystem* - Jaroslav Kysela
3.  [Kernel.org: Writing an ALSA Driver](https://www.kernel.org/doc/html/latest/sound/kernel-api/writing-an-alsa-driver.html)
