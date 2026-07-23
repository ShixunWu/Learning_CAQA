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

### 5. 可靠性与可用性（Ch1 §1.7）

大规模系统中，可靠性是重要的设计约束：

- **MTTF（Mean Time To Failure）**：平均无故障时间，衡量硬件可靠性
- **MTTR（Mean Time To Repair）**：平均修复时间，取决于运维能力
- **MTBF（Mean Time Between Failures）**：MTBF = MTTF + MTTR，平均故障间隔
- **可用性（Availability）**：\[ \text{Availability} = \frac{\text{MTTF}}{\text{MTTF} + \text{MTTR}} \]

例如：服务器 MTTF=25 年，MTTR=1 小时，可用性 = 219000/(219000+1) ≈ 99.9995%。但对 50,000 台服务器的 WSC，预期每天约 5-6 台故障——故障是常态而非例外。这为后续 Phase 7 的 WSC 故障处理策略奠定基础。

---

## 动手实验

详见 [lab.md](lab.md)

---

## 自测

详见 [self-test.md](self-test.md)
