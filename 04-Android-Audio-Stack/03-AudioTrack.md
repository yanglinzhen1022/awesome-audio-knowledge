# AudioTrack 深度解析 (AudioTrack Deep Dive)

`AudioTrack` 是 Android 播放链路的起点。对于开发者，它是播放 PCM 的工具；对于架构师，它是理解 **Linux 共享内存、Binder 异步通信、实时调度** 的最佳案例。

---

## 1. Java 层核心 API 使用与避坑

在 Java 层，实例化一个 `AudioTrack` 需要精确配置参数。

```java
// 核心配置示例
AudioAttributes attributes = new AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_MEDIA) // 用途：多媒体
        .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC) // 内容：音乐
        .build();

AudioFormat format = new AudioFormat.Builder()
        .setEncoding(AudioFormat.ENCODING_PCM_16BIT) // 量化精度
        .setSampleRate(44100) // 采样率
        .setChannelMask(AudioFormat.CHANNEL_OUT_STEREO) // 双声道
        .build();

// 获取系统建议的最小缓冲区（非常重要，决定了是否会卡顿）
int bufferSizeInBytes = AudioTrack.getMinBufferSize(44100, 
                    AudioFormat.CHANNEL_OUT_STEREO, 
                    AudioFormat.ENCODING_PCM_16BIT);

AudioTrack track = new AudioTrack(attributes, format, bufferSizeInBytes, 
                    AudioTrack.MODE_STREAM, AudioManager.AUDIO_SESSION_ID_GENERATE);
```

### 🧠 🧠 深度思考：为什么要有 getMinBufferSize？
`getMinBufferSize` 返回的不是一个随便的数字，它是由底层 HAL 层上报的硬件周期（Period Size）决定的。如果你的缓冲区小于这个值，AudioFlinger 还没来得及混合数据，硬件已经读完了，就会产生 **Underflow（断音/爆音）**。

---

## 2. JNI 层的“桥梁”作用

Java 层的 `AudioTrack.java` 只是个壳，核心逻辑在 Native 层的 `android_media_AudioTrack.cpp` 中。

```cpp
// 简化版的 JNI 逻辑展示 (frameworks/base/core/jni/android_media_AudioTrack.cpp)
static jint android_media_AudioTrack_setup(JNIEnv *env, jobject thiz, ...) {
    // 1. 在 Native 层创建一个真正的 AudioTrack 对象 (C++)
    // libaudioclient 提供的核心类
    sp<AudioTrack> lpTrack = new AudioTrack();
    
    // 2. 将 Native 对象的指针存入 Java 对象的私有成员变量变量 mNativeTrackInJavaObj 中
    // 这样后续的 write/play 才能找到对应的 C++ 对象
    env->SetLongField(thiz, javaAudioTrackFields.nativeTrackInJavaObj, (jlong)lpTrack.get());
}
```

---

## 3. Native 层：全链路初始化调用栈 (Expert Only)

这是理解 AudioTrack 启动过程最关键的代码路径。从 `set()` 到与 `AudioFlinger` 建立连接，经历了以下核心步骤：

### 3.1 核心调用栈源码级解析 (Call Stack)

1.  **`AudioTrack::set(...)`**：
    *   **职责**：校验参数（采样率、格式等），计算 `frameCount`。
    ```cpp
    status_t AudioTrack::set(audio_stream_type_t streamType, uint32_t sampleRate, ...) {
        // 校验采样率与格式
        if (!audio_is_valid_format(format)) return BAD_VALUE;
        // 计算每一帧的大小 (channels * bytes_per_sample)
        mFrameSize = audio_bytes_per_sample(format) * channelCount;
        // 随后进入关键的创建流程
        return createTrack_l();
    }
    ```

2.  **`AudioTrack::createTrack_l(...)`**：
    *   **策略查询**：调用 `AudioSystem::getOutputForAttr`。此步骤会向 `AudioPolicyManager` 请求一个 `audio_io_handle_t`（输出句柄）。
    ```cpp
    status_t AudioTrack::createTrack_l() {
        // 🚀 灵魂步骤：询问 Policy 大脑，我该去哪个输出线程？
        status = AudioSystem::getOutputForAttr(&mAttributes, &output, mSessionId, ...);
        
        // 拿到 output 句柄后，发起跨进程 Binder 调用
        sp<IAudioTrack> track = audioFlinger->createTrack(input, output, &status);
    }
    ```

3.  **`AudioFlinger::createTrack(...)`** (Server 侧执行)：
    *   **分配资源**：在对应的 `PlaybackThread` 中创建 `Track` 对象并分配匿名共享内存。
    ```cpp
    sp<IAudioTrack> AudioFlinger::createTrack(...) {
        // 找到对应的线程 (MixerThread / DirectThread)
        PlaybackThread *thread = checkPlaybackThread_l(output);
        // 创建 Track 实例，内部会分配 ashmem
        track = thread->createTrack_l(client, streamType, ...);
        // 返回 Binder 接口给 Client
        return new TrackHandle(track);
    }
    ```

### 3.2 建立同步机制 (Proxy Setup)
一旦 Binder 调用返回，Client 侧会执行：
```cpp
// AudioTrack.cpp 内部逻辑
mAudioTrackShared = track->getCblk(); // 获取控制块
mDataMemory = track->getBuffers();    // 获取数据区

// 🚀 初始化代理类
mProxy = new AudioTrackClientProxy(mAudioTrackShared, mDataMemory, ...);
```

---

## 4. 匿名共享内存与零拷贝机制

AudioTrack 内部维护了一个环形缓冲区。为了保证线程安全，引入了 `AudioTrackClientProxy` 和 `AudioTrackServerProxy`。

*   **App (Producer)**：调用 `write()` -> `AudioTrackClientProxy::obtainBuffer()` -> 填充 PCM -> `releaseBuffer()` (更新写指针)。
*   **AudioFlinger (Consumer)**：`AudioTrackServerProxy::obtainBuffer()` (读取数据) -> 送入混音器 -> `releaseBuffer()` (更新读指针)。

```mermaid
sequenceDiagram
    participant App as App (Client)
    participant SM as Shared Memory
    participant AF as AudioFlinger (Server)

    Note over App, AF: 播放初始化完成 (mmap 映射成功)
    App->>SM: 写入 PCM 数据 (Write Index 增加)
    App->>AF: 发送信号 (通知有新数据，可选)
    AF->>SM: 获取 PCM 数据 (Read Index 增加)
    AF->>HAL: 送入硬件播放
```

---

## 5. AudioTrack 状态机

AudioTrack 有严格的状态生命周期，违反状态转换是 App 层音频问题的首要原因：

```mermaid
stateDiagram-v2
    [*] --> IDLE: new AudioTrack()
    IDLE --> INITIALIZED: set() 成功
    INITIALIZED --> ACTIVE: play()
    ACTIVE --> PAUSED: pause()
    PAUSED --> ACTIVE: play() 恢复
    ACTIVE --> STOPPED: stop()
    STOPPED --> ACTIVE: play() 重新开始
    PAUSED --> STOPPED: stop()
    STOPPED --> FLUSHED: flush()
    FLUSHED --> ACTIVE: play()
    ACTIVE --> [*]: release()
    STOPPED --> [*]: release()
    PAUSED --> [*]: release()
```

**关键状态转换规则**：

| 操作 | 当前状态要求 | 违规后果 |
|:---|:---|:---|
| `play()` | INITIALIZED / STOPPED / PAUSED | IDLE 状态调用会返回 `INVALID_OPERATION` |
| `write()` | ACTIVE (MODE_STREAM) | STOPPED 状态写入数据会被丢弃 |
| `flush()` | STOPPED / PAUSED | ACTIVE 状态 flush 会导致数据丢失 |
| `stop()` | ACTIVE / PAUSED | 已有数据会被 drain 播完后才真正停止 |

---

## 6. Output Flag 与播放路径选择

AudioTrack 创建时携带的 Flag 直接决定了数据走哪条通路：

| Flag | 对应 Thread | 延迟 | Buffer 策略 | 典型场景 |
|:---|:---|:---|:---|:---|
| `FLAG_DEEP_BUFFER` | MixerThread (deep) | ~80-150ms | 大 buffer，省电 | 音乐播放 |
| `FLAG_FAST` | MixerThread (fast) | ~5-20ms | 小 buffer，FastTrack | 游戏/按键音 |
| `FLAG_LOW_LATENCY` | MixerThread (fast) | ~5-10ms | 最小 buffer | 实时音效 |
| `FLAG_DIRECT` | DirectOutputThread | ~5ms | 无混音 | 直通 DAC (Hi-Res) |
| `FLAG_MMAP_NOIRQ` | MmapThread | ~2-5ms | MMAP 共享 DMA | AAudio 独占 |
| `FLAG_COMPRESS_OFFLOAD` | OffloadThread | ~100ms+ | DSP 解码 | MP3/AAC 硬解 |

### 6.1 Flag 到 Thread 的映射流程

```mermaid
graph TD
    AT["AudioTrack 创建<br/>(携带 output flags)"] --> APM["AudioPolicyManager<br/>getOutputForAttr()"]
    APM --> |FLAG_DEEP_BUFFER| DEEP["MixerThread<br/>(deep_buffer output)"]
    APM --> |FLAG_FAST| FAST["MixerThread<br/>(FastTrack 子通道)"]
    APM --> |FLAG_DIRECT| DIRECT["DirectOutputThread<br/>(bypass mixer)"]
    APM --> |FLAG_COMPRESS_OFFLOAD| OFFLOAD["OffloadThread<br/>(DSP 硬解)"]
    APM --> |FLAG_MMAP_NOIRQ| MMAP["MmapThread<br/>(DMA 共享)"]
```

### 6.2 FastTrack 准入条件

不是所有 `FLAG_FAST` 请求都能进入 FastTrack，AudioFlinger 会做严格检验：

```cpp
// AudioFlinger::PlaybackThread::createTrack_l()
bool isTimed = (flags & AUDIO_OUTPUT_FLAG_FAST) != 0;
if (isTimed) {
    // 必须满足以下所有条件:
    // 1. 采样率与 HAL 输出一致 (无需重采样)
    // 2. 声道数 ≤ HAL 输出声道数
    // 3. 格式为 PCM_16 或 PCM_FLOAT
    // 4. frameCount ≤ FastMixer 的 frameCount
    // 5. 当前 FastTrack 槽位未满 (默认最多 8 条)
    if (!canUseFastTrack) {
        // 降级为 NormalTrack，走 MixerThread
        flags &= ~AUDIO_OUTPUT_FLAG_FAST;
    }
}
```

---

## 7. write() 全链路数据流

### 7.1 MODE_STREAM 写入流程

```mermaid
sequenceDiagram
    participant App as App Thread
    participant Proxy as AudioTrackClientProxy
    participant SHM as 环形共享内存 (ashmem)
    participant Server as ServerProxy (AudioFlinger)
    participant Mixer as AudioMixer
    participant HAL as Audio HAL
    
    App->>Proxy: write(buffer, size)
    Proxy->>Proxy: obtainBuffer() 获取可写区域
    Note over Proxy: 如果 buffer 已满则阻塞等待
    Proxy->>SHM: memcpy PCM 数据到环形区域
    Proxy->>Proxy: releaseBuffer() 更新 mRear 指针
    
    Note over Server: ThreadLoop 周期唤醒 (~5ms)
    Server->>SHM: obtainBuffer() 读取可用数据
    Server->>Mixer: 交给 AudioMixer 混音
    Mixer->>HAL: 混音结果写入 HAL
    Server->>Server: releaseBuffer() 更新 mFront 指针
```

### 7.2 环形缓冲区结构

```
共享内存布局:
┌─────────────────────────────────────────────────┐
│  audio_track_cblk_t (控制块, 64 bytes)          │
│  ├── mServer (读指针, AF 更新)                   │
│  ├── mPosition (写指针, App 更新)                │
│  ├── mVolumeLR (音量)                           │
│  ├── mSampleRate (采样率)                       │
│  └── mFlags (标志位)                            │
├─────────────────────────────────────────────────┤
│  PCM Data Buffer (环形区域)                      │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┐            │
│  │ W │ W │ R │ R │ R │ E │ E │ W │            │
│  └───┴───┴───┴───┴───┴───┴───┴───┘            │
│  W=已写未读  R=正在被AF读  E=空闲可写            │
└─────────────────────────────────────────────────┘

frameCount = bufferSizeInBytes / frameSize
```

---

## 8. 时间戳与 A/V 同步

### 8.1 getTimestamp() 原理

```java
AudioTimestamp ts = new AudioTimestamp();
if (track.getTimestamp(ts) == AudioTrack.SUCCESS) {
    long framesPlayed = ts.framePosition;   // 硬件已播放的帧数
    long nanoTime = ts.nanoTime;            // 对应的系统时间 (CLOCK_MONOTONIC)
    
    // 计算当前音频播放位置 (考虑 buffer 中的延迟)
    long currentFrames = framesPlayed + 
        (System.nanoTime() - nanoTime) * sampleRate / 1_000_000_000L;
}
```

### 8.2 延迟分解

```
端到端延迟 = App Buffer 延迟 + AudioFlinger Buffer 延迟 + HAL Buffer 延迟 + 硬件延迟

典型数值 (48kHz, deep_buffer):
  App Buffer:    ~8192 frames / 48000 = 170ms
  AF Buffer:     ~960 frames / 48000 = 20ms
  HAL Buffer:    ~960 frames / 48000 = 20ms
  硬件 (DSP+DAC): ~2ms
  合计:           ~212ms

典型数值 (48kHz, low_latency):
  App Buffer:    ~480 frames / 48000 = 10ms
  AF Buffer:     ~240 frames / 48000 = 5ms
  HAL Buffer:    ~240 frames / 48000 = 5ms
  硬件:          ~2ms
  合计:           ~22ms
```

---

## 9. Offload 播放 (压缩音频硬件解码)

当 AudioTrack 使用 `ENCODING_AAC_LC`/`ENCODING_MP3` 等压缩格式时，数据直接发送到 DSP 硬件解码器：

```mermaid
graph LR
    APP["App<br/>write(compressed data)"] --> AF["OffloadThread"]
    AF --> |"压缩帧"| HAL["Audio HAL<br/>(compress_offload)"]
    HAL --> DSP["ADSP 解码器<br/>(MP3/AAC/FLAC)"]
    DSP --> |"PCM"| DAC["DAC → Speaker"]
```

**Offload 优势**：CPU 接近零负载（数据搬运由 DMA 完成），适合后台长时播放省电。

**Offload 限制**：
- 不经过 AudioMixer → 无法与其他流混音
- 音效必须在 DSP 侧实现
- 仅支持 `MODE_STREAM`
- `pause()`/`resume()`/`drain()` 有特殊语义

---

## 10. start() 启动流程 (play 到硬件播放)

调用 `play()` 后的完整链路，从 Java 到硬件开始输出：

### 10.1 调用栈源码解析

```cpp
// frameworks/av/media/libaudioclient/AudioTrack.cpp
void AudioTrack::start() {
    AutoMutex lock(mLock);
    
    if (mState == STATE_ACTIVE) return;  // 已经在播放
    
    mState = STATE_ACTIVE;
    
    // 如果之前是 STOPPED/FLUSHED，重置位置
    if (previousState == STATE_STOPPED || previousState == STATE_FLUSHED) {
        mPosition = 0;
        mProxy->setEpoch(mProxy->getEpoch() - mProxy->getPosition());
    }
    
    // 关键：通过 Binder 通知 AudioFlinger 侧激活 Track
    status_t status = mAudioTrack->start();
    
    if (status != NO_ERROR) {
        mState = previousState;  // 失败回滚
    }
}
```

### 10.2 Server 侧 (AudioFlinger) 处理

```cpp
// frameworks/av/services/audioflinger/Tracks.cpp
status_t AudioFlinger::PlaybackThread::Track::start() {
    // 1. 状态检查
    if (isTerminated() || mState == PAUSING || mState == IDLE) {
        return INVALID_OPERATION;
    }
    
    // 2. 切换到 ACTIVE 状态
    mState = ACTIVE;
    
    // 3. 加入所属 PlaybackThread 的活跃列表
    PlaybackThread *playbackThread = (PlaybackThread *)thread.get();
    status = playbackThread->addTrack_l(this);
    
    return status;
}

// frameworks/av/services/audioflinger/Threads.cpp
status_t AudioFlinger::PlaybackThread::addTrack_l(const sp<Track>& track) {
    // 加入 mActiveTracks
    mActiveTracks.add(track);
    
    // 如果 Thread 处于 standby，唤醒它
    if (mStandby) {
        mWaitWorkCV.signal();  // 唤醒 threadLoop()
    }
    
    return NO_ERROR;
}
```

### 10.3 时序总结

```
App: track.play()
  → AudioTrack::start()
    → mAudioTrack->start() [Binder IPC]
      → TrackHandle::start()
        → Track::start()
          → mState = ACTIVE
          → PlaybackThread::addTrack_l(track)
            → mActiveTracks.add(track)
            → 唤醒 threadLoop() (如果在 standby)
              → threadLoop() 开始从共享内存读取数据
              → AudioMixer 混音
              → HAL write() → 硬件输出声音
```

---

## 11. Callback 回调机制与事件驱动模型

### 11.1 Native Callback 事件类型

```cpp
// frameworks/av/media/libaudioclient/include/media/AudioTrack.h
class AudioTrack {
public:
    enum event_type {
        EVENT_MORE_DATA = 0,      // 需要更多数据 (callback mode 核心事件)
        EVENT_UNDERRUN = 1,       // 缓冲区下溢
        EVENT_LOOP_END = 2,       // 循环播放到末尾
        EVENT_MARKER = 3,         // 到达设定的帧位置标记
        EVENT_NEW_POS = 4,        // 位置更新通知 (周期性)
        EVENT_BUFFER_END = 5,     // MODE_STATIC buffer 播放完毕
        EVENT_NEW_IAUDIOTRACK = 6,// Track 被重建 (AudioFlinger 恢复后)
        EVENT_STREAM_END = 7,     // Offload 模式: 流播放结束
        EVENT_NEW_TIMESTAMP = 8,  // 新时间戳可用
    };
    
    // 回调函数签名
    typedef void (*callback_t)(int event, void* user, void* info);
};
```

### 11.2 Callback 模式 vs Write 模式对比

```cpp
// === Write 模式 (MODE_STREAM + 手动 write) ===
// App 负责在独立线程中循环调用 write()
void playbackThread() {
    track->play();
    while (playing) {
        track->write(buffer, size, WRITE_BLOCKING);  // 阻塞等待空间
    }
}

// === Callback 模式 (数据回调) ===
// AudioFlinger 在需要数据时主动回调 App
void audioCallback(int event, void* user, void* info) {
    if (event == AudioTrack::EVENT_MORE_DATA) {
        AudioTrack::Buffer* buf = (AudioTrack::Buffer*)info;
        // 填充 PCM 数据到 buf->raw
        generateAudio(buf->raw, buf->size);
        // 返回后 AudioFlinger 自动消费
    }
}

// 创建 callback 模式的 AudioTrack
sp<AudioTrack> track = new AudioTrack();
track->set(AUDIO_STREAM_MUSIC, sampleRate, format, channelMask,
           frameCount,
           AUDIO_OUTPUT_FLAG_FAST,
           audioCallback,    // 回调函数
           userData,         // 回调上下文
           0,               // notificationFrames (0=默认)
           sharedBuffer,    // nullptr for STREAM mode
           false,           // threadCanCallJava
           sessionId);
```

### 11.3 Callback 线程模型

```
Callback 模式内部工作原理:

  AudioTrack 内部创建 AudioTrackThread:
    → 独立线程, 优先级 ANDROID_PRIORITY_AUDIO
    → threadLoop() 中循环:
      1. mProxy->obtainBuffer() — 等待可写空间
      2. 调用 mCbf(EVENT_MORE_DATA, ...) — 回调 App 填数据
      3. mProxy->releaseBuffer() — 提交数据

  优势:
    - App 无需管理写入线程
    - 系统自动按 period 节奏请求数据
    - 天然适合实时音频合成 (游戏/合成器)
    
  注意:
    - 回调中不能做耗时操作 (阻塞/锁/malloc)
    - 回调线程优先级高, 避免优先级反转
```

---

## 12. MODE_STATIC vs MODE_STREAM 实现差异

### 12.1 核心区别

```
MODE_STREAM:
  App: write() → 环形 Buffer → AudioFlinger 持续消费
  共享内存: 环形缓冲区 (固定大小, 循环覆写)
  适用: 持续流式播放 (音乐/通话/游戏背景音)

MODE_STATIC:
  App: 一次性写入完整 buffer → play() → 硬件循环播放
  共享内存: 线性 Buffer (一次性分配全部数据大小)
  适用: 短促音效 (按键音/通知音/游戏打击音)
```

### 12.2 内部实现源码

```cpp
// frameworks/av/media/libaudioclient/AudioTrack.cpp

// MODE_STATIC 的 buffer 分配
status_t AudioTrack::set(...) {
    if (mTransfer == TRANSFER_SHARED) {  // MODE_STATIC
        // 分配整个音频大小的共享内存 (非环形)
        mSharedBuffer = sharedBuffer;  // App 传入的 buffer
        frameCount = mSharedBuffer->size() / mFrameSize;
        // 不需要 AudioTrackClientProxy 的环形逻辑
    }
}

// MODE_STATIC 的 write (一次性)
ssize_t AudioTrack::write(const void* buffer, size_t size, ...) {
    if (mTransfer == TRANSFER_SHARED) {
        // 直接 memcpy 到共享内存起始位置
        memcpy(mSharedBuffer->unsecurePointer(), buffer, size);
        return size;  // 立即返回
    }
    // MODE_STREAM: 走环形 buffer 的 obtainBuffer/releaseBuffer
    ...
}
```

### 12.3 性能对比

| 维度 | MODE_STREAM | MODE_STATIC |
|:---|:---|:---|
| 首次播放延迟 | 低 (边写边播) | 高 (需先写完所有数据) |
| 重复播放延迟 | 高 (需重新填充) | **极低** (数据已在共享内存) |
| 内存占用 | 小 (仅 buffer_size) | 大 (完整音频数据) |
| CPU 占用 | 持续 (不断 write) | 极低 (play 后无需 CPU) |
| 循环播放 | App 自行控制 | `setLoopPoints()` 硬件级循环 |
| 适用时长 | 无限制 | 建议 < 5秒 (内存限制) |

```java
// MODE_STATIC 典型用法 (短音效)
AudioTrack staticTrack = new AudioTrack.Builder()
    .setAudioAttributes(attrs)
    .setAudioFormat(format)
    .setBufferSizeInBytes(totalPcmBytes)  // 完整音效大小
    .setTransferMode(AudioTrack.MODE_STATIC)
    .build();

// 一次性写入
staticTrack.write(pcmData, 0, pcmData.length);

// 设置循环: 从头到尾循环 3 次
staticTrack.setLoopPoints(0, totalFrames, 3);
staticTrack.play();  // 无需再 write, 硬件自动循环
```

---

## 13. restoreTrack_l() — AudioFlinger 崩溃恢复

### 13.1 恢复机制

当 `audioserver` 进程崩溃重启后，所有 AudioTrack 的 Binder 连接断开。Android 设计了自动恢复机制：

```cpp
// frameworks/av/media/libaudioclient/AudioTrack.cpp
status_t AudioTrack::restoreTrack_l(const char *from) {
    ALOGW("dead IAudioTrack, %s, creating a new one from %s()", 
          isOffloadedOrDirect_l() ? "Offloaded or Direct" : "", from);
    
    // 1. 保存当前播放位置
    uint32_t position = mPosition;
    
    // 2. 重新查询 AudioPolicy 获取 output
    audio_io_handle_t output;
    AudioSystem::getOutputForAttr(&mAttributes, &output, ...);
    
    // 3. 重新创建 Track (与初始化流程相同)
    result = createTrack_l();
    
    if (result == NO_ERROR) {
        // 4. 恢复播放位置
        mProxy->setEpoch(mProxy->getEpoch() - mPosition);
        
        // 5. 如果之前在播放, 重新 start
        if (mState == STATE_ACTIVE) {
            mAudioTrack->start();
        }
        
        // 6. 触发回调通知 App
        // EVENT_NEW_IAUDIOTRACK 告知 App: Track 已重建
    }
    return result;
}
```

### 13.2 触发时机

```
AudioFlinger 崩溃恢复时序:

  audioserver 崩溃 (crash/kill/ANR watchdog)
       │
       ▼
  system_server 检测到死亡 → 重新启动 audioserver
       │
       ▼
  App 侧 AudioTrack 下次操作 (write/getTimestamp/...)
    → Binder 调用失败 (DEAD_OBJECT)
       │
       ▼
  restoreTrack_l() 被触发:
    → 重新 getOutputForAttr() (可能路由已变化)
    → 重新 createTrack_l() (新的共享内存)
    → 恢复位置 & 状态
    → 用户无感知 (最多有一次短暂中断 ~200ms)

特殊情况:
  - Offload Track: 无法恢复位置 (DSP 状态丢失), 需要 App 重新 seek
  - Direct Track: 可能因设备变化无法恢复, 返回错误
```

### 13.3 App 侧感知

```java
// App 通常不需要处理恢复, 但可以监听:
track.setPlaybackPositionUpdateListener(new AudioTrack.OnPlaybackPositionUpdateListener() {
    @Override
    public void onMarkerReached(AudioTrack track) { }
    
    @Override
    public void onPeriodicNotification(AudioTrack track) {
        // 如果 AudioFlinger 重启, 这个回调可能会有一次间断
        // 正常情况下 App 无需额外处理
    }
});

// Native 层 callback 会收到 EVENT_NEW_IAUDIOTRACK
// 通知 App: Track 已被重建, 可能需要重新设置音量/音效等参数
```

---

## 14. 常见问题与专家级排查

| 问题 | 根因 | 解决方案 |
|:---|:---|:---|
| **write() 阻塞** | 共享内存已满，AF 消费太慢 | 检查 underrun 日志；不要在 UI 线程 write |
| **Underrun (断音)** | App 写入速度跟不上 HAL 消费 | 增大 buffer / 提高写入线程优先级 |
| **延迟过高** | 使用了 deep_buffer | 改用 `FLAG_LOW_LATENCY` 或 AAudio |
| **getTimestamp 返回失败** | Track 未进入 ACTIVE 或 HAL 不支持 | 确认 play() 后等待足够帧数再查询 |
| **stop() 后有残响** | drain 机制：HAL 需播完剩余数据 | 使用 flush() 强制清空 |
| **多次 play/stop 后无声** | Track 进入异常状态 | 检查 `getState()` + 重新 release/create |

### 14.1 调试命令

```bash
# 查看当前所有 Track 状态
adb shell dumpsys media.audio_flinger | grep -A 20 "Track"

# 查看 underrun 统计
adb shell dumpsys media.audio_flinger | grep -i "underrun"

# 查看共享内存使用
adb shell dumpsys media.audio_flinger | grep "cblk"
```

---

## 15. 关键参考 (References)

1.  [AOSP AudioTrack.cpp](https://android.googlesource.com/platform/frameworks/av/+/refs/heads/main/media/libaudioclient/AudioTrack.cpp)
2.  [Android Developer: AudioTrack](https://developer.android.com/reference/android/media/AudioTrack)
3.  [Android Audio Latency](https://source.android.com/docs/core/audio/latency)
4.  [AOSP AudioTrackShared.h (环形缓冲区实现)](https://android.googlesource.com/platform/frameworks/av/+/refs/heads/main/media/libaudioclient/include/media/AudioTrackShared.h)

---
*下一章：[AudioRecord 录音流程解析](./04-AudioRecord.md)*
