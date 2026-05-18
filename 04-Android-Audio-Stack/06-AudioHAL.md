# Audio HAL 接口规范 (Audio Hardware Abstraction Layer)

Audio HAL 是 Android “软件世界”与“物理硬件”的交分界线。对于专业音频工程师，理解 HAL 意味着能够将芯片厂商（如高通、MTK）提供的驱动与 Android 系统完美对接。

---

## 1. 从 HIDL 到 AIDL 的架构跨越

从 Android 14 开始，Audio HAL 正在全面从 HIDL 转向 **AIDL**。

*   **HIDL (Legacy)**：基于 `/dev/hwbinder`。接口定义在 `.hal` 文件中。
*   **AIDL (Modern)**：基于标准 Binder。接口定义在 `.aidl` 文件中。

### 🚀 核心接口代码 (AIDL 风格)
```aidl
// IModule.aidl - 代表一个硬件模块
interface IModule {
    // 创建播放流
    StreamDescriptor openOutputStream(in StreamContext context);
    // 创建录音流
    StreamDescriptor openInputStream(in StreamContext context);
    // 硬件参数设置
    void setAudioPortConfig(in AudioPortConfig port);
}
```

---

## 2. HAL 内部实现：对接 Linux ALSA

HAL 的职责是将 Android 的通用请求转换为具体的驱动操作（通常是 **tinyalsa** 库的调用）。

### 🚀 实战：write() 方法的穿透之旅

当 `AudioFlinger` 调用 HAL 的 `write()` 时，HAL 实现类的核心逻辑如下：

```cpp
// 典型的 HAL 实现代码片段 (C++)
ssize_t StreamOut::write(const void* buffer, size_t bytes) {
    // 1. 获取 tinyalsa 的 pcm 句柄
    struct pcm *pcm_handle = mStream->pcm;
    
    // 2. 调用 Linux 系统级 API，将数据推给内核驱动
    // 🚀 专家点：这里通常是阻塞的，直到硬件 DMA 搬运完数据
    int status = pcm_write(pcm_handle, buffer, bytes);
    
    if (status != 0) {
        ALOGE("pcm_write failed: %s", pcm_get_error(pcm_handle));
    }
    return bytes;
}
```

---

## 3. 重要机制：Fast Path (低延迟路径)

为了满足专业音频应用（如电吉他效果器）的需求，HAL 必须支持 **Fast Path**。
*   **标志位**：在 `openOutputStream` 时，Flags 包含 `AUDIO_OUTPUT_FLAG_FAST`。
*   **实现要求**：HAL 不能在 `write()` 中进行任何耗时的锁操作或复杂的算法，必须尽可能快地将数据丢给驱动。

---

## 4. 如何开发并调试一个新的 Audio HAL？

### 4.1 配置文件描述
除了 C++ 代码，你必须在设备树或 vendor 分区提供 `audio_effects.xml` 和 `audio_policy_configuration.xml`。

### 4.2 调试神级命令
*   **查看服务状态**：`adb shell lshal | grep audio` (如果是 HIDL)。
*   **查看驱动节点**：`adb shell cat /proc/asound/cards`。
*   **抓取硬件延迟**：在 HAL 源码中加入 `clock_gettime(CLOCK_MONOTONIC)`，计算 `pcm_write` 的耗时。

---

## 5. 常见难题：采样率不匹配 (Resampling)

如果 App 传来的采样率是 44.1k，但你的 HAL 声明只支持 48k。
*   **方案 A (推荐)**：让 AudioFlinger 完成重采样。HAL 只需在配置中诚实上报。
*   **方案 B (性能模式)**：HAL 内部调用厂商私有的 DSP 算法完成采样率转换（SRC），减轻主 CPU 负担。

---
*下一模块：[05. Linux 音频子系统 (Linux Audio Subsystem) - 驱动级实战](../05-Linux-Audio-Subsystem/README.md)*
