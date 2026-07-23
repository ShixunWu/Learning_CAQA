# Week 17：VLIW 与 EPIC

> **阅读范围**：Appendix H
> **核心问题**：把 ILP 挖掘从硬件转移到编译器，能带来什么优势？

---

## 阅读目标

1. 理解 VLIW（Very Long Instruction Word）的基本思想
2. 掌握编译器静态调度的优势和局限
3. 了解 EPIC（Explicitly Parallel Instruction Computing）/IA-64 的设计理念
4. 理解 VLIW 的代码膨胀和二进制兼容性问题

---

## 核心概念

### VLIW 基本原理

传统超标量：硬件在运行时动态发现 ILP → 复杂的发射逻辑
VLIW：编译器在编译时发现 ILP → 打包成一条长指令，硬件简单执行

一条 VLIW 指令包含多个操作槽（slot），每个槽对应一个 FU：

```
| Op1: ADD  | Op2: MUL  | Op3: LD   | Op4: NOP  |
    Int ALU    FP MUL      Mem Unit     (空槽)
```

### VLIW 的优势

- 硬件极简化：无需动态调度、无需乱序执行、无需重命名
- 确定性延迟：编译器知道每条指令的确切延迟
- 功耗低：硬件复杂度低 → 功耗优势

### VLIW 的挑战

1. **代码膨胀**：NOP 填充未使用的 slot → 代码体积增大
2. **二进制兼容性**：不同 FU 配置需要重新编译（Intel IA-64 试图用 EPIC 解决）
3. **编译复杂度**：需要全局调度、软件流水、循环展开
4. **不可预测延迟**：Cache miss 导致 stall → 所有 slot 都被阻塞

### EPIC / IA-64 (Itanium)

- 指令打包成 128-bit bundle（3 条 41-bit 指令 + 5-bit 模板）
- 使用**推测加载**（speculative load）和**数据推测**减少 memory stall
- 谓词执行（predication）：消除分支
- 最终失败原因：编译器难以充分挖掘 ILP，性能不及超标量

### VLIW vs 超标量

| 特性 | VLIW | 超标量 |
|------|------|--------|
| ILP 发掘 | 编译器（静态）| 硬件（动态）|
| 硬件复杂度 | 低 | 高 |
| 功耗 | 低 | 高 |
| 代码兼容性 | 差 | 好 |
| 对不可预测延迟 | 敏感 | 可容忍 |

---

## 动手实验

详见 [lab.md](lab.md)

## 自测

详见 [self-test.md](self-test.md)
