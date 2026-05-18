# AudioFocus 音频焦点机制

为了解决多个应用同时播放音频时的冲突问题（如：听歌时来电话），Android 引入了 **AudioFocus** 机制。

---

## 1. 核心原理：焦点栈 (Focus Stack)

AudioFocus 本质上是一种**基于协作的多应用通信机制**。系统在 `MediaFocusControl` 中维护了一个焦点栈。

*   **唯一性**：通常情况下，栈顶的应用拥有音频焦点。
*   **后入先得**：新申请焦点的应用会被压入栈顶，原焦点拥有者会收到焦点丢失通知。

---

## 2. 焦点申请类型 (Gain Types)

开发者必须根据播放意图选择合适的申请类型：

| 申请类型 | 典型场景 | 备注 |
| :--- | :--- | :--- |
| `AUDIOFOCUS_GAIN` | 播放音乐、视频。 | 长时间占用，会打断其他应用。 |
| `AUDIOFOCUS_GAIN_TRANSIENT` | 导航播报、闹钟。 | 短时间占用。 |
| `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK` | 微信消息提示音。 | 允许原应用降低音量播放（Ducking）。 |
| `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE`| 语音识别、录音。 | 不允许任何其他声音干扰。 |

---

## 3. 焦点丢失处理 (Loss Types)

当你的 App 失去焦点时，必须做出响应：

*   **`AUDIOFOCUS_LOSS`**：永久失去焦点。**必须停止播放**并释放资源。
*   **`AUDIOFOCUS_LOSS_TRANSIENT`**：暂时失去焦点（如来电）。应该暂停播放，待焦点恢复后继续。
*   **`AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK`**：暂时失去焦点，但允许继续播放。应该主动降低音量。

---

## 4. 源码级处理流程

1.  **App 侧**：调用 `AudioManager.requestAudioFocus()`。
2.  **SystemServer 侧**：`AudioService` 将请求转发给 `MediaFocusControl`。
3.  **栈操作**：检查 `canReassignAudioFocus()`。如果是抢占式，将新请求压入 `mFocusStack`。
4.  **回调通知**：调用原焦点拥有者的 `IAudioFocusDispatcher` 接口，触发其 `onAudioFocusChange` 回调。

---

## 5. 专家调试与 Log 分析

### 5.1 查看焦点栈状态
`adb shell dumpsys audio`
搜索 **"Audio Focus stack entries"**，你可以看到当前哪个 UID 持有焦点，以及栈内排队的应用。

### 5.2 实战案例：为什么两边同时出声？
*   **原因 1**：其中一个 App 没有申请焦点（流氓 App）。
*   **原因 2**：两个 App 使用了不同的 **AudioZone**（在车载场景中常见）。
*   **原因 3**：App 收到 `LOSS_TRANSIENT_CAN_DUCK` 后没有主动降低音量代码。

---
*下一模块：[AudioService 系统管理中心](./09-AudioService.md)*

---
*Next Module: [05. Linux 音频子系统 (Linux Audio Subsystem)](../05-Linux-Audio-Subsystem/README.md)*
