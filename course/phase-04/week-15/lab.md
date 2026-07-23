# Week 15 实验：ROB + Tomasulo 模拟器扩展

## 实验目标

在 Week 14 的 Tomasulo 模拟器基础上增加 ROB，实现推测执行和精确异常恢复。

---

## 代码框架

```python
class ROBEntry:
    def __init__(self, idx):
        self.idx = idx
        self.busy = False
        self.instr_type = ""   # "ALU", "LOAD", "STORE", "BRANCH"
        self.dest = ""         # 目标寄存器
        self.value = None
        self.ready = False
        self.pc = 0            # 用于分支恢复

class TomasuloWithROB:
    def __init__(self, rob_size=8):
        self.rob = [ROBEntry(i) for i in range(rob_size)]
        self.rob_head = 0
        self.rob_tail = 0
        # ... 保留站、寄存器状态表同 Week 14

    def issue(self, instr):
        """发射：同时分配 RS 和 ROB"""
        if self.rob[(self.rob_tail + 1) % len(self.rob)].busy:
            return False  # ROB 满
        # 分配 ROB 条目
        entry = self.rob[self.rob_tail]
        entry.busy = True
        entry.instr_type = instr["type"]
        entry.dest = instr["dest"]
        entry.pc = instr["pc"]
        self.rob_tail = (self.rob_tail + 1) % len(self.rob)
        # 分配保留站（同 Week 14）
        # ...

    def commit(self):
        """提交：ROB 头部就绪则写回寄存器文件"""
        entry = self.rob[self.rob_head]
        if entry.busy and entry.ready:
            if entry.instr_type == "BRANCH" and entry.value != entry.predicted:
                self.flush_after(self.rob_head)  # 误预测！
            elif entry.dest:
                # 写回寄存器文件（如果 reg_status 仍指向此 ROB，清空）
                pass
            entry.busy = False
            self.rob_head = (self.rob_head + 1) % len(self.rob)

    def flush_after(self, rob_idx):
        """清空该 ROB 条目之后的所有条目"""
        i = (rob_idx + 1) % len(self.rob)
        while i != self.rob_tail:
            self.rob[i].busy = False
            i = (i + 1) % len(self.rob)
        self.rob_tail = (rob_idx + 1) % len(self.rob)
```

---

## 任务

1. **实现 Commit 阶段**：FIFO 顺序提交，误预测时清空
2. **测试分支恢复**：注入一条误预测分支，观察 ROB 如何清空后续指令
3. **对比性能**：有无推测执行（分支后是否发射指令）的 CPI 对比
4. **精确异常测试**：在指令流中注入一条会触发异常的指令（如除零），验证其到达 ROB 头部时后续指令尚未提交

---

## 交付物

1. 扩展后的模拟器代码
2. 误预测恢复的逐周期日志
3. 推测 vs 非推测的 CPI 对比
