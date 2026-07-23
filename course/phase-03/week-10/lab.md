# Week 10 实验：Ripes 可视化流水线

## 实验目标

使用 Ripes 处理器可视化工具，加载简单的 RISC-V 程序，观察 5 级流水线各阶段的行为，计算执行周期数。

---

## 环境准备

1. 下载安装 Ripes：https://github.com/mortbopet/Ripes/releases
2. 启动后选择处理器模型：**5-stage (Classic RISC-V)**

---

## 实验步骤

### Step 1：加载测试程序

在 Ripes 编辑器中输入以下 RISC-V 汇编：

```asm
addi x1, x0, 10      # x1 = 10
addi x2, x0, 20      # x2 = 20
add  x3, x1, x2      # x3 = x1 + x2
addi x4, x3, 5       # x4 = x3 + 5
sub  x5, x4, x1      # x5 = x4 - x1
nop
nop
```

### Step 2：逐周期观察流水线

1. 切换到 **Pipeline** 视图
2. 点击 "Step Cycle" 逐个周期前进
3. 每步截图，记录 IF/ID/EX/MEM/WB 每个阶段正在执行的指令
4. 画出时序图（横轴=周期，纵轴=指令），标注 RAW 依赖关系

### Step 3：计算周期数

| 指令 | IF | ID | EX | MEM | WB |
|------|----|----|----|-----|-----|
| addi x1 | C1 | C2 | C3 | C4 | C5 |
| addi x2 | C2 | C3 | C4 | C5 | C6 |
| add x3 | C3 | C4 | C5→stall? | C6 | C7 |
| ... |    |    |    |     |    |

- 完整填写表格，说明 `add x3` 的 EX 阶段是否需要停顿（stall）
- 实际总周期数 = ? （含 nop 流出）

### Step 4：验证流水线加速比

- 假设单周期 CPI=1，周期时间 = 最长阶段延迟
- 计算上述程序在单周期 CPU 与 5 级流水线 CPU 的加速比
- 公式：\[ \text{Speedup} = \frac{CPI_{\text{单周期}} \times \text{指令数}}{CPI_{\text{流水线}} \times (\text{指令数} + k - 1)} \]

---

## 交付物

1. 每周期截图（至少覆盖第一条指令的完整 5 阶段）
2. 完整的流水线时空图（表格形式）
3. 总周期数计算结果及加速比分析
