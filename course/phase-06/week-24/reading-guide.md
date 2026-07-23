# 第24周 阅读指南：多处理器体系结构

> 对应教材：CAQA 6th Edition, Chapter 5 §5.1-5.3

## 学习目标

1. 理解多处理器分类：SMP、DSM、Multicore
2. 掌握共享内存模型：UMA vs NUMA
3. 能分析多处理器系统的可扩展性瓶颈

## 核心概念

### 多处理器分类

| 类型 | 特征 | 延迟 | 例子 |
|------|------|------|------|
| SMP (Symmetric Multiprocessor) | 所有处理器平等访存 | 均匀 | 早期多核 x86 |
| DSM (Distributed Shared Memory) | 物理分布，逻辑共享 | 非均匀 | SGI Origin |
| Multicore | 单芯片多处理器 | 低核间延迟 | 现代所有 CPU |

### 共享内存模型

**UMA (Uniform Memory Access)**：
- 所有核访问所有内存的延迟相同
- 通过共享总线或交叉开关连接
- 可扩展性受限（总线带宽瓶颈）

**NUMA (Non-Uniform Memory Access)**：
- 本地内存访问快，远程内存访问慢
- 典型比例：本地 100ns vs 远程 300ns（3:1 或更高）
- 需 NUMA-aware 调度优化

### 缓存一致性基础
- 多核共享内存的核心挑战：**缓存一致性（Cache Coherence）**
- 若不处理：Core 0 写 X=1，Core 1 仍可能读到 X=0
- 解决方案：Snooping 协议（下讲详述）

### SMP 可扩展性

**Amdahl 定律多核版**：
\[
S = \frac{1}{(1-F) + F/P}
\]
- \(F\)：可并行的比例
- \(P\)：处理器数量
- 例：\(F=0.9, P=32\) → \(S=1/(0.1+0.9/32) \approx 7.8\times\)

**Gustafson 定律**：
\[
S = P - \alpha(P-1)
\]
- 问题规模随处理器数增长时更准确

### 多核架构关键指标
- **核间通信延迟**：Core-to-core latency (ns)
- **共享 Cache 层级**：L2/L3 共享 vs 私有
- **互连带宽**：Crossbar / Ring / Mesh 的 bisection bandwidth

## 关键计算

**NUMA 平均延迟**：
\[
T_{avg} = \frac{L_{local} \times N_{local} + L_{remote} \times N_{remote}}{N_{total}}
\]

## 阅读任务

1. 精读 §5.1：多处理器分类学
2. 精读 §5.2：集中式共享内存架构的挑战
3. 精读 §5.3：分布式共享内存架构的性能特性

## 思考题

1. 为什么总线带宽是 SMP 可扩展性的主要瓶颈？
2. NUMA 系统上，OS 调度器为何需要感知 NUMA 拓扑？
3. 多核时代为何单线程性能增长放缓？
