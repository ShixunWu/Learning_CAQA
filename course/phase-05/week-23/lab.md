# 第23周 阶段项目：向量/SIMD/GPU 矩阵乘法综合对比

> 综合 Phase 5 所学，从性能、功耗、代码复杂度三个维度对比三种数据并行方案

## 项目目标

在矩阵乘法 \(C = A \times B\)（N×N 单精度）上，实现并对比：
1. **RISC-V 向量**（Spike 模拟，RVV）
2. **x86 AVX2 SIMD**（硬件实测）
3. **CUDA GPU**（硬件实测）

## Part 1：代码框架

### 向量版本（RVV — Spike 模拟）
```bash
# vec_matmul.S — 伪代码示意，实际用 RVV intrinsics
# 内层: 加载 A 的行向量 → B 的每一列做点积
# 使用 vfmacc (fused multiply-accumulate) 向量指令
riscv64-unknown-elf-gcc -O2 -march=rv64gcv matmul_vec.c -o matmul_vec
spike pk matmul_vec 512  # N=512
```

### AVX2 版本（x86 — 硬件实测）
```c
// matmul_avx2.c — 复用第 20 周的 tiled 实现
// 编译: gcc -O3 -mavx2 -mfma matmul_avx2.c -o matmul_avx2
```

### CUDA 版本（GPU — 硬件实测）
```c
// matmul_cuda.cu — 复用第 22 周的 tiled 实现
// 编译: nvcc -O3 -arch=sm_80 matmul_cuda.cu -o matmul_cuda
```

## Part 2：性能数据采集脚本

```python
#!/usr/bin/env python3
"""综合对比三种方案的矩阵乘法性能"""
import subprocess, json, time, os

SIZES = [128, 256, 512, 1024, 2048]
CONFIGS = {
    "RVV":    {"cmd": ["spike", "pk", "./matmul_vec"], "timeout": 300},
    "AVX2":   {"cmd": ["./matmul_avx2"], "timeout": 60},
    "CUDA":   {"cmd": ["./matmul_cuda"], "timeout": 30},
}

results = {}
for name, cfg in CONFIGS.items():
    if not os.path.exists(cfg["cmd"][0]):
        print(f"跳过 {name}: 可执行文件不存在")
        continue
    results[name] = {}
    for n in SIZES:
        # 运行并获取时间和功耗（RAPL 或 nvidia-smi）
        cmd = cfg["cmd"] + [str(n)]
        t1 = time.perf_counter()
        p = subprocess.run(cmd, capture_output=True, text=True, 
                           timeout=cfg["timeout"])
        t2 = time.perf_counter()
        gflops = 2.0 * n**3 / (t2 - t1) / 1e9  # O(N³)/time
        results[name][n] = {
            "time_s": t2 - t1,
            "gflops": gflops
        }

# 打印对比表
print(f"{'N':>6}", end="")
for name in results:
    print(f"  {name:>12}", end="")
print()
for n in SIZES:
    print(f"{n:>6}", end="")
    for name in results:
        g = results[name][n]["gflops"]
        print(f"  {g:12.2f}", end="")
    print()

# 保存 JSON
with open("phase5_results.json", "w") as f:
    json.dump(results, f, indent=2)
```

## Part 3：代码复杂度分析

```python
#!/usr/bin/env python3
"""统计各实现的代码复杂度"""
import subprocess

files = {
    "RVV": "matmul_vec.c",
    "AVX2": "matmul_avx2.c",
    "CUDA": "matmul_cuda.cu",
}

for name, f in files.items():
    lines = len(open(f).readlines())
    intrinsics = subprocess.check_output(
        f"grep -c '_mm\\|__sync\\|vle32' {f}", shell=True, text=True
    ).strip()
    print(f"{name}: {lines} 行, ~{intrinsics} 条特殊指令")
```

## Part 4：能耗估算

```python
#!/usr/bin/env python3
"""估算能耗效率"""
# 假设各平台 TDP:
tdp = {"RVV": 15, "AVX2": 65, "CUDA": 300}  # Watts

import json
with open("phase5_results.json") as f:
    results = json.load(f)

for n in [512, 1024, 2048]:
    print(f"\nN={n}:")
    for name, tdp_w in tdp.items():
        if name in results:
            t = results[name][str(n)]["time_s"]
            energy_j = tdp_w * t  # Joules
            efficiency = results[name][str(n)]["gflops"] / tdp_w  # GFLOPS/W
            print(f"  {name}: {energy_j:.1f}J, {efficiency:.1f} GFLOPS/W")
```

## 实验报告要求

1. 绘制三种方案的 GFLOPS vs N 曲线（一张图）
2. 绘制 GFLOPS/W（能效）对比柱状图
3. 用表格列出：代码行数、使用的特殊指令/API 数、调试难度评分
4. 回答：在 N=1024 时，哪个方案综合最优？为什么？
