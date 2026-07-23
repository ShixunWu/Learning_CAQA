# 第19周 实验：RISC-V 向量扩展（RVV）编程

> 使用 Spike 模拟器进行 RISC-V Vector 汇编编程

## 实验环境

```bash
# 安装 RISC-V 工具链与 Spike
sudo apt install gcc-riscv64-unknown-elf spike  # Linux
# 或从源码编译: https://github.com/riscv-software-src/riscv-isa-sim
```

## 实验 1：向量加法（RVV 汇编）

编写 `vadd.s`，实现 C[i] = A[i] + B[i]，长度 n=128。

```asm
# vadd.s — RISC-V 向量加法 (VLEN=256, SEW=32 → 每寄存器 8 个元素)
    .section .text
    .globl _start
_start:
    li t0, 128           # n = 128
    vsetvli t1, t0, e32  # VL = min(VLMAX, n), SEW=32
    vle32.v v0, (a0)     # 加载 A
    vle32.v v1, (a1)     # 加载 B
    vadd.vv v2, v0, v1   # v2 = v0 + v1
    vse32.v v2, (a2)     # 存回 C
    sub t0, t0, t1       # n -= VL
    bnez t0, _start      # 处理剩余元素
    ret
```

## 实验 2：向量点积（Dot Product）

编写 `vdot.s`，计算 A·B = Σ(A[i] × B[i])。

```asm
# vdot.s — 带归约的向量点积
    .section .text
    .globl vdot
vdot:
    vsetvli t1, a3, e32  # VL=min(VLMAX, n)
    vle32.v v0, (a0)     # v0 = A
    vle32.v v1, (a1)     # v1 = B
    vmul.vv v2, v0, v1   # v2 = A * B (逐元素)
    vredsum.vs v3, v2, v0 # 归约求和 → v3[0]
    vmv.x.s a0, v3       # 结果移到标量寄存器
    ret
```

## 实验 3：标量 vs 向量性能对比

```bash
#!/bin/bash
# benchmark.sh — 对比标量与向量 DAXPY 性能

# 标量版本编译
riscv64-unknown-elf-gcc -O2 -march=rv64gc daxpy_scalar.c -o daxpy_scalar

# 向量版本编译 (RVV)
riscv64-unknown-elf-gcc -O2 -march=rv64gcv daxpy_vector.c -o daxpy_vector

# 用 Spike 运行并计指令数
echo "=== 标量版本 ==="
spike --ic=1000000000:1000000000 pk daxpy_scalar

echo "=== 向量版本 ==="
spike --ic=1000000000:1000000000 pk daxpy_vector
```

## 实验 4：用 Python 模拟向量执行时间

```python
#!/usr/bin/env python3
"""模拟向量处理器 DAXPY (Y = a*X + Y) 执行时间"""

def vector_exec_time(n, mvl=64, startup=10, per_element=1):
    """
    n: 向量长度
    mvl: 最大向量长度
    startup: 启动开销（周期）
    per_element: 每元素处理时间（周期）
    """
    import math
    strips = math.ceil(n / mvl)
    # DAXPY: 2 loads + 1 mul + 1 add + 1 store = 多个 convoy
    n_convoys = 3  # LV + MULVS + ADDVV/SV（假设部分链接）
    total = strips * (startup * n_convoys + mvl * per_element * n_convoys)
    total -= (strips * mvl - n) * per_element * n_convoys  # 修正最后一轮
    return int(total)

# 测试不同长度
for n in [64, 128, 256, 512, 1024]:
    vec_t = vector_exec_time(n)
    scalar_t = n * 7  # 标量：每次迭代 ~7 周期
    print(f"n={n:5d}: 向量={vec_t:6d}cyc, 标量={scalar_t:6d}cyc, 加速比={scalar_t/vec_t:.2f}x")
```

## 实验报告要求

1. 记录不同向量长度下 vadd 的指令数，与标量版本对比
2. 分析 startup 开销在短向量时的影响
3. 讨论 RVV 的 `vsetvli` 指令如何支持可变向量长度
