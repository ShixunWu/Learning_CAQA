# Week 34 实验：顶点项目可行性评估

## 实验目标

根据你选择的顶点项目方向，运行基线实验、收集初始数据，
完成可行性评估报告。

## 通用准备步骤

所有项目都需要完成以下环境准备：

```bash
# 创建项目工作目录
mkdir -p ~/capstone-project/
cd ~/capstone-project/

# 初始化 Git 版本控制
git init
echo "# Capstone Project" > README.md
git add README.md
git commit -m "Initial commit"
```

## 项目 1：gem5 RISC-V CPU — 基线实验

```python
"""
gem5 RISC-V 基线性能收集脚本 (Python wrapper)
假设已安装 gem5 和 RISC-V 工具链
"""
import subprocess
import os
import json

GEM5_PATH = os.environ.get('GEM5_PATH', '/opt/gem5')
CONFIG = f"{GEM5_PATH}/configs/example/se.py"

def run_gem5_simulation(cpu_type: str, benchmark: str, 
                         issue_width: int = 4) -> dict:
    """运行 gem5 SE 模式仿真并收集统计信息"""
    
    cmd = [
        "python3", CONFIG,
        "--cpu-type", cpu_type,        # MinorCPU / O3CPU
        "--cmd", benchmark,
        "--caches",
        "--l1d_size", "32kB",
        "--l1i_size", "32kB",
        "--l2_size", "256kB",
        f"--issue-width={issue_width}",
    ]
    
    print(f"运行: {' '.join(cmd)}")
    
    # 模拟运行（实际环境中取消注释）
    # result = subprocess.run(cmd, capture_output=True, text=True)
    # 解析 stats.txt
    
    return {
        "cpu_type": cpu_type,
        "benchmark": benchmark,
        "issue_width": issue_width,
        "sim_seconds": 0.0,      # 从 stats.txt 解析
        "cpi": 0.0,
        "ipc": 0.0,
        "branch_mispred_rate": 0.0,
    }


# 基线扫描：变化发射宽度
print("=" * 50)
print("gem5 RISC-V 超标量基线扫描")
print("=" * 50)

configs = [
    ("MinorCPU", 1, "基线 - 顺序单发射"),
    ("O3CPU", 2, "超标量 2-way"),
    ("O3CPU", 4, "超标量 4-way"),
    ("O3CPU", 8, "超标量 8-way"),
]

for cpu, width, desc in configs:
    stats = run_gem5_simulation(cpu, "dhrystone", width)
    print(f"{desc}: IPC={stats['ipc']:.3f}, CPI={stats['cpi']:.3f}")
```

## 项目 2：缓存优化 — 基线实验

```python
"""
缓存性能分析基线脚本
使用 pycachesim 或自定义缓存模拟器
"""
from collections import OrderedDict
import random

class CacheSimulator:
    """简单的缓存模拟器（用于基线分析）"""
    
    def __init__(self, size_kb: int, assoc: int, block_size: int = 64):
        self.block_size = block_size
        num_blocks = size_kb * 1024 // block_size
        self.num_sets = num_blocks // assoc
        self.assoc = assoc
        # 每组一个 LRU 有序字典
        self.sets = [OrderedDict() for _ in range(self.num_sets)]
        self.hits = 0
        self.misses = 0
    
    def access(self, address: int):
        block_addr = address // self.block_size
        set_idx = block_addr % self.num_sets
        tag = block_addr // self.num_sets
        
        cache_set = self.sets[set_idx]
        
        if tag in cache_set:
            # 命中：更新 LRU
            cache_set.move_to_end(tag)
            self.hits += 1
        else:
            # 未命中
            self.misses += 1
            if len(cache_set) >= self.assoc:
                # 驱逐 LRU
                cache_set.popitem(last=False)
            cache_set[tag] = True
    
    def get_stats(self):
        total = self.hits + self.misses
        return {
            "hits": self.hits,
            "misses": self.misses,
            "hit_rate": self.hits / total if total > 0 else 0,
            "miss_rate": self.misses / total if total > 0 else 0,
        }


# 运行基线测试
if __name__ == "__main__":
    print("=" * 50)
    print("缓存参数扫描基线")
    print("=" * 50)
    
    for size in [16, 32, 64, 128, 256]:  # KB
        for assoc in [1, 2, 4, 8]:
            cache = CacheSimulator(size_kb=size, assoc=assoc)
            
            # 模拟顺序访问 + 随机访问混合
            for i in range(10000):
                if random.random() < 0.7:
                    addr = i * 64  # 顺序
                else:
                    addr = random.randint(0, 1024*1024)  # 随机
                cache.access(addr)
            
            stats = cache.get_stats()
            print(f"L1 {size}KB {assoc}-way: "
                  f"Miss Rate = {stats['miss_rate']:.4f}")
```

## 项目 3：GPU Kernel — 基线实验

```python
"""
GPU Kernel 优化基线 - 朴素矩阵乘法
"""
import numpy as np
import time

# 以下为 CUDA kernel 的 Python 伪代码/NumPy 基线
def baseline_matmul_cpu(A: np.ndarray, B: np.ndarray, N: int) -> float:
    """CPU 基线矩阵乘法（用于对比）"""
    start = time.perf_counter()
    C = np.zeros((N, N))
    for i in range(N):
        for j in range(N):
            for k in range(N):
                C[i, j] += A[i, k] * B[k, j]
    elapsed = time.perf_counter() - start
    gflops = 2 * N**3 / elapsed / 1e9
    return gflops

# CUDA Kernel 伪代码（实际需用 PyCUDA 或直接写 .cu）
CUDA_KERNEL_NAIVE = """
__global__ void matmul_naive(float* A, float* B, float* C, int N) {
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
"""

if __name__ == "__main__":
    N = 1024
    A = np.random.randn(N, N).astype(np.float32)
    B = np.random.randn(N, N).astype(np.float32)
    
    # CPU 基线
    cpu_gflops = baseline_matmul_cpu(A, B, 256)  # 先用小矩阵测
    print(f"CPU Naive (N=256): {cpu_gflops:.2f} GFLOPS")
    
    # NumPy (BLAS) 对比
    start = time.perf_counter()
    C_np = A @ B
    np_time = time.perf_counter() - start
    np_gflops = 2 * N**3 / np_time / 1e9
    print(f"NumPy/BLAS (N={N}): {np_gflops:.2f} GFLOPS")
```

## 项目 4 & 5：数据收集框架

对于 NoC 和 DNN 加速器项目，基线实验通常基于 Python 建模：

```python
# 通用设计空间探索框架
from itertools import product

def design_space_explorer(param_grid: dict, simulator_fn):
    """穷举参数空间，收集性能/能耗数据"""
    keys = list(param_grid.keys())
    values = list(param_grid.values())
    results = []
    
    for combo in product(*values):
        params = dict(zip(keys, combo))
        result = simulator_fn(**params)
        result['params'] = params
        results.append(result)
    
    return results

# 示例：DNN 加速器参数空间
param_grid = {
    "array_size": [16, 32, 64, 128],
    "sram_kb": [64, 128, 256, 512],
    "dataflow": ["WS", "OS", "RS"],
    "frequency_mhz": [200, 500, 700, 1000],
}

print(f"设计空间大小: ", end="")
size = 1
for v in param_grid.values():
    size *= len(v)
print(f"{size} 个配置点")
```

## 交付物清单

1. **可行性评估报告**（1-2 页）：包含工具链验证结果、基线数据、风险分析
2. **项目提案**（按 reading-guide 模板）
3. **Git 仓库**：包含所有基线代码和结果
