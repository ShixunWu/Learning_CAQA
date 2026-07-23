# 第22周 自测题：GPU 性能优化

> 闭卷完成，时间 30 分钟。合并访问、Bank Conflict、Roofline 计算为主。

## 题目

### Q1：合并访问判断
一个 warp（32 线程）执行 `float x = A[threadIdx.x * N]`，其中 N=64。假设 `A` 起始地址对齐到 128 字节。每次访问间隔 N×4 = 256 字节。请问这是合并访问吗？需要多少次内存事务（假设事务粒度为 32 字节）？

### Q2：Bank Conflict 分析
共享内存 `__shared__ float sm[32][32]`。warp 内线程 `tid` 访问 `sm[tid][0]`。是否有 bank conflict？如果是访问 `sm[0][tid]` 呢？

### Q3：填充消除 Bank Conflict
要使 `__shared__ float sm[32][32]` 的列访问 `sm[tid][c]` 无 bank conflict（c 为常数），应将第二维填充到多少？写出填充后的声明。

### Q4：Roofline 瓶颈判断
GPU：峰值 15 TFLOPS，带宽 1.2 TB/s。Kernel A 算术强度 0.5 FLOP/Byte，Kernel B 算术强度 60 FLOP/Byte。判断各自的性能上限和瓶颈类型。

### Q5：Tiling 的算术强度
矩阵乘法 C=A×B，维度 N=1024，每线程计算 1 个 C 元素。Naive：每输出读 2N 个元素（2×1024×4=8192 bytes）。Tiled（TILE=32）：A 和 B 各 N/TILE 次从全局内存加载 tile，每次 tile 大小 TILE²×4。计算 Tiled 版本的算术强度。

### Q6：非合并访问带宽损失
理论上合并访问时 32 线程只需 1 次 128B 事务。若 strided 访问导致每线程 1 次独立的事务（32B 事务中仅 4B 有效），带宽有效利用率是多少？

### Q7：Warp Shuffle vs Shared Memory 延迟
Warp shuffle 延迟 ~1 cycle，共享内存延迟 ~20 cycle。一个 warp 归约需要 5 次数据交换。仅考虑这 5 次交换的延迟差异（不计 ALU），从共享内存版本切换到 shuffle 版本能节省多少 cycle？

### Q8：Double Buffering 效果
Tiled 矩阵乘法使用双缓冲后，异步拷贝（`cp.async`）可使全局内存加载与计算完全重叠。若全局加载 tile 需 40 cycle，计算 tile 需 80 cycle，不重叠时总时间 120 cycle/tile。完全重叠后，每 tile 时间变为多少？加速比是多少？

---

## 答案

### A1
线程 0 访问 `A[0]`，线程 1 访问 `A[64]`，线程 i 访问 `A[i×64]`。间隔 = 256 字节。
32 字节事务粒度 → 每线程需要独立的事务。
需要 **32 次事务**，带宽利用率 = 4/32 ≈ **12.5%**，是严重的非合并访问。

### A2
- `sm[tid][0]`：线程 `tid` 访问 `sm[tid][0]`。Bank = `tid×32+0` 的 bank = `tid×4+0` mod 32 → 连续线程访问连续 bank → **无 bank conflict**。
- `sm[0][tid]`：线程 `tid` 访问 `sm[0][tid]`。Bank = `0×32+tid` mod 32 = `tid` mod 32 → 所有线程访问同一 bank → **32-way bank conflict**（广播除外）。

### A3
填充 1 列使其不是 32 的倍数：`__shared__ float sm[32][33]`。
此时 `sm[tid][c]` 的 bank = `tid×33+c` mod 32，由于 33 mod 32 = 1，bank = `tid+c` mod 32，连续线程错开 1 bank → **无 conflict**。

### A4
Kernel A：上限 = min(15000, 1200×0.5) = min(15000, 600) = **600 GFLOPS**，**内存带宽瓶颈**。
Kernel B：上限 = min(15000, 1200×60) = min(15000, 72000) → **15000 GFLOPS = 15 TFLOPS**，**计算瓶颈**。

### A5
Naive：每个输出 2N 次全局读取 × 4 bytes = 8N bytes = 8192 bytes/输出。
Tiled（TILE=32, N=1024）：全局读取 A 总量 = (N/TILE)³ × TILE² × 4 = 32³ × 1024 × 4 = 134,217,728 bytes（A 和 B 各一份，共 268,435,456 bytes）。写 C = N²×4 = 4,194,304 bytes。总字节 = 272,629,760。
总 FLOP = 2N³ = 2,147,483,648。
算术强度 = 2,147,483,648 / 272,629,760 ≈ **7.9 FLOP/Byte**。
（对比 naive：2N³ / (2N×N²×4 + N²×4) ≈ 2N³ / (8N³ + 4N²) ≈ 0.25 FLOP/Byte。Tiled 相比 naive 内存效率提升约 30 倍。）

### A6
每次 32B 事务中有效负载 4B（只有一个 float 被使用）。
有效带宽利用率 = 4/32 = **12.5%**。实际有效带宽 = 900 GB/s × 0.125 = 112.5 GB/s。

### A7
共享内存：5 × 20 = 100 cycles
Warp shuffle：5 × 1 = 5 cycles
节省 = 100 - 5 = **95 cycles**（纯数据移动延迟）。

### A8
重叠后 = max(40, 80) = **80 cycle/tile**（加载被计算完全隐藏）。
加速比 = 120/80 = **1.5×**。
