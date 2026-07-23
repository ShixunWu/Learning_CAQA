# 第20周 实验：x86 SIMD 矩阵乘法

> 使用 SSE 和 AVX Intrinsics 实现矩阵乘法，对比性能

## 实验环境

```bash
# Linux (GCC)
sudo apt install g++

# 编译命令
g++ -O2 -msse2 -o matmul_sse matmul_sse.cpp
g++ -O2 -mavx2 -o matmul_avx matmul_avx.cpp
```

## 实验 1：标量基线矩阵乘法

```cpp
// matmul_scalar.cpp
void matmul_scalar(float *C, const float *A, const float *B, int N) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            float sum = 0.0f;
            for (int k = 0; k < N; k++) {
                sum += A[i * N + k] * B[k * N + j];
            }
            C[i * N + j] = sum;
        }
    }
}
```

## 实验 2：SSE 向量化矩阵乘法

```cpp
// matmul_sse.cpp
#include <xmmintrin.h>  // SSE
void matmul_sse(float *C, const float *A, const float *B, int N) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            __m128 sum = _mm_setzero_ps();
            for (int k = 0; k < N; k += 4) {
                // 加载 A[i][k..k+3] (4 floats)
                __m128 a = _mm_loadu_ps(&A[i * N + k]);
                // 加载 B[k..k+3][j] — 需要 gather 或用转置矩阵
                __m128 b = _mm_set_ps(B[(k+3)*N+j], B[(k+2)*N+j],
                                      B[(k+1)*N+j], B[k*N+j]);
                sum = _mm_add_ps(sum, _mm_mul_ps(a, b));
            }
            // 水平求和
            sum = _mm_hadd_ps(sum, sum);
            sum = _mm_hadd_ps(sum, sum);
            _mm_store_ss(&C[i * N + j], sum);
        }
    }
}
```

## 实验 3：AVX2 向量化（8 路并行）

```cpp
// matmul_avx.cpp
#include <immintrin.h>  // AVX2
void matmul_avx(float *C, const float *A, const float *B, int N) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j += 8) {
            __m256 c = _mm256_loadu_ps(&C[i * N + j]);
            for (int k = 0; k < N; k++) {
                __m256 a = _mm256_set1_ps(A[i * N + k]);
                __m256 b = _mm256_loadu_ps(&B[k * N + j]);
                c = _mm256_fmadd_ps(a, b, c);  // FMA: c = a*b + c
            }
            _mm256_storeu_ps(&C[i * N + j], c);
        }
    }
}
```

## 实验 4：性能测量框架

```python
#!/usr/bin/env python3
"""运行并分析 SIMD 矩阵乘法性能"""
import subprocess, time, sys

sizes = [64, 128, 256, 512, 1024]
versions = [
    ("标量", "./matmul_scalar"),
    ("SSE",  "./matmul_sse"),
    ("AVX2", "./matmul_avx"),
]

print(f"{'N':>6}", end="")
for name, _ in versions:
    print(f"  {name:>10}", end="")
print(f"  {'SSE加速比':>10}  {'AVX加速比':>10}")

for n in sizes:
    print(f"{n:>6}", end="", flush=True)
    times = {}
    for name, exe in versions:
        t1 = time.perf_counter()
        subprocess.run([exe, str(n)], capture_output=True)
        t2 = time.perf_counter()
        times[name] = t2 - t1
        print(f"  {times[name]:10.4f}", end="")
    sse_speedup = times["标量"] / times["SSE"]
    avx_speedup = times["标量"] / times["AVX2"]
    print(f"  {sse_speedup:10.2f}  {avx_speedup:10.2f}")
```

## 实验报告要求

1. 绘制各版本的 GFLOPS 曲线，与理论峰值对比
2. 分析为什么实际加速比达不到 4× (SSE) 或 8× (AVX)
3. 讨论矩阵转置对 B 矩阵访问模式的影响（列访问 vs 行访问）
