# Week 14：Tomasulo 算法

> **阅读范围**：Ch3 §3.4-3.5
> **核心问题**：如何用保留站（Reservation Station）和 CDB 同时消除 RAW、WAW 和 WAR？

---

## 阅读目标

1. 理解 Tomasulo 算法相比记分板的三个关键改进
2. 掌握保留站、CDB（公共数据总线）的工作原理
3. 理解寄存器重命名如何消除 WAW/WAR
4. 能手工追踪 Tomasulo 算法的指令执行过程

---

## 核心概念

### Tomasulo vs Scoreboard

| 特性 | Scoreboard | Tomasulo |
|------|-----------|----------|
| WAW | Issue 阻塞 | 寄存器重命名消除 |
| WAR | Write 阻塞 | 寄存器重命名消除 |
| 前推 | 无 | CDB 广播 |
| 结构冒险 | 按 FU 类型检测 | 保留站不足时阻塞 |
| 实现 | CDC 6600 | IBM 360/91 |

### 保留站（Reservation Station）

每个功能单元有若干保留站，存储：
- 操作码（Op）
- 源操作数 1 的值 Vj 或标签 Qj（哪个 RS 将产生此值）
- 源操作数 2 的值 Vk 或标签 Qk
- 目标寄存器（仅用于 CDB 匹配）
- Busy 标志

**关键**：操作数可以是"值"或"标签（tag）"。标签就是产生该值的保留站编号。

### CDB（Common Data Bus）

- 所有结果通过 CDB 广播
- 保留站和寄存器文件同时监听 CDB
- 匹配标签即捕获数据 → 实现了前推（无需额外路径）

### 三阶段

1. **Issue**：从指令队列取指令，分配到空闲 RS。如果 RS 满 → stall
2. **Execute**：两个操作数都就绪（Vj,Vk 有效）→ 开始执行
3. **Write Result**：结果通过 CDB 广播，等待的 RS 和 RegFile 同时捕获

### 寄存器重命名的本质

指令 `ADD F0, F2, F4` 发射时，不去读 F2/F4 的值，而是去查寄存器状态表看哪个 RS 将产生这些值。这就把"寄存器名"映射到了"RS 编号"——消除 WAW/WAR 的根本原因。

---

## 动手实验

详见 [lab.md](lab.md)

## 自测

详见 [self-test.md](self-test.md)
