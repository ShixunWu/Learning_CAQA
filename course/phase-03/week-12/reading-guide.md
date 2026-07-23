# Week 12：分支预测基础

> **阅读范围**：Appendix C §C.5
> **核心问题**：如何猜测分支方向，减少控制冒险带来的流水线气泡？

---

## 阅读目标

1. 理解控制冒险的代价：分支 penalties = 分支频率 × 误预测率 × 误预测惩罚
2. 掌握静态预测（always taken / BTFNT）与动态预测的区别
3. 理解 1-bit 和 2-bit 预测器的状态机
4. 了解 BTB 和相关预测器（correlating predictor）的工作原理

---

## 核心概念

### 控制冒险代价

\[
\text{Branch Penalty} = \text{Branch Frequency} \times \text{Misprediction Rate} \times \text{Misprediction Penalty}
\]

假设 20% 分支，10% 误预测，2-cycle 惩罚 → 额外 CPI = 0.2×0.1×2 = 0.04

### 静态预测

| 策略 | 规则 | 准确率 |
|------|------|--------|
| Always Not-Taken | 继续取 PC+4 | ~40-60% |
| Always Taken | 总是跳转 | ~40-60% |
| BTFNT | 向后分支 Taken（循环），向前 Not-Taken（if-else）| ~65-85% |

### 动态预测：1-bit vs 2-bit

**1-bit 预测器**：记住上次方向，下次预测相同
- 问题：循环退出时连续两次误预测（最后一次迭代 + 下次进入）

**2-bit 预测器**（4 状态饱和计数器）：

```
        T            T            T
Strong N ──→ Weak N ──→ Weak T ──→ Strong T
    ↑          │          ↑          │
    └──NT──────┘          └──NT──────┘
```

- 需要连续两次误预测才改变方向 → 循环退出只有 1 次误预测

### BTB（Branch Target Buffer）

缓存分支目标地址，在 IF 阶段就根据 PC 查到目标地址。命中时：1-cycle 分支；不命中时：需要计算目标。

### 相关预测器（Correlating / Two-Level）

用**全局分支历史**（最近 m 个分支的 T/NT 模式）索引 PHT（Pattern History Table），捕获分支之间的相关性。

例如：(m,n) 预测器 = m 位全局历史，2^m 个 n-bit 计数器。

---

## 动手实验

详见 [lab.md](lab.md)

## 自测

详见 [self-test.md](self-test.md)
