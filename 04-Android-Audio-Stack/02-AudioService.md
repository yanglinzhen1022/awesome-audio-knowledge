# AudioService 系统管理中心

`AudioService` 是 Android 音频框架在 Java 层的核心，运行在 `system_server` 进程中。它扮演着“总管”的角色，协调应用层请求与 Native 层服务。

---

## 1. 启动与初始化流程 (Initialization Sequence)

`AudioService` 的初始化由 `SystemServer` 触发。

### 1.1 核心启动路径
1.  **`SystemServer.java`**：调用 `mSystemServiceManager.startService(AudioService.Lifecycle.class)`。
2.  **`AudioService` 构造函数**：
    *   **创建 Handler**：初始化 `AudioHandler`，用于异步处理音量和状态变更。
    *   **数据库监听**：注册对系统音量设置（Settings.System）的监听。
    *   **连接 Native**：建立与 `AudioPolicyService` 的 Binder 连接。

---

## 2. 核心职责

1.  **外部策略入口**：处理来自 `AudioManager` 的所有 Java API 请求。
2.  **音量管理**：维护不同 Stream 类型的音量 Index，并持久化到设置数据库中。
3.  **设备连接状态**：监听耳机、蓝牙、外置声卡的插拔事件。
4.  **音频焦点**：通过 `MediaFocusControl` 维护焦点栈。

---

## 3. 源码级工作模式：Handler 异步架构

为了不阻塞 SystemServer 主线程，`AudioService` 大量使用了 `AudioHandler` 处理耗时任务。

```java
// AudioService.java 内部消息机制
private class AudioHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_SET_DEVICE_VOLUME:
                // 异步设置底层增益
                onSetDeviceVolume((VolumeStreamState) msg.obj, msg.arg1);
                break;
            case MSG_BROADCAST_AUDIO_BECOMING_NOISY:
                // 广播“耳机拔出”消息，通知 App 自动暂停
                sendBecomingNoisyIntent();
                break;
        }
    }
}
```

---

## 4. 专家级排查：音量突变与 Safe Volume

如果你发现音量会自动跳变，通常与以下逻辑有关：
*   **Safe Volume**：Android 强制的听力保护逻辑。当使用耳机时长超过限制或音量过高时，系统会强制下调音量索引。
*   **Volume Group 映射**：检查 `audio_policy_configuration.xml` 中的映射是否重叠，导致一次调节影响了多个流。

---

## 5. 关键参考 (References)

1.  [AOSP Source: AudioService.java](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/audio/AudioService.java)
2.  [Android Volume Control Mechanism - Google Docs](https://source.android.com/docs/core/audio/volume)

---
*下一章：[Android 音频系统概览 (Overview)](./02-Overview.md)*

---
*Next Topic: [AudioTrack 播放流程解析](./03-AudioTrack.md)*
