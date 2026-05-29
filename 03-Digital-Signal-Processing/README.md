# 03. 数字信号处理与算法 (Digital Signal Processing & Algorithms)

本模块介绍音频领域的核心算法，涵盖从基础的语音通信处理到高级的 AI 语音交互。

## 📖 章节导航

1.  **[语音通信 3A 算法 (3A Algorithms)](./01-3A-Algorithms.md)**
    *   回声消除 (AEC)：自适应滤波器原理与信号流向。
    *   自动降噪 (ANS)：谱减法、维纳滤波与 AI 降噪。
    *   自动增益控制 (AGC)：能量检测与动态增益映射。

2.  **[音效处理 (Audio Effects)](./02-Audio-Effects.md)**
    *   均衡器 (EQ)：PEQ 参数调节与 GEQ 原理。
    *   动态范围控制 (DRC)：压缩器、限制器的核心参数。
    *   空间音频 (Spatial Audio)：HRTF 虚拟环绕声与混响模拟。

3.  **[语音交互算法 (Voice Interaction)](./03-Voice-Interaction.md)**
    *   语音交互全链路：KWS -> ASR -> NLU -> TTS。
    *   语音活动检测 (VAD) 的意义。
    *   车载多音区识别与离线化要求。

4.  **[空间音频 (Spatial Audio)](./04-Spatial-Audio.md)**
    *   人类空间听觉原理：ITD、ILD 与耳廓效应。
    *   HRTF 渲染与个性化方案。
    *   Ambisonics 球谐函数声场编码。
    *   商业方案：Dolby Atmos、Apple Spatial Audio、Sony 360RA。
    *   头部追踪与车载空间音频。

5.  **[音频编解码格式 (Audio Codec Formats)](./05-Audio-Codec-Formats.md)**
    *   通用格式：AAC、Opus、FLAC、MP3。
    *   蓝牙格式：SBC、aptX、LDAC、LC3 (LE Audio)。
    *   语音通信：AMR-WB、EVS、Opus。
    *   沉浸式：Dolby AC-4、MPEG-H、IAMF。

7.  **[音频编解码帧结构深度解析 (Codec Frame Deep Dive)](./07-Codec-Frame-Deep-Dive.md)**
    *   AAC：ADTS 帧头逐字段解析、MDCT 编码工具链、Raw Frame 内部结构、HE-AAC SBR 扩展。
    *   Opus：TOC Byte 解析、SILK 语音核心 (LP/LTP/Range coding)、CELT 音乐核心 (MDCT/PVQ)、Hybrid 模式。
    *   LC3：SNS 频谱整形、Dead-zone 量化、帧比特流格式、LC3 vs SBC vs AAC 帧级对比。
    *   RTP 音频打包：RFC 7587 (Opus) / RFC 3640 (AAC) 打包格式与 SDP 示例。
    *   实战：Python ADTS/Opus 帧解析脚本、ffprobe 分析、btsnoop LC3 抓取。

6.  **[AI 音频处理 (AI-Powered Audio)](./06-AI-Audio.md)**
    *   AI 降噪：RNNoise / DTLN / DCCRN 等模型架构与对比。
    *   AI 回声消除：NN-AEC 原理与混合 Pipeline。
    *   端侧部署：TFLite / SNPE / QNN、量化、模型压缩。
    *   语音识别 ASR：Conformer-Transducer、Whisper、端侧 KWD。
    *   语音合成 TTS：FastSpeech2 + HiFi-GAN 端侧方案。
    *   神经音频编解码：EnCodec / SoundStream / RVQ。

---
[返回主目录](../README.md)
