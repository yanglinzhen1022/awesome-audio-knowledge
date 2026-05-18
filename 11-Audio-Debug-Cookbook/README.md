# 11. 音频调试手册 (Audio Debug Cookbook)

本模块提供全链路音频调试的系统方法论与实战 Cookbook，涵盖从 App 层到硬件层的问题定位思路、工具使用与典型案例。

## 📖 章节导航

1.  **[全链路音频调试方法论](./01-Debug-Methodology.md)**
    *   音频数据流全链路拆解与分层定位。
    *   Systrace / Perfetto 音频分析实战。
    *   音频 Tombstone / ANR / Crash 分析。
    *   常见问题 Cookbook：无声、爆音、延迟、功耗、路由异常。

2.  **[跨模块全链路数据流](./02-Cross-Layer-Data-Flow.md)**
    *   场景一：从 App `play()` 到喇叭出声的 7 阶段数据流与延迟分解。
    *   场景二：耳机插入的全栈变化（Hardware → Kernel → Policy → DAPM）。
    *   知识库模块关联地图：串联 01-11 各模块。

---
[返回主目录](../README.md)
