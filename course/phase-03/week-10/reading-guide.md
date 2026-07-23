# Week 10：5级流水线

> **阅读范围**：Appendix C §C.1-C.2
> **核心问题**：流水线如何在不改变单条指令延迟的情况下提高吞吐率？

---

## 阅读目标

1. 理解经典 RISC 5 级流水线：IF/ID/EX/MEM/WB
2. 能用流水线时空图分析指令执行
3. 理解流水线的理想加速比

## 核心概念

### 5 级流水线

| 阶段 | 全称 | 做什么 |
|------|------|--------|
| IF | Instruction Fetch | 从 I-Cache 取指令，PC+4 |
| ID | Instruction Decode | 译码，读寄存器 |
| EX | Execute | ALU 运算，计算地址 |
| MEM | Memory Access | 访存（仅 load/store） |
| WB | Write Back | 结果写回寄存器 |

### 流水线加速比

理想情况下（n 条指令，k 级流水线）：
\[
\text{Speedup} = \frac{n \cdot k}{k + n - 1} \xrightarrow{n \to \infty} k
\]

所以 5 级流水线最大理想加速比是 5x。

> 为什么实际达不到 5x？因为流水线冒险（hazards）会引入气泡（bubbles）。

### 流水线寄存器

每两级之间有一个**流水线寄存器**，存储该指令在该阶段的中间结果。这些寄存器引入了 Clock-to-Q + Setup 开销，是流水线深度的物理极限。

## 动手实验

详见 [lab.md](lab.md)

## 自测

详见 [self-test.md](self-test.md)
