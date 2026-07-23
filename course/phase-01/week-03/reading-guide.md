# Week 3：指令集体系结构全景

> **阅读范围**：Appendix A + Appendix K（课本内附录A + 在线附录K）
> **核心问题**：ISA 设计决策如何影响性能、功耗和可实现性？

---

## 阅读目标

1. 对比 RISC vs CISC 的哲学差异
2. 了解 RISC-V 的 ISA 设计原则
3. 理解 ISA 分类（Stack/Accumulator/Register-Memory/Register-Register）
4. 掌握寻址模式对指令数和 CPI 的影响

---

## 核心概念

### ISA 分类

| 类型 | 代表 | ALU 指令操作数来源 | 特点 |
|------|------|--------------------|------|
| Stack | x87 FP | 栈顶 | 代码密度最高，但并行度最差 |
| Accumulator | 早期 CPU | 累加器+内存 | 简单，但内存流量大 |
| Register-Memory | x86 | 寄存器+内存 | 一条指令可访存，编码复杂 |
| Register-Register | RISC-V, ARM | 寄存器+寄存器 | Load/Store分离，规整 |

### RISC 设计原则（附录 K）

1. **简单指令**：每条指令做一件事，周期数可预测
2. **Load/Store 架构**：只有 load/store 访存
3. **大量寄存器**：32个通用寄存器（RISC-V）或更多
4. **固定长度编码**：RISC-V 基础指令 32-bit，简化译码
5. **少量寻址模式**：降低硬件复杂度

### RISC-V 特色

- **模块化 ISA**：RV32I（基础整数）为必须，M/A/F/D/C 为可选扩展
- **压缩指令（C扩展）**：16-bit 指令，提升代码密度
- **开放标准**：无专利费，学术界和工业界共同推动

---

## 动手实验

详见 [lab.md](lab.md)

---

## 自测

详见 [self-test.md](self-test.md)
