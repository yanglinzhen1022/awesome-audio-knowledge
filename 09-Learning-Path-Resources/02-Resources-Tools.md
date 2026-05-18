# 音频开发资源与工具推荐 (Resources & Tools)

好的工具和参考资料可以事半功倍。以下是音频行业最常用的经典资源。

---

## 1. 经典书籍 (Books)

### 1.1 基础理论类
*   **《The Master Handbook of Acoustics》** - F. Alton Everest：声学界的圣经，涵盖了从声学基础到录音室设计的一切知识。
*   **《Principles of Digital Audio》** - Ken C. Pohlmann：深入浅出地讲解了采样、量化、CD、压缩等数字音频基础。

### 1.2 信号处理与算法类
*   **《Digital Audio Signal Processing》** - Udo Zölzer：DSP 算法开发的必备参考书，包含 EQ、混响、动态处理的数学模型。
*   **《Adaptive Filter Theory》** - Simon Haykin：研究 AEC（回声消除）算法的深度进阶教材。

### 1.3 工程实践类
*   **《Embedded Android》** - Karim Yaghmour：虽然是讲 Android 系统，但其中关于 HAL 和硬件驱动的部分对音频开发非常有参考价值。

---

## 2. 优秀开源项目 (Open Source Projects)

*   **[WebRTC](https://webrtc.org/)**：包含了世界上最成熟的 3A 算法（AEC, ANS, AGC）实现。
*   **[Oboe (Google)](https://github.com/google/oboe)**：一个 C++ 库，用于在 Android 上构建高性能、低延迟的音频应用。
*   **[FFmpeg](https://ffmpeg.org/)**：音频/视频编解码、协议处理的百科全书。
*   **[Sox (Sound eXchange)](http://sox.sourceforge.net/)**：命令行音频处理的神器，被称为“音频界的瑞士军刀”。

---

## 3. 常用软件工具 (Software Tools)

### 3.1 分析与编辑
*   **Audacity**：开源、免费的跨平台音频编辑器，查看频谱图（Spectrogram）非常方便。
*   **Adobe Audition**：专业级音频处理软件，具有强大的降噪和波形分析功能。

### 3.2 测量与调音
*   **REW (Room EQ Wizard)**：完全免费的房间声学分析软件，广泛用于音箱测量和 EQ 调试。
*   **Audio Precision APx500**：配合 AP 硬件使用的控制软件（收费，行业标准）。

### 3.3 录音与 Dump 工具 (Android)
*   `adb shell tinycap`：Android 下最原始的录音 Dump 工具。
*   `adb shell dumpsys media.audio_flinger`：查看 Android 系统当前音频流状态的神级命令。

---

## 4. 在线社区与博客

*   **[Hydrogenaudio](https://www.hydrogenaudio.org/)**：极客级别的音频技术讨论社区。
*   **[KVR Audio](https://www.kvraudio.com/)**：专注于音效插件和音乐制作技术。
*   **[AOSP Audio Documentation](https://source.android.com/devices/audio)**：Android 音频开发的官方最权威文档。

---
[返回主目录](../README.md)
