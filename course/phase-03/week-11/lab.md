# Week 11 实验：Ripes 冒险与前推分析

## 实验目标

使用 Ripes 构造 RAW 数据冒险场景，对比有无前推时的停顿差异，量化 CPI 影响。

---

## 实验步骤

### Step 1：构造 RAW 冒险（无前推）

在 Ripes 中输入以下代码，**关闭 Forwarding**（Processor Settings → Enable Forwarding = OFF）：

```asm
addi x1, x0, 5       # x1 = 5
addi x2, x1, 10      # x2 = x1 + 10  ← RAW: 依赖上一条的 x1
addi x3, x2, 15      # x3 = x2 + 15  ← RAW: 依赖上一条的 x2
addi x4, x3, 20      # x4 = x3 + 20  ← RAW: 依赖上一条的 x3
```

逐周期记录，统计：
- 总周期数
- 每条指令的 stall 次数
- 有效 CPI = 总周期数 / 指令数

### Step 2：开启前推

开启 Forwarding，重新运行相同代码。对比：
- 总周期数
- 有效 CPI
- 哪些 stall 被消除了？

### Step 3：load-use 冒险（前推无法消除）

```asm
lw   x1, 0(x0)       # load from memory
addi x2, x1, 10      # RAW: load-use，前推也无法避免 1 个 stall
addi x3, x2, 5       # RAW: 可用前推消除
```

- 关闭 Forwarding：统计 stall 数
- 开启 Forwarding：统计 stall 数
- 为什么 load-use 前推无法消除？计算地址→访存→前推的路径比 ALU→ALU 长一个周期

### Step 4：计算 CPI 影响

假设理想 CPI=1，每条 ALU→ALU 依赖在无前推时需 2 个 stall，load-use 无前推时需 2 个 stall、有前推时仍需 1 个 stall（因为数据在 MEM 阶段才可用，无法在 EX 阶段前推到下一条指令）。

对于以下指令混合：
- 30% 指令有 RAW 依赖（其中 80% 是 ALU→ALU，20% 是 load-use）

计算：
- 无前推时的有效 CPI
- 有前推时的有效 CPI
- 前推带来的加速比

---

## 交付物

1. Step 1-3 的周期计数与截图
2. Step 4 的计算过程
3. 用一句话总结：哪种冒险前推无法解决？
