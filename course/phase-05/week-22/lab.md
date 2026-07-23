# 第22周 实验：GPU 矩阵乘法优化

> 从 Naive → Tiled Shared Memory → Warp Shuffle，逐步优化至接近 cuBLAS

## 实验环境

```bash
nvcc -O3 -arch=sm_80 -o matmul_opt matmul_opt.cu
```

## 实验 1：Naive 基线（作为对照）

```cuda
// 复用第 21 周的 naive 实现
__global__ void sgemm_naive(const float *A, const float *B, float *C, int N) {
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;
    if (row < N && col < N) {
        float sum = 0.0f;
        for (int k = 0; k < N; k++)
            sum += A[row * N + k] * B[k * N + col];
        C[row * N + col] = sum;
    }
}
```

## 实验 2：共享内存 Tiling

```cuda
#define TILE 32
__global__ void sgemm_tiled(const float *A, const float *B, float *C, int N) {
    __shared__ float As[TILE][TILE];
    __shared__ float Bs[TILE][TILE];

    int row = blockIdx.y * TILE + threadIdx.y;
    int col = blockIdx.x * TILE + threadIdx.x;
    float sum = 0.0f;

    for (int t = 0; t < (N + TILE - 1) / TILE; t++) {
        // 合作加载 tile
        if (row < N && (t * TILE + threadIdx.x) < N)
            As[threadIdx.y][threadIdx.x] = A[row * N + t * TILE + threadIdx.x];
        else
            As[threadIdx.y][threadIdx.x] = 0.0f;

        if (col < N && (t * TILE + threadIdx.y) < N)
            Bs[threadIdx.y][threadIdx.x] = B[(t * TILE + threadIdx.y) * N + col];
        else
            Bs[threadIdx.y][threadIdx.x] = 0.0f;

        __syncthreads();

        for (int k = 0; k < TILE; k++)
            sum += As[threadIdx.y][k] * Bs[k][threadIdx.x];

        __syncthreads();
    }
    if (row < N && col < N)
        C[row * N + col] = sum;
}
```

## 实验 3：Warp Shuffle 归约

```cuda
// warp-level 向量点积归约 (用于优化内层循环的一部分)
__inline__ __device__ float warp_reduce_sum(float val) {
    for (int offset = 16; offset > 0; offset /= 2)
        val += __shfl_down_sync(0xffffffff, val, offset);
    return val;
}

__global__ void sgemm_warp_tiled(const float *A, const float *B, float *C, int N) {
    __shared__ float As[TILE][TILE];
    __shared__ float Bs[TILE][TILE];

    int row = blockIdx.y * TILE + threadIdx.y;
    int col = blockIdx.x * TILE + threadIdx.x;
    float sum = 0.0f;

    for (int t = 0; t < N / TILE; t++) {
        As[threadIdx.y][threadIdx.x] = A[row * N + t * TILE + threadIdx.x];
        Bs[threadIdx.y][threadIdx.x] = B[(t * TILE + threadIdx.y) * N + col];
        __syncthreads();

        for (int k = 0; k < TILE; k++)
            sum += As[threadIdx.y][k] * Bs[k][threadIdx.x];

        __syncthreads();
    }
    if (row < N && col < N)
        C[row * N + col] = sum;
}
```

## 实验 4：Roofline 分析工具

```python
#!/usr/bin/env python3
"""GPU Roofline 模型绘制"""
import matplotlib.pyplot as plt
import numpy as np

# GPU 参数 (A100)
peak_gflops = 19500   # FP32: 19.5 TFLOPS
peak_bw = 1555         # GB/s (HBM2e)

ai = np.logspace(-2, 3, 1000)  # FLOP/Byte
roofline = np.minimum(peak_gflops, peak_bw * ai)

# 标记不同 kernel 的算术强度
kernels = {
    "vecadd":  (0.083, 1400),    # 2 ops / (3*8 bytes) = 0.083
    "naive_mm":(16.0, 200),
    "tiled_mm":(32.0, 5000),
    "cublas":  (128.0, 16000),
}

fig, ax = plt.subplots(figsize=(8, 6))
ax.loglog(ai, roofline, 'k-', linewidth=2, label='Roofline')
for name, (x, y) in kernels.items():
    ax.loglog(x, y, 'o', markersize=10, label=name)
ax.set_xlabel('Arithmetic Intensity (FLOP/Byte)')
ax.set_ylabel('Performance (GFLOPS)')
ax.legend()
ax.set_title('GPU Roofline Model')
plt.savefig('roofline.png')
print("Roofline 图已保存到 roofline.png")
```

## 实验报告要求

1. 对比 Naive / Tiled / Warp 三个版本的 GFLOPS（N=1024, 2048, 4096）
2. 分析 Tiled 版本中 Bank Conflict 是否存在（TILE=32 → bank=32 → 访问 `As[ty][k]` 时列索引 `k` 在 warp 内变化，可能冲突）
3. 绘制 Roofline 图，标出各版本位置，分析所处的瓶颈区域
