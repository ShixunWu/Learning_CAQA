# 第21周 实验：CUDA 编程基础

> 使用 CUDA C 实现向量加法与矩阵乘法，探索 Block 大小对性能的影响

## 实验环境

```bash
# 确认有 NVIDIA GPU 和 CUDA 工具包
nvidia-smi
nvcc --version

# 编译
nvcc -O3 -o vecadd vecadd.cu
nvcc -O3 -o matmul matmul.cu
```

## 实验 1：CUDA 向量加法

```cuda
// vecadd.cu
#include <stdio.h>
#include <cuda_runtime.h>

__global__ void vecadd(const float *A, const float *B, float *C, int N) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < N) {
        C[idx] = A[idx] + B[idx];
    }
}

int main() {
    int N = 1 << 24;  // 16M elements
    size_t size = N * sizeof(float);
    float *d_A, *d_B, *d_C;
    cudaMalloc(&d_A, size); cudaMalloc(&d_B, size); cudaMalloc(&d_C, size);

    // 不同 Block 大小对比
    int blockSizes[] = {32, 64, 128, 256, 512, 1024};
    for (int bs : blockSizes) {
        int gridSize = (N + bs - 1) / bs;
        cudaEvent_t start, stop;
        cudaEventCreate(&start); cudaEventCreate(&stop);

        cudaEventRecord(start);
        vecadd<<<gridSize, bs>>>(d_A, d_B, d_C, N);
        cudaEventRecord(stop);
        cudaEventSynchronize(stop);

        float ms = 0;
        cudaEventElapsedTime(&ms, start, stop);
        float bw = (3.0f * size) / (ms / 1000.0f) / 1e9;  // GB/s
        printf("Block=%4d  Time=%.3fms  BW=%.1fGB/s\n", bs, ms, bw);
    }
    cudaFree(d_A); cudaFree(d_B); cudaFree(d_C);
}
```

## 实验 2：CUDA 矩阵乘法（Naive）

```cuda
// matmul_naive.cu
__global__ void matmul_naive(const float *A, const float *B, float *C, int N) {
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;
    if (row < N && col < N) {
        float sum = 0.0f;
        for (int k = 0; k < N; k++) {
            sum += A[row * N + k] * B[k * N + col];
        }
        C[row * N + col] = sum;
    }
}
// main: dim3 block(16,16), grid((N+15)/16, (N+15)/16);
```

## 实验 3：Block 大小扫描脚本

```python
#!/usr/bin/env python3
"""自动扫描 CUDA 矩阵乘法的 Block 大小"""
import subprocess, re

results = []
for bx in [8, 16, 32]:
    for by in [8, 16, 32]:
        output = subprocess.check_output(
            ["./matmul", str(1024), str(bx), str(by)],
            text=True
        )
        # 解析 GFLOPS
        m = re.search(r"(\d+\.?\d*)\s*GFLOPS", output)
        if m:
            gflops = float(m.group(1))
            results.append((bx, by, bx*by, gflops))
            print(f"Block=({bx},{by}) Threads={bx*by:4d} => {gflops:8.2f} GFLOPS")

# 找最优
best = max(results, key=lambda x: x[3])
print(f"\n最佳: Block=({best[0]},{best[1]}) {best[3]:.2f} GFLOPS")
```

## 实验 4：Occupancy 计算器

```python
#!/usr/bin/env python3
"""GPU Occupancy 计算器"""

def compute_occupancy(block_size, regs_per_thread, shared_mem_per_block,
                      max_threads_sm=2048, max_regs_sm=65536,
                      max_smem_sm=49152, max_blocks_sm=32):
    """计算 SM 占用率"""
    # 受限于每个维度
    blocks_by_threads = max_threads_sm // block_size
    blocks_by_regs = max_regs_sm // (block_size * regs_per_thread)
    blocks_by_smem = (max_smem_sm // shared_mem_per_block) if shared_mem_per_block else 999
    blocks_by_hw = max_blocks_sm

    active_blocks = min(blocks_by_threads, blocks_by_regs, 
                        blocks_by_smem, blocks_by_hw)
    active_threads = active_blocks * block_size
    occupancy = active_threads / max_threads_sm * 100

    print(f"Block大小: {block_size}")
    print(f"  线程限制: {blocks_by_threads} blocks")
    print(f"  寄存器限制: {blocks_by_regs} blocks (每线程 {regs_per_thread} regs)")
    print(f"  共享内存限制: {blocks_by_smem} blocks ({shared_mem_per_block}B/block)")
    print(f"  硬件限制: {blocks_by_hw} blocks")
    print(f"  → {active_blocks} active blocks, {active_threads} threads")
    print(f"  → Occupancy: {occupancy:.1f}%")
    return occupancy

# 典型场景
print("=== 场景 A: 寄存器密集型 ===")
compute_occupancy(256, 128, 0)
print("\n=== 场景 B: 共享内存密集型 ===")
compute_occupancy(256, 32, 16384)
```

## 实验报告要求

1. 记录不同 Block 大小下 vecadd 的有效带宽，解释最优 Block 大小的原因
2. 对 matmul，分析 Occupancy 与有效 GFLOPS 的关系
3. 讨论 Naive 实现的 B 矩阵访问导致的非合并访问问题
