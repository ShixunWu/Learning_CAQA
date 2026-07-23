# Week 2 实验：基准测试批判

---

## 实验 1：几何平均 vs 算术平均

### 任务

三个程序在两个处理器上的执行时间如下：

| 程序 | 处理器 A | 处理器 B | 加速比(B/A) |
|------|----------|----------|-------------|
| P1 | 10s | 5s | 2.0 |
| P2 | 20s | 10s | 2.0 |
| P3 | 30s | 400s | 0.075 |

```python
import numpy as np

speedups = np.array([2.0, 2.0, 0.075])

arith_mean = np.mean(speedups)
geo_mean = np.exp(np.mean(np.log(speedups)))

print(f"算术平均: {arith_mean:.3f}")
print(f"几何平均: {geo_mean:.3f}")
```

### 思考

1. 算术平均报告 B 比 A 快 1.36x。你相信吗？
2. 几何平均说 B 只有 A 的 0.67x。哪个更合理？
3. 如果反过来算 A/B 的加速比，算术平均会变吗？几何平均呢？

---

## 实验 2：MIPS 陷阱

### 任务

同一个 C 程序（数组求和），分别用 RISC-V 和 x86 编译，统计指令数和执行时间：

| 指标 | RISC-V | x86 |
|------|--------|-----|
| 指令数 | 1000 | 600 |
| CPI | 1.0 | 1.5 |
| 频率 | 2 GHz | 2 GHz |

计算：
1. 两者各自的 MIPS
2. 两者各自的实际执行时间
3. 哪个更快？

### 结果

```python
# RISC-V
time_rv = 1000 * 1.0 / 2e9  # = 500 ns
mips_rv = 1000 / (time_rv * 1e6)  # = 2000 MIPS

# x86
time_x86 = 600 * 1.5 / 2e9  # = 450 ns
mips_x86 = 600 / (time_x86 * 1e6)  # = 1333 MIPS

print(f"RISC-V: {time_rv*1e9:.0f}ns, {mips_rv:.0f} MIPS")
print(f"x86: {time_x86*1e9:.0f}ns, {mips_x86:.0f} MIPS")
print(f"x86 is {time_rv/time_x86:.2f}x faster, but has LOWER MIPS!")
```

**结论**：MIPS 高 ≠ 性能好。这就是为什么用 MIPS 衡量性能是"毫无意义的"。

---

## 实验 3（选做）：在 Spike 上运行 CoreMark

```bash
# 编译 CoreMark for RISC-V
git clone https://github.com/eembc/coremark.git
cd coremark
make CC=riscv64-unknown-elf-gcc PORT_DIR=linux64

# 用 Spike 运行
spike pk coremark.exe
```

### 交付物

1. 几何平均 vs 算术平均的计算和结论
2. MIPS 陷阱的数值证明
3. 一句话总结：用什么指标替代 MIPS？
