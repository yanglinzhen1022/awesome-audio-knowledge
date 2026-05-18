# 音效处理 (Audio Effects: EQ, DRC, Spatial Audio)

音效处理是对音频信号进行深度加工，以达到艺术修饰或硬件补偿的目的。本章通过传递函数和控制曲线深入剖析其原理。

---

## 1. 均衡器 (EQ, Equalizer) 深度细节

### 1.1 数字滤波器的实现：Biquad (双二阶)
几乎所有的数字 PEQ 都是由 Biquad 滤波器级联而成的。其差分方程为：
$$y[n] = b_0x[n] + b_1x[n-1] + b_2x[n-2] - a_1y[n-1] - a_2y[n-2]$$

### 1.2 核心参数的数学意义
*   **$f_c$ (中心频率)**：滤波器作用的频率点。
*   **Gain (增益)**：波峰或波谷的高度。
*   **Q 值**：决定频带宽度。带宽 $BW = f_c / Q$。

---

## 2. 动态范围控制 (DRC) 控制逻辑

DRC 的核心是**增益映射曲线 (Gain Mapping)**。

### 2.1 压缩器 (Compressor) 的转换函数
在对数域（dB）中，转换函数定义为：
$$y_{dB} = \begin{cases} x_{dB} & \text{if } x_{dB} < T \\ T + \frac{x_{dB} - T}{R} & \text{if } x_{dB} \ge T \end{cases}$$
*   $T$：阈值 (Threshold)
*   $R$：压缩比 (Ratio)

### 2.2 专家调试：膝点 (Knee)
*   **Hard Knee**：增益在阈值处突然改变。
*   **Soft Knee**：在阈值附近平滑过渡，听感更自然，减少处理痕迹。

---

## 3. 空间音频与 HRTF

空间音频的核心是将单声道信号通过卷积处理，模拟空间方位感。

### 3.1 HRTF 卷积过程
$$y_L(n) = x(n) * h_L(\theta, \phi, n)$$
$$y_R(n) = x(n) * h_R(\theta, \phi, n)$$
*   $h_L, h_R$：对应特定方位 $(\theta, \phi)$ 的冲激响应（从 HRTF 数据库中提取）。

### 3.2 距离感模拟
*   **混响 (Reverb)**：早期反射音比例越高，听感距离越近；后期混响越长，空间越大。
*   **空气吸收**：距离越远，高频成分衰减越快（低通滤波效应）。

---

## 4. Android 音效集成框架 (Effect Framework)

在 Android 系统中，音效根据其作用位置被分为三个层级：

1.  **Track Effects**：挂载在单个应用流上（如：均衡器只对播放器生效）。
2.  **Stream Effects**：挂载在特定流类型上（如：全局压低所有的 Music 流）。
3.  **Device Effects**：挂载在硬件出口（如：针对某个特定的扬声器进行的频率校准）。

### 🚀 架构趋势：HAL 侧处理
从 Android 13 开始，Google 推荐将绝大多数音效算法从 `AudioServer` 下沉到 `AudioHAL` 进程。这样可以利用厂商私有的 DSP 算力，降低主 CPU 功耗并减少延迟。

---

## 5. 关键参考 (References)

1.  *Introduction to Sound Processing* - Davide Rocchesso
2.  [The Biquad Cookbook - Robert Bristow-Johnson](https://chromium.googlesource.com/chromium/src/+/master/third_party/blink/renderer/platform/audio/Biquad.h)
3.  [Google Resonance Audio Technical Guide](https://github.com/resonance-audio/resonance-audio)
