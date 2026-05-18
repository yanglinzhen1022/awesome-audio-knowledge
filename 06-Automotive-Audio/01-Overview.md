# 车载音频系统概览 (Automotive Audio System Overview)

车载音频系统比手机更强调**安全性、隔离性与实时性**。在 Android Automotive (AAOS) 架构中，音频不再由单个 App 控制，而是受全局车辆策略约束。

---

## 1. AAOS 音频控制：CarAudioService

在手机版 Android 中，`AudioService` 负责一切。在车机中，由 `CarAudioService` 接管并扩展了其功能。

### 1.1 Context (音频上下文)
AAOS 不直接操作 Usage，而是定义了 **Context**。每一个 Context 对应一类业务逻辑。
*   **MUSIC**：音乐播放。
*   **NAVIGATION**：导航播报。
*   **VOICE_COMMAND**：语音助手。
*   **CALL**：电话通话。
*   **ALARM**：车辆报警音。

### 1.2 Context 与 Bus 的静态绑定
在配置文件中，Context 被永久绑定到具体的硬件 Bus 上，这种设计保证了路由的**确定性**。

---

## 2. 安全与直通路径 (Direct Path)

由于汽车的特殊性，某些声音（如：倒车雷达、未系安全带警报）**绝对不允许延迟或静音**。

### 2.1 硬件直通 (Safety Bypass)
现代座舱设计中，关键报警音不经过 Android 系统，而是直接由 **MCU (微控制器)** 通过 I2S 或模拟线路直接输入到外部功放 (DSP Amp) 的混合节点。即使 Android 崩溃或正在冷启动，警报音也能即时响起。

---

## 3. 车载音频的核心指标

*   **全双工性能**：在高速行驶（80km/h+）的高噪声环境下，实现清晰的语音通话。
*   **分区隔离度**：驾驶员说话，后排语音助理不应被唤醒。
*   **启动时间**：从车辆上电到第一声倒车雷达响起的延迟通常要求小于 **2 秒**。

---

## 4. 专家调试：car_audio_configuration.xml 解析

```xml
<!-- 关键配置片段 -->
<audioZone id="0" isPrimary="true">
    <volumeGroups>
        <group>
            <device address="bus0_media">
                <context context="music"/>
            </device>
            <device address="bus1_navigation">
                <context context="navigation"/>
            </device>
        </group>
    </volumeGroups>
</audioZone>
```
*专家提示：通过 `adb shell dumpsys car_service` 可以查看当前系统中所有音区的 Bus 绑定状态。*

---

## 5. 关键参考 (References)

1.  [Android Automotive Audio Official Guide](https://source.android.com/devices/automotive/audio)
2.  [ISO 26262 - Functional Safety for Audio](https://www.iso.org/standard/43906.html)
