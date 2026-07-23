# Week 11：流水线冒险与前推

> **阅读范围**：Appendix C §C.3-C.4
> **核心问题**：流水线中有哪些"冒险"？如何用前推（forwarding）消除数据冒险？

---

## 阅读目标

1. 识别三种流水线冒险：结构冒险、数据冒险、控制冒险
2. 掌握 RAW/WAW/WAR 三种数据依赖的区别
3. 理解前推路径的设计及其消除停顿的条件
4. 会计算带前推/不带前推时的 CPI

---

## 核心概念

### 三种冒险

| 类型 | 原因 | 典型场景 |
|------|------|----------|
| 结构冒险 | 硬件资源冲突 | 单端口存储器同时 IF + MEM |
| 数据冒险 | 指令间数据依赖 | RAW: `add x1,x2,x3` → `sub x4,x1,x5` |
| 控制冒险 | 分支跳转改变 PC | `beq` 需等到 EX 才知目标 |

### 数据依赖分类

- **RAW（Read After Write）**：真依赖，必须等待，最常见
- **WAW（Write After Write）**：写后写，仅在乱序执行中出现
- **WAR（Write After Read）**：读后写，仅当乱序且反依赖

> 5 级顺序流水线中，只有 RAW 会导致冒险！

### 前推路径

\[
\text{EX/MEM} \xrightarrow{\text{forward}} \text{EX 输入}
\]
\[
\text{MEM/WB} \xrightarrow{\text{forward}} \text{EX 输入}
\]

- 从 EX/MEM 前推：结果在 EX 阶段刚算出，可直接转发给下一条
- 从 MEM/WB 前推：结果在 WB 阶段才写回，但可提前从 MEM/WB 流水线寄存器读取
- 前推不能消除的情况：**load-use 冒险**（`lw` 后紧跟使用该值的指令），仍需 1 个 stall

### 流水线互锁（Pipeline Interlock）

硬件检测到 RAW 冒险时自动插入气泡的机制。前推能消除的冒险不需要互锁，只有 load-use 等前推无法覆盖的场景才触发。

---

## 动手实验

详见 [lab.md](lab.md)

## 自测

详见 [self-test.md](self-test.md)
