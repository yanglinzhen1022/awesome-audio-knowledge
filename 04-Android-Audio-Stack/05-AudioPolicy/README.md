# AudioPolicy 策略管理详解 (AudioPolicy Deep Dive)

如果说 AudioFlinger 是“体力劳动者”，那么 `AudioPolicy` 就是“决策层”。它负责回答一个终极问题：**“在这个时刻，这路声音，应该发往哪个物理设备？”**

---

## 1. 策略引擎的核心概念：Strategy vs. Usage

为了简化路由逻辑，Android 引入了 **Strategy (策略)**。

*   **Usage (应用层)**：如 `USAGE_MEDIA`（看视频）、`USAGE_ALARM`（闹钟）。
*   **Strategy (系统层)**：将相似的 Usage 归类。例如，`USAGE_MEDIA` 和 `USAGE_GAME` 通常被归类为 `STRATEGY_MEDIA`。
*   **Mapping (映射)**：
    *   **Usage -> Strategy**：在 `AudioPolicyManager::getStrategyForUsage()` 中硬编码或配置。
    *   **Strategy -> Device**：在 `AudioPolicyManager::getDeviceForStrategy()` 中动态计算。

---

## 2. 核心路由计算逻辑 (专家级)

当系统状态发生变化（如：耳机插入）时，`AudioPolicyManager` 会重新执行以下算法：

```cpp
// AudioPolicyManager.cpp 伪代码逻辑
audio_devices_t AudioPolicyManager::getDeviceForStrategy(routing_strategy strategy) {
    // 1. 检查是否有强行指定的路由 (如：开发者模式强制扬声器)
    if (mForceUse[FOR_MEDIA] == FORCE_SPEAKER) return AUDIO_DEVICE_OUT_SPEAKER;

    // 2. 根据策略进行优先级排序
    switch (strategy) {
        case STRATEGY_PHONE:
            // 通话时：蓝牙耳机 > 有线耳机 > 听筒 > 扬声器
            if (bluetooth_available) return AUDIO_DEVICE_OUT_BLUETOOTH_SCO;
            if (wired_headset_available) return AUDIO_DEVICE_OUT_WIRED_HEADSET;
            return AUDIO_DEVICE_OUT_EARPIECE;
            
        case STRATEGY_MEDIA:
            // 媒体音：A2DP蓝牙 > 有线耳机 > 扬声器
            if (a2dp_available) return AUDIO_DEVICE_OUT_BLUETOOTH_A2DP;
            return AUDIO_DEVICE_OUT_SPEAKER;
    }
}
```

---

## 3. 配置文件深度剖析：audio_policy_configuration.xml

这是定义系统硬件能力的“说明书”。小白常在这里配置错误，导致系统无法开机或无声。

```xml
<!-- 简化版配置示例 -->
<audioPolicyConfiguration version="7.0">
    <modules>
        <module name="primary" halVersion="3.0">
            <attachedDevices>
                <item>Speaker</item>
                <item>Built-In Mic</item>
            </attachedDevices>
            <mixPorts>
                <!-- 定义 AudioFlinger 提供的入口 -->
                <mixPort name="primary output" role="source" flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
            </mixPorts>
            <devicePorts>
                <!-- 定义 物理硬件出口 -->
                <devicePort tagName="Speaker" type="AUDIO_DEVICE_OUT_SPEAKER" role="sink">
                </devicePort>
            </devicePorts>
            <routes>
                <!-- 🚀 关键：定义哪些 MixPort 可以连接到哪个 DevicePort -->
                <route type="mix" sink="Speaker" sources="primary output"/>
            </routes>
        </module>
    </modules>
</audioPolicyConfiguration>
```

---

## 4. 车载系统的特殊性 (Automotive Specific)

在车载版 Android (AAOS) 中，AudioPolicy 发生了重大变化：
1.  **静态路由**：不再使用复杂的动态优先级，而是通过 `car_audio_configuration.xml` 强制指定某个 **Context** 走某个 **Bus**。
2.  **多音区支持**：APM 必须识别请求来自哪个音区 (Zone)，并根据音区 ID 返回对应的总线设备。

---

## 5. 专家调试手段

如果你发现声音“走错了地方”，使用以下命令排查：

*   `adb shell dumpsys audio`：查看当前的音频焦点 (Focus) 堆栈，看是谁抢占了声音。
*   `adb shell dumpsys media.audio_policy`：
    *   查看 **mAvailableOutputDevices**：看系统是否真的识别到了你的外设（如：USB声卡）。
    *   查看 **mOutputs**：查看当前每一个活跃流的路由路径。

---
*下一章：硬件之门 [Audio HAL 接口规范与 AIDL 实现](../06-AudioHAL.md)*
