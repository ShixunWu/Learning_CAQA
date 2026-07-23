# 第21周 阅读指南：GPU 体系结构

> 对应教材：CAQA 6th Edition, Chapter 4 §4.6-4.8

## 学习目标

1. 理解 SIMT（Single Instruction Multiple Thread）与 SIMD 的本质区别
2. 掌握 GPU 线程层次：Thread → Warp/Wavefront → Block → Grid
3. 理解 GPU 内存层次结构及其对性能的决定性影响

## 核心概念

### SIMT vs SIMD
| 特性 | SIMD (CPU) | SIMT (GPU) |
|------|-----------|-------------|
| 并行单元 | 固定宽度向量寄存器 | 线程束 (Warp=32) |
| 分支处理 | 需手动 mask | 硬件自动处理（但影响效率） |
| 线程上下文 | 无 | 大量线程可切换隐藏延迟 |
| 编程模型 | Intrinsics | CUDA/OpenCL 标量编程 |

### GPU 线程层次
- **Thread**：执行单元，有独立 PC 和寄存器
- **Warp (NVIDIA)** / **Wavefront (AMD)**：32/64 线程的调度单元，锁步执行
- **Thread Block**：同一 SM 内协作线程组，可同步(`__syncthreads`)
- **Grid**：Kernel 启动的所有 Block 的集合

### Streaming Multiprocessor (SM)
- NVIDIA A100：108 SM，每 SM 64 FP32 CUDA Core + 4 Tensor Core
- 每 SM 的共享资源：寄存器文件（65536 × 32-bit）、共享内存（最大 164 KB）、L1 Cache
- Warp Scheduler：每 SM 4 个，每个可调度 1 warp/cycle

### GPU 内存层次
| 内存类型 | 延迟 | 带宽 | 作用域 | 大小 |
|---------|------|------|--------|------|
| 寄存器 (Reg) | ~0 cycle | — | Thread | ~255/thread |
| 共享内存 (SMEM) | ~20 cycle | ~1.5 TB/s | Block | 164 KB/SM |
| L1 Cache | ~30 cycle | ~1 TB/s | SM | 128 KB/SM |
| L2 Cache | ~200 cycle | ~3 TB/s | Device | 40 MB |
| 全局内存 (HBM2e) | ~400 cycle | ~2 TB/s | Device | 40/80 GB |

### GPU 延迟隐藏
- GPU 不依赖大缓存减少延迟，而依赖大量线程切换来隐藏延迟
- 关键公式：需要的线程数 = 内存延迟 × 吞吐量
- 例如：400 cycle 延迟 × 32 ops/cycle = 12800 线程在飞行中

## 关键计算

**Occupancy（占用率）**：
\[
\text{Occupancy} = \frac{\text{Active Warps/SM}}{\text{Max Warps/SM}}
\]
受限于：寄存器使用、共享内存使用、Block 大小。

## 阅读任务

1. 精读 §4.6：理解 SIMT 执行模型与 warp 调度
2. 精读 §4.7：GPU 内存层次与数据移动策略
3. 精读 §4.8：Fermi → Volta → Ampere 的架构演进

## 思考题

1. SIMT 的 warp divergence 如何影响性能？如何避免？
2. 为什么 GPU 选择小缓存 + 大吞吐而非 CPU 的大缓存策略？
3. 一个 SM 上同时驻留多个 Block 有什么好处？
