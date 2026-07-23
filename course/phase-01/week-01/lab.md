# Week 1 实验：性能对比器

> **目标**：用 Python 可视化处理器性能公式和 Amdahl 定律

---

## 实验 1：Iron Law 性能对比

### 任务

编写一个 Python 程序，输入两个处理器的参数，对比它们的执行时间。

### 代码模板

```python
import matplotlib.pyplot as plt
import numpy as np

def cpu_time(ic, cpi, clock_rate_ghz):
    """计算 CPU 执行时间（秒）
    ic: 指令数
    cpi: 每条指令平均周期数
    clock_rate_ghz: 时钟频率 (GHz)
    """
    return ic * cpi / (clock_rate_ghz * 1e9)

# 两个处理器的参数
proc_a = {"ic": 1e9, "cpi": 1.0, "freq": 3.0, "name": "Processor A"}
proc_b = {"ic": 1.2e9, "cpi": 0.8, "freq": 2.5, "name": "Processor B"}

time_a = cpu_time(proc_a["ic"], proc_a["cpi"], proc_a["freq"])
time_b = cpu_time(proc_b["ic"], proc_b["cpi"], proc_b["freq"])

print(f"{proc_a['name']}: {time_a:.3f}s")
print(f"{proc_b['name']}: {time_b:.3f}s")
print(f"Speedup A vs B: {time_b/time_a:.2f}x")
```

### 思考

1. 哪个处理器更快？为什么？
2. 如果 Processor B 的编译器优化使得 IC 减少 10%，它能反超吗？

---

## 实验 2：Amdahl 定律可视化

### 任务

画出 Amdahl 定律曲线：横轴是加速比 S，不同曲线对应不同的 f 值。

### 代码模板

```python
def amdahl_speedup(f, S):
    """计算 Amdahl 定律的总体加速比"""
    return 1 / ((1 - f) + f / S)

# 参数
S_values = np.linspace(1, 1000, 1000)  # 加速比从1到1000
f_values = [0.5, 0.7, 0.9, 0.95, 0.99]

plt.figure(figsize=(10, 6))
for f in f_values:
    speedups = amdahl_speedup(f, S_values)
    plt.plot(S_values, speedups, label=f'f={f}')

# 标记渐近线
for f in [0.5, 0.9, 0.99]:
    plt.axhline(y=1/(1-f), color='gray', linestyle='--', alpha=0.3)
    plt.text(1000, 1/(1-f), f'limit={1/(1-f):.1f}x', fontsize=8)

plt.xscale('log')
plt.xlabel('优化部分的加速比 S')
plt.ylabel('总体加速比')
plt.title("Amdahl's Law")
plt.legend()
plt.grid(True, alpha=0.3)
plt.savefig('amdahl_law.png', dpi=150)
plt.show()

# 关键结论打印
print("\n=== Amdahl 定律关键结论 ===")
for f in f_values:
    limit = 1 / (1 - f)
    print(f"f={f:.2f} (优化占比 {f*100:.0f}%) → 最大理论加速比 = {limit:.1f}x")
```

### 观察

1. f=0.5 时，即使优化部分无限快，整体也只能快 2 倍
2. f=0.99 时，理论上限是 100 倍 —— 但你需要优化 99% 的代码才能做到

---

## 实验 3（选做）：用 Spike 测量指令数

```bash
# 用 Spike 的 --ic 选项统计指令数
spike --ic=1000000:10000000 pk program_name

# 或用 gem5 获取详细统计
build/RISCV/gem5.opt configs/example/se.py -c program_name

# 在输出 m5out/stats.txt 中找到：
# sim_insts: 执行的指令总数
# sim_ticks: 总周期数
# 可计算 IPC = sim_insts / sim_ticks
```

---

## 交付物

1. 运行实验 1 和实验 2 的代码
2. 一张 Amdahl 定律曲线图
3. 回答：为什么 "make the common case fast" 是体系结构设计的金科玉律？
