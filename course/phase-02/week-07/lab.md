# Week 7 实验：gem5 Cache 层次探索

## 任务：在 gem5 中配置多级 Cache

### 基础配置

```bash
# 进入 gem5 目录
cd $GEM5_HOME

# 运行一个 benchmark（如 matrix multiply）
# 带 L1 Cache
build/RISCV/gem5.opt configs/example/se.py \
  --cmd=/path/to/matmul \
  --caches \
  --l1d_size=32kB --l1i_size=32kB \
  --l1d_assoc=4 --l1i_assoc=2

# 带 L1 + L2 Cache
build/RISCV/gem5.opt configs/example/se.py \
  --cmd=/path/to/matmul \
  --caches --l2cache \
  --l1d_size=32kB --l1i_size=32kB \
  --l2_size=256kB --l2_assoc=8
```

### 参数扫描实验

创建一个 Python 脚本来批量运行 gem5 并收集结果：

```python
import subprocess, re

def run_gem5(l1d_size, l2_size):
    cmd = f"build/RISCV/gem5.opt configs/example/se.py \
      --cmd=matmul --caches --l2cache \
      --l1d_size={l1d_size}kB --l2_size={l2_size}kB"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    # 从 m5out/stats.txt 提取缺失率
    with open("m5out/stats.txt") as f:
        for line in f:
            if "system.cpu.dcache.overall_miss_rate" in line:
                miss_rate = float(line.split()[1])
    return miss_rate

# 扫描不同配置
for l1d in [16, 32, 64]:
    for l2 in [128, 256, 512]:
        mr = run_gem5(l1d, l2)
        print(f"L1d={l1d}KB L2={l2}KB → Miss Rate: {mr:.3f}")
```

### 任务

1. 对比只有 L1 vs L1+L2 的性能差异
2. 画一张 L1/L2 大小 vs 总执行时间的热力图

---

## 交付物

1. gem5 配置脚本
2. 热力图
3. 一句话：L2 在什么情况下价值最大？
