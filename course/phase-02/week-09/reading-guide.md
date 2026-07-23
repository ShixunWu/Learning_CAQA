# Week 9：存储系统扩展 + Phase 2 项目

> **阅读范围**：Appendix D（RAID、SSD、存储可靠性）
> **本周是 Phase 2 的综合实践周**

---

## 阅读目标

1. 理解 RAID 0/1/5/6 的可靠性模型
2. 了解 SSD 的FTL（Flash Translation Layer）和写放大
3. 能用 CACTI 对 Cache 进行面积/功耗建模

## 核心概念

### RAID 级别

| 级别 | 描述 | 空间效率 | 容错能力 |
|------|------|----------|----------|
| RAID 0 | 条带化，无冗余 | 100% | 0 盘 |
| RAID 1 | 镜像 | 50% | 1 盘 |
| RAID 5 | 分布式奇偶校验 | (N-1)/N | 1 盘 |
| RAID 6 | 双奇偶校验 | (N-2)/N | 2 盘 |

### SSD 写放大

由于 Flash 不能原地覆写，写入时需擦除整个块（通常MB级）再写。即使应用层只写 4KB，实际物理写入可能远大于 4KB → **写放大因子（WAF）**。

---

## Phase 2 项目：存储层次优化方案

用 CACTI 对以下 Cache 配置建模，输出面积/延迟/功耗：

```bash
# CACTI 示例配置
./cacti -infile cache.cfg
```

编写一份 **1 页报告**，分析：
1. 给定面积 budget（如 10mm²），如何分配 L1/L2/L3 Cache 大小最优？
2. 如果功耗限制是 5W，你的方案是否需要调整？
3. 从 AMAT 角度论证你的选择

---

## Phase 2 综合测验

详见 [phase-review.md](../phase-review.md)

## 检查点

- [ ] 能设计和分析多级 Cache 层次
- [ ] 能手算 AMAT 并比较不同配置
- [ ] 理解 TLB 和大页的作用
- [ ] 了解存储系统可靠性的基本概念
