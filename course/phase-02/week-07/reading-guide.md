# Week 7：Cache 性能优化（下）

> **阅读范围**：Ch2 §2.4-2.6
> **核心问题**：如何通过预取、多级Cache、非阻塞Cache进一步提升性能？

---

## 阅读目标

1. 理解硬件/软件预取的工作原理和风险
2. 掌握多级 Cache 的 AMAT 计算公式
3. 理解非阻塞 Cache（MSHR）如何容忍缺失延迟
4. 能用 gem5 配置多级 Cache 并分析缺失率

## 核心概念

### 多级 Cache 的 AMAT

\[
\text{AMAT} = t_{L1} + m_{L1} \times (t_{L2} + m_{L2} \times t_{mem})
\]

其中 \(m_{L1}\) 是 L1 缺失率，\(m_{L2}\) 是 L2 **局部**缺失率。

> 注意：\(m_{L2}\) 是 L2 的局部缺失率（L2 miss / L1 miss），不是全局缺失率（L2 miss / 所有访存）。

### 预取（Prefetching）

- **硬件预取**：检测访问模式后自动预取（如顺序预取、stride预取）
- **软件预取**：编译器插入 prefetch 指令
- **风险**：预取无用数据→污染Cache、浪费带宽

### 非阻塞 Cache

- **MSHR**（Miss Status Holding Register）：记录正在处理的缺失请求
- 允许在 miss 期间继续处理后续 hit → 容忍延迟
- 关键指标：MSHR 条目数决定可同时容忍多少个未完成的 miss

## 动手实验

详见 [lab.md](lab.md)

## 自测

详见 [self-test.md](self-test.md)
