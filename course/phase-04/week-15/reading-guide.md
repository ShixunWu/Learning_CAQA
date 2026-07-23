# Week 15：推测执行与重排序缓冲

> **阅读范围**：Ch3 §3.6-3.8
> **核心问题**：如何让 Tomasulo 支持推测执行？误预测后如何精确恢复？

---

## 阅读目标

1. 理解为什么 Tomasulo 需要 ROB 来支持推测
2. 掌握 ROB（Reorder Buffer）的四字段：指令类型、目标寄存器、值、就绪标志
3. 理解推测执行下的分支恢复机制
4. 理解精确异常的概念及 ROB 如何实现它

---

## 核心概念

### Tomasulo 的局限

原始 Tomasulo 算法不支持推测执行——分支后的指令即使发射了，结果也可能需要丢弃。而且 Tomasulo 中 Write Result 直接写入寄存器文件，无法"撤销"。

### ROB（Reorder Buffer）

ROB 是介于保留站和寄存器文件之间的 FIFO 队列：

| 字段 | 含义 |
|------|------|
| Instruction Type | 分支/store/ALU/Load |
| Destination | 目标寄存器编号 |
| Value | 计算结果 |
| Ready | 是否已计算完成 |

**写回顺序**：ROB 按程序顺序（FIFO）提交结果到寄存器文件。误预测时，直接清空 ROB 中分支后的所有条目。

### 推测执行流程

1. **Issue**：指令同时进入保留站和 ROB
2. **Execute**：与 Tomasulo 相同，但结果先写入 ROB 而非寄存器文件
3. **Write Result**：结果写入 ROB 对应条目
4. **Commit**：当指令到达 ROB 头部且结果就绪 → 写入寄存器文件（或 store 写内存）

### 精确异常

ROB 按程序顺序 Commit，异常在 Commit 阶段检测。发生异常的指令到达 ROB 头部时，其后的指令尚未 Commit → 直接清空流水线，精确恢复到异常指令。

### 分支误预测恢复

分支预测错误时：
1. 清空 ROB 中比该分支更新的所有条目
2. 清空保留站（或进行状态恢复）
3. 从正确路径重新取指

---

## 动手实验

详见 [lab.md](lab.md)

## 自测

详见 [self-test.md](self-test.md)
