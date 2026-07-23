# Week 13 自测：记分板

## Q1：记分板 vs 顺序流水线

记分板动态调度相比经典 5 级流水线（带前推），主要克服了哪种限制？
A) 结构冒险  B) RAW 冒险  C) 严格顺序执行  D) 控制冒险

## Q2：WAW 冒险检测

以下两条指令在哪一阶段会发生 WAW 冲突？记分板如何处理？
```
DIV  F0, F2, F4
ADD  F0, F6, F8   ; 同样写 F0
```

## Q3：记分板三张表

对于以下指令序列（Mult=10c, Add=2c），填写第 3 周期的 Register Result Status：

```
MULT F0, F2, F4    ; Issue @C1
SUB  F8, F0, F6    ; Issue @C2
ADD  F10, F8, F2   ; Issue @C3
```

- C3 结束时，哪些寄存器处于 "等待" 状态？各自被哪个 FU 写入？

## Q4：WAR 场景

```
ADD  F2, F4, F6    ; F2 = F4 + F6, 2c
SUB  F4, F8, F10   ; F4 = F8 - F10, 2c, WAR: 上条要读 F4
```

- SUB 的 Issue 会被阻塞吗？为什么？
- SUB 的 Write 会被阻塞吗？什么条件下？
- 画出这两条指令从 Issue 到 Write 的时间线

## Q5：CPI 计算

在记分板下，指令序列：
```
MULT F0, F2, F4   ; 10c
ADD  F6, F0, F8   ; 2c  (RAW on F0)
SUB  F10, F12, F14 ; 2c  (independent)
```

- 每条指令各阶段的时间（Issue/ReadOps/Exec/Write）
- 总执行周期
- 有效 CPI

---

## 答案

### A1
**C**。记分板允许数据就绪的指令提前执行（乱序执行），打破了顺序流水线中必须按程序顺序进入 EX 的限制。

### A2
DIV 在 Issue 时检查到 ADD 也要写 F0 → WAW。处理方式：ADD 在 Issue 阶段被阻塞，直到 DIV 完成 Write 后才能发射。

### A3
C3 结束时 Register Result Status：
- F0 → 被 Mult1 写入（尚未完成）
- F8 → 被 Add 写入（尚未完成）
- F10 → 被 Add 写入（注意：Add FU 只有一个，SUB 和 ADD 不能同时占用，ADD 需等待。实际上 ADD 因结构冒险可能在 Issue 被阻塞）

### A4
- SUB 的 Issue **不会被阻塞**：Issue 只检查结构冒险和 WAW, F4 没有被作为目标寄存器等待
- SUB 的 Write **会被阻塞**：ADD 还需要读 F4 的旧值（WAR），直到 ADD 完成 Read Ops 后，SUB 才能 Write
- 时间线：ADD Issue@C1, ReadOps@C2, SUB Issue@C2, ReadOps@C3, ADD Exec@C3-4, SUB 等 ADD ReadOps 完成后才能 Write@C5

### A5
- MULT: Issue@1, ReadOps@2, Exec@3-12, Write@13
- ADD: Issue@2, ReadOps@13（等 F0 就绪），Exec@14-15, Write@16
- SUB: Issue@3, ReadOps@4, Exec@5-6, Write@7
- 总周期：16，CPI = 16/3 ≈ 5.33
- 顺序流水线（无前推）：MULT 阻塞 ADD 需额外等待，总周期更长
