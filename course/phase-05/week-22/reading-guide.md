# 第22周 阅读指南：GPU 性能优化

> 对应教材：CAQA 6th Edition, Chapter 4 §4.9-4.11

## 学习目标

1. 掌握内存合并访问（Coalescing）的原理
2. 理解 Bank Conflict 及其对共享内存性能的影响
3. 能应用 Roofline 模型分析程序瓶颈

## 核心概念

### 内存合并访问（Memory Coalescing）
- GPU 全局内存事务以 32/128 字节为单位（取决于架构）
- **合并访问**：同一 warp 的线程访问连续的地址 → 1 次事务
- **非合并访问**：地址分散 → 多次事务，带宽利用率极低
- 关键规则：第 i 个线程应访问第 i 个字（自然对齐）

### 共享内存 Bank Conflict
- 共享内存分为 32 个 bank（与 warp 大小匹配）
- 同 bank 同时被多个线程访问 → **bank conflict**（串行化）
- N-way bank conflict：N 个线程访问同一 bank → N 倍延迟
- 避免方法：填充（padding）使数组宽度不是 32 的倍数

### Occupancy 与性能
- 高 Occupancy ≠ 高性能（但低 Occupancy 往往低性能）
- 当 kernel 是计算瓶颈时，Occupancy > 50% 通常够用
- 当 kernel 是访存瓶颈时，需要更高 Occupancy 来隐藏延迟

### Roofline 模型
\[
\text{Attainable GFLOPS} = \min\left(\text{Peak GFLOPS},\ \text{Peak BW} \times \text{Arithmetic Intensity}\right)
\]

- **算术强度**：每字节访存对应的 FLOP（FLOP/Byte）
- **计算瓶颈区域**：高算术强度，受限于计算能力
- **访存瓶颈区域**：低算术强度，受限于内存带宽

### Warp Shuffle
- `__shfl_sync()`：warp 内线程直接交换寄存器数据
- 延迟远低于共享内存（~1 cycle vs ~20 cycles）
- 适用于 warp 级归约

### Tiling（分块）
- 将全局数据分块加载到共享内存，减少全局访存
- 矩阵乘法 tiling：\(O(N^3)\) 计算 vs \(O(N^2)\) 数据 → 算术强度 \(O(N)\)

## 关键计算

**非合并访问性能损失**：
若 stride 访问导致每线程一次不同的事务段：带宽利用率为 \(1/stride\)。

## 阅读任务

1. 精读 §4.9：GPU 性能分析工具与方法论
2. 精读 §4.10：优化案例——从 Naive 到 Tiled 矩阵乘
3. 精读 §4.11：异构系统中的负载均衡与任务划分

## 思考题

1. 为什么 `A[row][col]` 访问是合并的，而 `A[col][row]`（stride=N）不是？
2. 共享内存 bank conflict 和全局内存非合并访问，哪个对性能影响更大，为什么？
3. 一个 kernel 的算术强度为 0.5 FLOP/Byte，GPU 峰值 10 TFLOPS、带宽 500 GB/s，它处于 roofline 的哪个区域？
