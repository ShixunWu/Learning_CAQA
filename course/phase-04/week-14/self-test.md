# Week 14 自测：Tomasulo 算法

## Q1：寄存器重命名

在 Tomasulo 算法中，以下指令的 WAW 冒险如何被消除？
```
MULT F0, F2, F4    ; 10 cycles
ADD  F0, F6, F8    ; 2 cycles   ← WAW: 两条都写 F0
SUB  F10, F0, F12  ; 2 cycles   ← 读哪个 F0？
```

回答问题：
- MULT 发射后，F0 在 reg_status 中指向哪个 RS？
- ADD 发射后，F0 被重映射到哪个 RS？SUB 的源操作数 F0 会获得哪个标签？
- 最终 F0 的值是谁的结果？为什么这符合程序语义？

## Q2：CDB 广播

保留站 Add1 的 Qj = "Mul2"，当 Mul2 在 CDB 上广播结果时，Add1 如何响应？如果 Add1 的 Vk 也已经就绪，下一步会发生什么？

## Q3：Tomasulo vs Scoreboard

对于以下有 WAW 和 WAR 的序列，分别说明记分板和 Tomasulo 的行为：
```
DIV  F2, F6, F8     ; 40c
ADD  F6, F10, F12   ; 2c, WAW(不相关) + WAR(F6被DIV读)
SUB  F10, F14, F16  ; 2c
```

记分板：ADD 在 Issue 被阻塞（WAW on F6）？还是在 Write 被阻塞（WAR）？
Tomasulo：各指令发射和执行的情况？

## Q4：保留站分配

Tomasulo 处理器有 3 个 Add RS 和 2 个 Mul RS。如果有 4 条 ADD 指令连续发射，第 4 条会发生什么？对 CPI 有何影响？

## Q5：综合计算

给定 2 个 Add RS、1 个 Mul RS：
```
MULT F0, F2, F4   ; 10c
ADD  F2, F6, F8   ; 2c, 独立于上条
ADD  F10, F2, F0  ; 2c, RAW on both F2 and F0
```
画出保留站状态表（Issue 后、执行中、Write 后），并计算总周期数。

---

## 答案

### A1
- MULT 发射后：F0 → "Mul0"（假设分配到 Mul0）
- ADD 发射后：F0 → "Add0"（重映射），reg_status 更新为 Add0
- SUB 发射时读 F0 → 获得 Qj="Add0" → 等待 Add0 的结果。SUB 将读取 ADD 的结果（最新写入者）。
- 最终 F0 = ADD 的结果，符合程序语义（后一条覆盖前一条）。

### A2
Add1 监听到 CDB 上 Mul2 的广播，发现自己的 Qj=="Mul2" → 捕获数据到 Vj，清空 Qj。若 Vk 也已就绪（无 Qk），则下一步 Add1 可以进入 Execute。

### A3
- 记分板：ADD 的 Issue：检查 WAW → F6 被 DIV 占用 → ADD 阻塞在 Issue 阶段。即使 WAW 不存在，ADD 也会在 Write 阶段被阻塞（WAR on F6——DIV 还要读 F6）。
- Tomasulo：DIV 发射到 Mul RS，reg_status[F6]=DIV_RS。ADD 发射到 Add RS，F6 被重命名为 ADD_RS，不影响 DIV。SUB 也直接发射。所有三条指令在 Tomasulo 下几乎可以同时发射。

### A4
第 4 条 ADD 因无空闲 RS 被阻塞在 Issue 阶段，产生结构冒险。这反映为 CPI 增大——后续指令即使数据就绪也无法发射。

### A5
**Tomasulo 无推测（无 ROB）**，2 个 Add RS，1 个 Mul RS：

- C1: MULT→Mul0（读 F2 旧值、F4 → 立即 ReadOps）。ADD1→Add0（读 F6、F8 → 立即 ReadOps）。reg_status[F2]=Add0, reg_status[F0]=Mul0。ADD2→Add1（仍有空闲 RS），但 Qj=Add0（等 F2），Qk=Mul0（等 F0）。
- C2-C11: MULT Exec@C2-11, Write@C12（Mul0 释放）。
- C2-C3: ADD1 Exec@C2-3, Write@C4（Add0 释放，广播 F2 新值）→ ADD2 从 CDB 捕获 F2，清除 Qj。
- C12: MULT Write（广播 F0）→ ADD2 从 CDB 捕获 F0，清除 Qk，两操作数就绪。
- C12-C13: ADD2 Exec, Write@C14。
- **总周期：14，CPI = 14/3 ≈ 4.67。**

> 注意：Tomasulo 无推测版本中 RS 在指令 Write（CDB 广播）后释放，指令也按数据流顺序完成，不涉及 ROB 的按序提交。若题目未提及 speculation/ROB，不应假设有 ROB。
