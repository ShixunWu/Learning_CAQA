# Week 35 阅读指南：实验方法论与性能分析

> 阶段：Capstone Project — 实现与调优 | 核心主题：实验方法论、统计严谨性

## 学习目标

- 掌握计算机架构实验的基本方法论
- 理解统计显著性检验在性能评估中的应用
- 学会使用 Roofline 模型和瓶颈分析方法
- 掌握性能数据的可视化最佳实践

## 核心概念

### 1. 实验方法论

**科学实验流程**：
```
假设 → 实验设计 → 数据收集 → 分析 → 结论 → 迭代
```

**关键原则**：
- **可控变量**：每次只改变一个参数，保持其他恒定
- **可复现性**：记录所有环境信息（编译器版本、flags、硬件配置）
- **统计显著性**：多次运行取均值，报告标准差

### 2. 性能指标选择

| 项目类型 | 主要指标 | 次要指标 |
|---------|---------|---------|
| CPU 微架构 | IPC, Branch Mispred Rate | 功耗, 面积估算 |
| 缓存系统 | Miss Rate, AMAT | 带宽利用率, 预取准确率 |
| GPU Kernel | GFLOPS, 带宽利用率 | Occupancy, 分支发散率 |
| NoC | 延迟, 吞吐量 | 功耗, 面积, 跳数 |
| DNN 加速器 | TOPS/W, MAC 利用率 | 延迟, DRAM 访问量 |

### 3. Roofline 模型

Roofline 模型是性能分析的利器：

\[
\text{Attainable Performance} = \min\left(\text{Peak FLOPs}, \text{Peak Bandwidth} \times \text{Arithmetic Intensity}\right)
\]

- **计算受限区**（右侧）：性能受限于计算峰值
- **内存受限区**（左侧）：性能受限于内存带宽

```python
def plot_roofline(peak_flops, peak_bw, kernel_data):
    """绘制 Roofline 图"""
    ai = np.logspace(-1, 4, 100)  # Arithmetic Intensity range
    compute_roof = np.minimum(peak_flops, peak_bw * ai)
    # kernel_data: list of (ai, achieved_gflops, label)
```

### 4. 统计严谨性

**基本统计量**：
- **均值（Mean）**：\(\bar{x} = \frac{1}{n}\sum x_i\)
- **标准差（Std Dev）**：\(s = \sqrt{\frac{1}{n-1}\sum (x_i - \bar{x})^2}\)
- **95% 置信区间**：\(\bar{x} \pm 1.96 \times \frac{s}{\sqrt{n}}\)

**常见陷阱**：
- ❌ 只跑一次就报结果
- ❌ 冷启动 + 热启动数据混用
- ❌ 忽略系统噪声（其他进程干扰）
- ✅ 至少跑 5 次，取中位数/均值
- ✅ 使用 `perf stat -r 5` 或等效方法
- ✅ 记录并报告变异系数（CV = s / mean）

### 5. 可视化最佳实践

| 图表类型 | 适用场景 | 示例 |
|---------|---------|------|
| 条形图 | 多配置对比 | 不同缓存大小的 Miss Rate |
| 折线图 | 参数扫描趋势 | 发射宽度 vs IPC |
| Roofline | 瓶颈诊断 | Kernel 在 Roofline 上的位置 |
| 热力图 | 2D 参数空间 | 缓存大小 × 关联度 → Miss Rate |
| 散点图 | Pareto 前沿 | 能效 vs 性能取舍 |

## 实施检查清单

- [ ] 环境配置已记录（OS, compiler, flags, hardware）
- [ ] 所有脚本已纳入版本控制
- [ ] 实验可一键复现（Makefile 或 run_all.sh）
- [ ] 原始数据以结构化格式保存（CSV/JSON，而非截图）
- [ ] 每个实验至少重复 3 次
- [ ] 异常值已检查并记录原因

## 阅读建议

1. 重点理解 Roofline 模型的构建和应用
2. 参考文献：Jain, "The Art of Computer Systems Performance Analysis"
3. 实践：用项目数据画一幅 Roofline 图，标注你的设计点
