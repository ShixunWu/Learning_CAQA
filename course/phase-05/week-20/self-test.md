# 第20周 自测题：SIMD 指令集

> 闭卷完成，时间 30 分钟。定量计算为主。

## 题目

### Q1：SIMD 宽度与元素个数
某 SSE 寄存器宽度为 128-bit。若处理的数据类型分别为 (a) float (32-bit), (b) double (64-bit), (c) int16_t，每个寄存器可打包多少个元素？

### Q2：对齐分析
一个 `float A[1000]` 数组的起始地址为 `0x7fffe018`。判断该地址对 SSE（16-byte）和 AVX（32-byte）是否对齐。

### Q3：非对齐加载惩罚
对齐的 `_mm_load_ps` 延迟为 4 周期，非对齐的 `_mm_loadu_ps` 延迟为 8 周期。循环体中有 2 次 load + 1 次 store，已向量化（4 元素/寄存器）。处理 n=10000 个 float，求对齐 vs 非对齐的总 load/store 周期差。

### Q4：SIMD 理论加速比
某处理器每个周期可发射 2 个 256-bit AVX FMA 指令，频率 3 GHz。计算其单精度（32-bit）峰值 GFLOPS。

### Q5：向量化比例分析
如下循环是否可以 SIMD 向量化？说明原因：
```c
for (int i = 0; i < N; i++) {
    a[i] = a[i-1] + b[i];  // 循环 1
}
for (int i = 0; i < N; i++) {
    a[i] = b[i] * c[i];    // 循环 2
}
```

### Q6：SIMD 加速比（Amdahl 视角）
程序 95% 在 SIMD 路径上（256-bit AVX），理论上 8× 加速。若 SIMD 路径实际仅获 5× 加速（因内存带宽限制），求整体加速比。

### Q7：寄存器压力
AVX2 有 16 个 YMM 寄存器。矩阵乘法内层循环需要多少寄存器（A 行片段、B 列片段、C 累加器各若干）？是否会溢出到栈？

### Q8：多版本兼容
若须同一二进制同时支持 SSE-only 和 AVX2 机器，应该采用什么技术？写出一个 CPUID 检测伪代码。

---

## 答案

### A1
(a) 128/32 = **4 个 float**
(b) 128/64 = **2 个 double**
(c) 128/16 = **8 个 int16_t**

### A2
0x7fffe018 mod 16 = 0x18 mod 16 = 24 mod 16 = 8 → 不对齐（SSE 对齐加载 `_mm_load_ps` 要求地址 16 字节对齐，即 mod 16 = 0）。
0x7fffe018 mod 32 = 24 mod 32 = 24 → 不对齐（AVX 对齐加载 `_mm256_load_ps` 要求地址 32 字节对齐，即 mod 32 = 0）。
> 对齐要求：SSE=16 字节边界，AVX=32 字节边界。若不对齐，需使用 `_mm_loadu_ps` / `_mm256_loadu_ps`，带来 2-3× 延迟惩罚，且跨 cache line 时额外增加总线事务。

### A3
load 次数 = ceil(10000/4) × 2 = 2500 × 2 = 5000 次
store 次数 = 2500 次
对齐总延迟 = 5000 × 4 + 2500 × 4 = 30000 周期
非对齐总延迟 = 5000 × 8 + 2500 × 4 = 50000 周期
差值 = **20000 周期**。

### A4
每周期：2 × (8 floats × 2 ops) = 32 FLOP/cycle（FMA=乘+加=2 ops）
峰值 = 3 GHz × 32 = **96 GFLOPS**。

### A5
循环 1：**不可向量化**，因为 `a[i]` 依赖 `a[i-1]`（循环携带的真依赖，RAW）。
循环 2：**可向量化**，各迭代独立，无依赖。

### A6
\( S = \frac{1}{(1-0.95) + 0.95/5} = \frac{1}{0.05 + 0.19} = \frac{1}{0.24} \approx \mathbf{4.17\times} \)。

### A7
最少需要：2 个 A 寄存器（broadcast 可复用一个）、1 个 B 寄存器、1 个 C 累加器，约 4 个。实际展开后可能用 8-12 个，不会溢出 16 个的限制。

### A8
使用 **函数多版本（Function Multi-versioning）** 或动态分发：
```c
if (__builtin_cpu_supports("avx2"))
    kernel = kernel_avx2;
else if (__builtin_cpu_supports("sse2"))
    kernel = kernel_sse;
else
    kernel = kernel_scalar;
```
