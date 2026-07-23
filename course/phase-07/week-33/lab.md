# Week 33 实验：计算机架构演化时间线可视化

## 实验目标

使用 matplotlib 创建计算机架构演化时间线图，展示从 1960 年代至今的关键架构
里程碑及其性能/功耗/晶体管数量的变化趋势。

## 实验背景

可视化架构演化有助于理解技术进步的非线性特征——性能增长并非匀速，
而是由若干关键转折点驱动。

## 任务：架构演化时间线

```python
"""
Computer Architecture Evolution Timeline
CAQA 6th Edition - Appendix L 配套可视化
"""
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import numpy as np

# 设置中文字体
plt.rcParams['font.sans-serif'] = ['SimHei', 'Microsoft YaHei', 'DejaVu Sans']
plt.rcParams['axes.unicode_minus'] = False

# ===== 架构里程碑数据 =====
milestones = [
    # (年份, 事件, 晶体管数, 频率MHz, 功耗W, 时代)
    (1964, "IBM System/360\nISA兼容性", 5e3, 0.001, 5, "大型机"),
    (1971, "Intel 4004\n首个微处理器", 2300, 0.74, 1, "微处理器"),
    (1978, "Intel 8086\nx86诞生", 29000, 5, 2.5, "微处理器"),
    (1985, "MIPS R2000\nRISC革命", 110000, 8, 3, "RISC"),
    (1987, "ARM2\n嵌入式RISC", 30000, 8, 1, "RISC"),
    (1993, "Intel Pentium\n超标量", 3.1e6, 60, 15, "超标量"),
    (1995, "Pentium Pro\n乱序执行", 5.5e6, 200, 35, "超标量"),
    (1999, "NVIDIA GeForce 256\nGPU", 23e6, 120, 30, "GPU"),
    (2001, "Intel Itanium\nEPIC/VLIW", 25e6, 800, 130, "VLIW"),
    (2005, "AMD Athlon 64 X2\n多核时代", 233e6, 2400, 110, "多核"),
    (2007, "iPhone (ARM)\n移动计算", 1e6, 412, 0.5, "移动"),
    (2011, "Intel Sandy Bridge\n集成GPU", 1.16e9, 3400, 95, "多核"),
    (2015, "Google TPU v1\nDSA推理", None, 700, 75, "DSA"),
    (2017, "NVIDIA V100\nTensor Core", 21.1e9, 1530, 300, "GPU"),
    (2020, "Apple M1\nARM桌面", 16e9, 3200, 15, "ARM桌面"),
    (2022, "NVIDIA H100\nHopper", 80e9, 1980, 700, "GPU/AI"),
]

# ===== 创建图表 =====
fig, axes = plt.subplots(2, 2, figsize=(16, 10))
fig.suptitle("计算机架构 60 年演化时间线 (1964-2022)", 
             fontsize=16, fontweight='bold')

# --- 图1: 晶体管数量趋势 ---
ax1 = axes[0, 0]
valid_transistors = [(y, t, e) for y, e, t, _, _, _ in milestones if t is not None]
years_t, transistors, events_t = zip(*valid_transistors)

ax1.semilogy(years_t, transistors, 'b-o', linewidth=2, markersize=8)
for y, t, e in valid_transistors:
    ax1.annotate(e.split('\n')[0], (y, t), textcoords="offset points",
                xytext=(0, 10), fontsize=7, ha='center',
                rotation=45)

# 摩尔定律参考线
year_range = np.linspace(1964, 2025, 100)
moore_ref = 5e3 * 2**((year_range - 1964) / 2)  # 每2年翻倍
ax1.semilogy(year_range, moore_ref, 'r--', linewidth=1, alpha=0.5, label='摩尔定律 (每2年翻倍)')

ax1.set_xlabel('年份')
ax1.set_ylabel('晶体管数量 (对数坐标)')
ax1.set_title('晶体管数量增长趋势')
ax1.legend(fontsize=8, loc='upper left')
ax1.grid(True, alpha=0.3)

# --- 图2: 时间线甘特图风格 ---
ax2 = axes[0, 1]
eras = ["大型机", "微处理器", "RISC", "超标量", "VLIW", "多核", "移动", "DSA", "GPU", "ARM桌面", "GPU/AI"]
era_y = {e: i for i, e in enumerate(eras)}

colors_map = {
    "大型机": "#3498db", "微处理器": "#2ecc71", "RISC": "#e74c3c",
    "超标量": "#9b59b6", "VLIW": "#f39c12", "多核": "#1abc9c",
    "移动": "#e67e22", "DSA": "#c0392b", "GPU": "#27ae60",
    "ARM桌面": "#8e44ad", "GPU/AI": "#16a085"
}

for y, event, _, _, _, era in milestones:
    color = colors_map.get(era, "gray")
    ax2.scatter(y, era_y[era], s=150, c=color, zorder=5, edgecolors='black', linewidth=0.5)
    ax2.annotate(event.split('\n')[0], (y, era_y[era]), 
                textcoords="offset points", xytext=(8, 4), fontsize=6.5,
                rotation=30, ha='left')

ax2.set_yticks(range(len(eras)))
ax2.set_yticklabels(eras, fontsize=9)
ax2.set_xlabel('年份')
ax2.set_title('各架构时代的关键里程碑')
ax2.set_xlim(1960, 2025)
ax2.grid(True, alpha=0.3, axis='x')

# --- 图3: 功耗趋势 ---
ax3 = axes[1, 0]
valid_power = [(y, e, p, w) for y, e, _, _, w, _ in milestones]
years_p, events_p, _, watts = zip(*valid_power)

ax3.plot(years_p, watts, 'r-s', linewidth=2, markersize=8)
# 标注
for y, e, _, w in valid_power:
    ax3.annotate(e.split('\n')[0], (y, w), textcoords="offset points",
                xytext=(0, 10), fontsize=7, ha='center', rotation=45)

ax3.axhline(y=100, color='gray', linestyle=':', alpha=0.5, label='100W 典型功耗上限')
ax3.set_xlabel('年份')
ax3.set_ylabel('功耗 (W)')
ax3.set_title('处理器功耗演进')
ax3.legend(fontsize=8)
ax3.grid(True, alpha=0.3)

# --- 图4: 性能/功耗比 (估算) ---
ax4 = axes[1, 1]
# 估算 SPEC 性能（以 MIPS R2000 = 1 为基准的相对性能）
spec_estimates = {
    "IBM System/360\nISA兼容性": 0.01,
    "Intel 4004\n首个微处理器": 0.05,
    "Intel 8086\nx86诞生": 0.2,
    "MIPS R2000\nRISC革命": 1.0,
    "ARM2\n嵌入式RISC": 0.8,
    "Intel Pentium\n超标量": 8,
    "Pentium Pro\n乱序执行": 20,
    "NVIDIA GeForce 256\nGPU": 0.5,  # GPU 不适合 SPEC
    "Intel Itanium\nEPIC/VLIW": 15,
    "AMD Athlon 64 X2\n多核时代": 50,
    "iPhone (ARM)\n移动计算": 3,
    "Intel Sandy Bridge\n集成GPU": 120,
    "Google TPU v1\nDSA推理": 0.1,  # 专用，不适合 SPEC
    "NVIDIA V100\nTensor Core": 0.3,
    "Apple M1\nARM桌面": 180,
    "NVIDIA H100\nHopper": 0.5,
}

perf_vals = []
labels = []
for y, event, _, freq, _, _ in milestones:
    perf = spec_estimates.get(event, 1.0)
    perf_vals.append(perf)
    labels.append(event.split('\n')[0])

colors = [colors_map.get(m[5], 'gray') for m in milestones]
bars = ax4.barh(range(len(labels)), perf_vals, color=colors, edgecolor='black', linewidth=0.5)
ax4.set_yticks(range(len(labels)))
ax4.set_yticklabels(labels, fontsize=7)
ax4.set_xlabel('相对性能 (MIPS R2000 = 1.0, 对数坐标)')
ax4.set_xscale('log')
ax4.set_title('各架构估算相对性能对比')
ax4.grid(True, alpha=0.3, axis='x')

plt.tight_layout()
plt.savefig('architecture_evolution_timeline.png', dpi=150, bbox_inches='tight')
plt.show()

print("时间线图已保存为 architecture_evolution_timeline.png")

# ===== 统计分析 =====
print("\n" + "=" * 55)
print("关键统计数据")
print("=" * 55)

# 晶体管增长率
for i in range(1, len(valid_transistors)):
    years_diff = valid_transistors[i][0] - valid_transistors[i-1][0]
    growth = valid_transistors[i][1] / valid_transistors[i-1][1]
    if years_diff > 0:
        cagr = growth ** (1/years_diff) - 1
        print(f"{valid_transistors[i-1][2].split(chr(10))[0]} → "
              f"{valid_transistors[i][2].split(chr(10))[0]}: "
              f"{growth:.1f}× ({years_diff}年, CAGR={cagr*100:.0f}%)")
```

## 分析任务

1. 在时间线图上添加 Dennard 缩微定律失效的分界线（约 2006 年）
2. 计算不同时代的性能/功耗比变化率
3. 预测 2025-2035 年可能出现的架构趋势，用虚线标注在图上

## 思考题

1. 哪些架构创新由"硬件驱动"？哪些由"软件/应用驱动"？
2. 为什么 x86 能够在多次架构革命中幸存，而大多数 ISA 消失了？
3. 历史是否正在重演？DSA 的兴起与 1980s 的协处理器（FPU）有何异同？
