# 第21周 自测题：GPU 体系结构

> 闭卷完成，时间 30 分钟。Occupancy 与带宽计算为主。

## 题目

### Q1：Warp 数量计算
一个 kernel launch 配置为 `<<<dim3(16,8), dim3(32,4)>>>`。请问总共启动了多少个 warp？

### Q2：Occupancy 计算
某 SM 最多支持 2048 线程、64 warp、32 block。一个 kernel 使用的 Block 大小为 256 线程，每线程 32 个寄存器（SM 共 65536 寄存器），不使用共享内存。求该 SM 上能驻留的最大 Block 数和 Occupancy。

### Q3：寄存器限制的 Occupancy
与 Q2 相同条件，但每线程使用 64 个寄存器。此时 Occupancy 如何变化？

### Q4：共享内存限制的 Occupancy
Block 大小 256，每线程 32 寄存器，但每 Block 使用 32 KB 共享内存（SM 有 64 KB 共享内存）。求最大驻留 Block 数和 Occupancy。

### Q5：全局内存带宽计算
GPU 全局内存带宽为 900 GB/s。处理 256×256 单精度矩阵乘法（C=A×B），Naive 实现每个输出元素需要从全局内存读取 2N 个元素。若 kernel 计算时间为 0（完全带宽瓶颈），理论最短执行时间是多少？

### Q6：Warp Divergence 分析
以下 CUDA 代码中，哪些 warp 会经历 divergence？
```cuda
__global__ void check(int *data) {
    int tid = threadIdx.x;
    if (tid < 16) {
        data[tid] = tid * 2;
    } else {
        data[tid] = tid;
    }
}
```

### Q7：延迟隐藏所需线程数
某 GPU 全局内存延迟约 400 cycle，每 SM 每周期最多执行 32 个 warp 指令。要保持内存流水线充分忙碌，每个 SM 最少需要多少活跃 warp？

### Q8：Grid/Block 维度
需要处理 1000×800 的图像（每个像素一个线程）。设计 Grid 和 Block 维度（2D），使线程总数最少浪费且 Block 不超过 1024 线程。

---

## 答案

### A1
Block 总数 = 16 × 8 = 128 blocks。
每 Block 线程数 = 32 × 4 = 128 threads。
总线程数 = 128 × 128 = 16384。
Warp = 16384 / 32 = **512 warps**。

### A2
- 线程限制：2048 / 256 = 8 blocks
- 寄存器限制：65536 / (256 × 32) = 65536 / 8192 = 8 blocks
- 硬件限制：32 blocks
- 有效：min(8, 8, 32) = **8 blocks**，2048 threads
- Occupancy = 2048/2048 = **100%**

### A3
寄存器限制：65536 / (256 × 64) = 65536 / 16384 = **4 blocks**
有效：min(8, 4, 32) = 4 blocks，1024 threads
Occupancy = 1024/2048 = **50%**。寄存器加倍 → 占用率减半。

### A4
- 线程限制：8 blocks
- 寄存器限制：8 blocks（与 Q2 相同）
- 共享内存限制：64 KB / 32 KB = **2 blocks**
- 有效：min(8, 8, 2, 32) = **2 blocks**，512 threads
- Occupancy = 512/2048 = **25%**

### A5
每次输出元素读取 2N = 512 个 float = 2048 bytes。
但有 N² = 65536 个元素。总数据量 ≈ 65536 × 512 × 4 = 134 MB（近似）。
更精确：C 读/写 = 0.25 MB，A 读 256×256×4 = 0.25 MB (但如果缓存不好每次重读则不同)...
Naive：Inner loop 读 B[:,j] = 256 floats × 4 = 1 KB/输出。每个输出读整个 B 列和 A 行：
总读 = N² × 2N × 4 = 256² × 512 × 4 = 134217728 bytes ≈ **128 MB**。
最短时间 = 128 MB / 900 GB/s = 128/921600 s ≈ **0.139 ms**（实际因计算开销会更长）。

### A6
同一个 warp 内（32 线程），前 16 线程走 `if` 分支 (`tid<16`)，后 16 线程走 `else` 分支。
**所有 warp 都会经历 divergence**。如果 Block 包含多个 warp（如 Block=256 共 8 warp）：
- Warp 0：tid 0-31 → 前 16 走 if，后 16 走 else → **divergence**
- Warp 1-7：tid 32-255 → 全部走 else → **无 divergence**

### A7
需隐藏 400 cycle 延迟。每 cycle 需要 1 warp 发射。
最少活跃 warp = 400 / 1 = **400 warps**。
这是理论下限；实际还需考虑依赖链等因素。

### A8
1000×800 = 800,000 线程。
设 Block = (32, 16) = 512 线程（2 的幂，方便）。
Grid = ceil(1000/32) × ceil(800/16) = 32 × 50 = 1600 blocks。
总线程 = 1600 × 512 = 819,200（浪费约 2.3%）。
更优：Block=(25, 32)=800，Grid=(40, 25) → 恰好 1000×800=800,000 线程，但 Block 不是 32 的倍数可能影响 warp 效率。实际推荐 **Block=(32,32)=1024, Grid=(32,25)**。
