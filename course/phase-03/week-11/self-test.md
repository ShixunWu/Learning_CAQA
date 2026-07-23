# Week 11 自测：冒险与前推

## Q1：RAW 依赖分析

对于以下 RISC-V 代码序列，画出 5 级流水线时空图，标注所有 RAW 依赖，并指出哪些依赖可通过前推消除：

```asm
add  x1, x2, x3
sub  x4, x1, x5
and  x6, x4, x7
or   x8, x6, x9
```

## Q2：前推与停顿计数

假设无前推时，每条 RAW 依赖需要 2 个 stall。有前推时，ALU→ALU 依赖 0 stall，load-use 依赖 1 stall。

```asm
lw   x1, 0(x2)
add  x3, x1, x4      # 依赖 x1 (load-use)
sub  x5, x3, x6      # 依赖 x3 (ALU→ALU)
```

- 无前推时，总 stall 数 = ?
- 有前推时，总 stall 数 = ?
- 有前推时总周期数 = (指令数 + stall 数 + 4)

## Q3：前推路径选择题

在经典 5 级流水线中，RAW 冒险的前推数据可以来自哪些流水线寄存器？
A) 仅 EX/MEM
B) 仅 MEM/WB
C) EX/MEM 和 MEM/WB
D) ID/EX 和 EX/MEM

## Q4：load-use 冒险

为什么 lw 指令后紧跟使用其结果的指令时，即使有前推也必须插入 1 个 stall？画图说明数据可用时间 vs 需要时间。

## Q5：CPI 综合计算

某程序执行 1000 条指令，其中：
- 20% 是 lw，其后 60% 情况紧跟使用该值（load-use）
- 30% 是 ALU 指令，其后 40% 情况有数据依赖

假设理想 CPI=1，无前推时 RAW=2 stall。计算无前推和有前推时的总执行周期和有效 CPI。

---

## 答案

### A1

流水线时空图（C=周期）：

| 指令 | C1 | C2 | C3 | C4 | C5 | C6 | C7 | C8 |
|------|----|----|----|----|----|----|----|----|
| add  | IF | ID | EX | MEM| WB |    |    |    |
| sub  |    | IF | ID | EX | MEM| WB |    |    |
| and  |    |    | IF | ID | EX | MEM| WB |    |
| or   |    |    |    | IF | ID | EX | MEM| WB |

依赖链：add→sub (x1), sub→and (x4), and→or (x6)，全部 RAW，全部可通过前推消除（均为 ALU→ALU）。

### A2
- 无前推：add 依赖 lw=2 stall + sub 依赖 add=2 stall = 4 stall
- 有前推：add 依赖 lw=1 stall + sub 依赖 add=0 stall = 1 stall
- 总周期：3+1+4=8 cycles

### A3
**C**。前推数据可来自 EX/MEM（刚算出的 ALU 结果）和 MEM/WB（上一条指令的结果即将写回）。

### A4
lw 的数据在 MEM 阶段结束时才可用（C4 末尾），但下一条指令在 EX 阶段初（C4 开头）就需要该数据。前推只能从 MEM/WB 转发（比 EX/MEM 晚一个周期），因此无论如何都差 1 个周期，必须插入 1 个 stall。

### A5
- load-use 指令数：200×0.6=120；ALU依赖数：300×0.4=120
- 无前推 stall：120×2 + 120×2 = 480；CPI=1+0.48=1.48；总周期=1000+480+4=1484
- 有前推 stall：120×1 + 120×0 = 120；CPI=1+0.12=1.12；总周期=1000+120+4=1124
- 加速比=1484/1124≈1.32x
