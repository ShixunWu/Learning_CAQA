# Week 16 实验：gem5 多发射与 SMT 实验

## 实验目标

使用 gem5 模拟器配置不同的发射宽度和 SMT 线程数，测量 IPC 变化。

---

## 环境准备

```bash
# 安装 gem5（如未安装）
git clone https://github.com/gem5/gem5.git
cd gem5
scons build/RISCV/gem5.opt -j$(nproc)
```

---

## 实验步骤

### Step 1：基准测试（单发射）

用 gem5 SE 模式运行一个矩阵乘法 benchmark：

```bash
build/RISCV/gem5.opt configs/example/se.py \
  --cpu-type=DerivO3CPU \
  --caches --l1d_size=64kB --l1i_size=32kB \
  --cmd=tests/test-progs/matrix_multiply/riscv-matrix-multiply
```

记录 IPC（指令/周期）。

### Step 2：改变发射宽度

修改 CPU 参数测试 1/2/4/8 发射：

```python
# 在 configs/example/se.py 或在命令行指定
# --cpu-type=DerivO3CPU
# 修改 O3CPU 参数：
#   dispatchWidth=2, issueWidth=2, commitWidth=2
```

| 发射宽度 | IPC | 相对加速比 |
|----------|-----|-----------|
| 1        | ?   | 1.0x      |
| 2        | ?   | ?         |
| 4        | ?   | ?         |
| 8        | ?   | ?         |

### Step 3：SMT 实验

```python
# 使用 2 条线程，4 发射
# 修改 --cpu-type=DerivO3CPU --smt=2
```

对比：2 线程 SMT 的 IPC  vs  2× 单线程 IPC。

### Step 4：分析瓶颈

- 为什么 8 发射相比 4 发射提升有限？
- SMT 在哪些场景收益最大？（计算密集型 vs 访存密集型）

---

## 交付物

1. 发射宽度-IPC 变化表及曲线
2. SMT 效果分析
3. gem5 stats.txt 关键字段截图（system.cpu.ipc）
