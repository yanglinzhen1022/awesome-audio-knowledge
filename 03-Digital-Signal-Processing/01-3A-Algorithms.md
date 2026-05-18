# 语音通信 3A 算法 (AEC, ANS, AGC)

在语音通话和会议系统中，3A 算法是保证通话质量的核心。本章将从数学模型和伪代码层面深入探讨其实现。

---

## 1. 回声消除 (AEC, Acoustic Echo Cancellation)

### 1.1 核心算法：NLMS (归一化最小均方)
AEC 的目标是找到一个滤波器系数向量 $\mathbf{w}$，使得估计的回声 $\hat{y}(n) = \mathbf{w}^T \mathbf{x}(n)$ 尽可能接近真实回声 $y(n)$。

**NLMS 更新公式**：
$$\mathbf{w}(n+1) = \mathbf{w}(n) + \mu \frac{e(n) \mathbf{x}(n)}{\|\mathbf{x}(n)\|^2 + \epsilon}$$
*   $\mu$：步长（决定收敛速度）。
*   $e(n)$：残差信号（麦克风输入 - 估计回声）。
*   $\epsilon$：防止分母为零的微小常数。

### 1.2 核心逻辑伪代码 (Python 风格)
```python
def process_aec(far_end_ref, mic_in, filter_state):
    # 1. 卷积：生成估计回声
    echo_hat = np.dot(filter_state.weights, far_end_ref)
    
    # 2. 减法：计算误差信号 (这就是我们要发送给对方的声音)
    error = mic_in - echo_hat
    
    # 3. 自适应更新：调整滤波器权重
    norm_factor = np.dot(far_end_ref, far_end_ref) + 1e-6
    filter_state.weights += step_size * (error * far_end_ref) / norm_factor
    
    return error
```

### 1.3 专家挑战：双讲检测 (Double-Talk Detection)
如果近端也在说话，自适应滤波器会将近端语音误认为误差，从而导致权重跑飞。专业 AEC 必须包含 DTD 逻辑：只有当远端有声音且近端无声音时才更新权重。

---

## 2. 自动降噪 (ANS, Automatic Noise Suppression)

### 2.1 谱减法 (Spectral Subtraction) 原理
假设噪声 $N(f)$ 与语音 $S(f)$ 是加性关系：$Y(f) = S(f) + N(f)$。
*   **步骤**：
    1.  在没有语音的间隙估计噪声功率谱 $\hat{P}_n(f)$。
    2.  从带噪信号谱中减去噪声谱：$|S(f)|^\gamma = |Y(f)|^\gamma - \alpha |\hat{P}_n(f)|^\gamma$。
*   **副作用**：容易产生“音乐噪声”（残留的随机孤立频谱点）。专业算法会使用**维纳滤波 (Wiener Filter)** 来平滑处理。

---

## 3. 自动增益控制 (AGC, Automatic Gain Control)

### 3.1 动态控制回路
AGC 像是一个自动调音师，其核心是 **Envelope Follower (包络跟踪器)**。

**增益计算逻辑**：
1.  **Level Detection**：计算信号的 RMS 或峰值。
2.  **Gain Computation**：如果电平低于目标值，增加增益；如果产生剪切，迅速压低。
3.  **Smoothing**：
    *   **Attack**：当需要减小增益时，动作要快（防止破音）。
    *   **Release**：当需要增加增益时，动作要慢（防止背景底噪呼吸效应）。

---

## 4. 关键参考 (References)

1.  *Adaptive Filter Theory* - Simon Haykin
2.  [WebRTC Audio Processing Implementation](https://webrtc.googlesource.com/src/+/refs/heads/main/modules/audio_processing/)
3.  [Acoustic Echo Cancellation - Wikipedia](https://en.wikipedia.org/wiki/Echo_suppression_and_cancellation)
