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
- SUB 的 Issue：若仅 1 个 Add FU，ADD 占用该 FU 直到 Write 完成（记分板中 FU 在 **Write 之后释放**，而非 Exec 之后），故 SUB 在 Issue 阶段因结构冒险被阻塞，须等 ADD Write 后才可 Issue。若有多个 Add FU，则 Issue 不被阻塞（F4 不是待写入目标，无 WAW）。
- SUB 的 Write：即使 SUB 已 Issue，Write 阶段仍需检查 WAR——ADD 的源操作数 F4 被 SUB 覆盖前，ADD 必须已完成 Read Ops。故 SUB 的 Write 被阻塞至 ADD 读取 F4 之后。
- 时间线（假设 1 个 Add FU，记分板单发射）：
  - C1: ADD Issue, C2: ADD ReadOps, C3-4: ADD Exec, C5: ADD Write（释放 Add FU）
  - C6: SUB Issue, C7: SUB ReadOps, C8-9: SUB Exec, C10: SUB Write
  - WAR 检查在 C10 通过（ADD 已在 C2 读完 F4），无需额外阻塞。
  - 若假设多 Add FU 可同时 Issue：ADD Issue@C1, ReadOps@C2; SUB Issue@C2, ReadOps@C3; ADD Exec@C3-4, SUB 的 Write 必须等 ADD 完成 ReadOps → Write@C5。

### A5
以下假设有 ≥2 个 Add FU（否则 SUB 会被 ADD 占用的 Add FU 阻塞在 Issue）。
记分板中 **FU 在 Write 完成后释放**（非 Exec 完成后），故 ADD 的 Add FU 在 C16 才释放。

- MULT: Issue@1, ReadOps@2, Exec@3-12, Write@13
- ADD: Issue@2, ReadOps@13（等 F0 就绪），Exec@14-15, Write@16
- SUB: Issue@3, ReadOps@4, Exec@5-6, Write@7
- 总周期：16，CPI = 16/3 ≈ 5.33
- 若仅 1 个 Add FU：SUB 需等 ADD 在 C16 释放 FU → Issue@17, ReadOps@18, Exec@19-20, Write@21, 总周期 21, CPI=7。这体现了记分板的结构冒险代价。
- 顺序流水线（无前推）：MULT 阻塞 ADD 需额外等待，总周期更长
