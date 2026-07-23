# Week 2：基准测试与可信度

> **阅读范围**：Ch1 §1.5-1.9
> **核心问题**：如何判断性能数字是否可信？

---

## 阅读目标

1. 理解 SPEC 基准测试套件的设计原则
2. 识别常见的基准测试陷阱
3. 能用 SPECratio 比较不同处理器的性能
4. 理解 MIPS/MFLOPS 为什么是糟糕的性能指标

---

## 核心概念速览

### 1. 基准测试的层级

| 层级 | 代表 | 优缺点 |
|------|------|--------|
| 真实应用 | SPEC CPU2017 | 最可信，但运行慢 |
| 修改过的应用 | 去掉I/O的核心代码 | 较好，有时失真 |
| 内核程序 | Livermore Loops, Linpack | 小，容易优化过度 |
| 玩具程序 | quicksort, sieve | 完全不可信 |
| 合成基准 | Dhrystone, Whetstone | 历史上被严重滥用 |

### 2. SPEC 是如何组织基准测试的

- **SPECrate**：测量吞吐率（多个程序副本同时运行）
- **SPECspeed**：测量单任务执行时间
- **基准**：以参考机器（如 Sun UltraSPARC）为 1.0
- 最终分数：所有子测试分数的**几何平均**

> **为什么是几何平均而不是算术平均？**
> 因为性能比是比率数据。几何平均不受参考机器选择的影响。

### 3. MIPS 为什么是"Meaningless Indicator of Processor Speed"

\[
\text{MIPS} = \frac{\text{IC}}{\text{Time} \times 10^6} = \frac{\text{Clock Rate}}{\text{CPI} \times 10^6}
\]

**问题**：不同 ISA 中，同样的操作可能需要不同数量的指令。RISC 指令多，CISC 指令少 → RISC 的 MIPS 反而可能更高？但这不意味着更快！

### 4. 能效比

\[
\text{Energy Efficiency} = \frac{\text{Performance}}{\text{Power}}
\]

现代处理器的核心约束不是晶体管数量，而是功耗墙。

---

## 动手实验

详见 [lab.md](lab.md)

---

## 自测

详见 [self-test.md](self-test.md)
