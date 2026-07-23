# Week 14 实验：Python Tomasulo 模拟器

## 实验目标

用 Python 实现 Tomasulo 算法，观察寄存器重命名如何消除 WAW/WAR。

---

## 代码框架

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class ReservationStation:
    name: str
    busy: bool = False
    op: str = ""
    Vj: Optional[float] = None
    Vk: Optional[float] = None
    Qj: Optional[str] = None   # 产生 Vj 的 RS 名
    Qk: Optional[str] = None   # 产生 Vk 的 RS 名
    dest: str = ""             # 目标寄存器
    time_left: int = 0         # 剩余执行时间

class Tomasulo:
    def __init__(self):
        self.reg_status = {}          # {reg: RS_name}
        self.rs_add = [ReservationStation(f"Add{i}") for i in range(3)]
        self.rs_mul = [ReservationStation(f"Mul{i}") for i in range(2)]
        self.instructions = []
        self.cycle = 0

    def add_instr(self, op, dest, src1, src2, latency):
        self.instructions.append({
            "op": op, "dest": dest, "src1": src1, "src2": src2,
            "latency": latency, "issued": False, "done": False
        })

    def issue(self, instr, rs):
        """发射指令到保留站"""
        if rs is None: return False  # 无空闲 RS
        rs.busy = True; rs.op = instr["op"]
        rs.dest = instr["dest"]; rs.time_left = instr["latency"]
        # 源操作数：查寄存器状态表
        rs.Qj = self.reg_status.get(instr["src1"], None)
        rs.Vj = None if rs.Qj else float(instr["src1"][1:])
        rs.Qk = self.reg_status.get(instr["src2"], None)
        rs.Vk = None if rs.Qk else float(instr["src2"][1:])
        # 更新寄存器状态
        self.reg_status[instr["dest"]] = rs.name
        return True

    def step(self):
        self.cycle += 1
        # 3. Write Result: 完成的 RS 通过 CDB 广播
        # 2. Execute: 操作数就绪的执行
        # 1. Issue: 发射下一条指令
        pass  # 学生实现


# 测试：对比有无 Tomasulo 的 WAW 处理
t = Tomasulo()
t.add_instr("MULT", "F0", "F2", "F4", 10)
t.add_instr("ADD",  "F0", "F6", "F8", 2)   # WAW on F0
t.add_instr("SUB",  "F10","F0", "F12", 2)  # RAW on F0（取决于哪个 F0）
```

---

## 任务

1. **实现 step()**：完成 Issue、Execute、Write Result 三阶段
2. **追踪 WAW 消除**：观察 F0 被重命名后，两条指令如何并行执行
3. **对比记分板**：同样的 WAW 场景，记分板会阻塞 Issue，Tomasulo 不会——证明这一点
4. **构造 WAR 场景**：验证 Tomasulo 通过寄存器重命名消除 WAR
5. **统计 CPI**：给定指令序列，计算 CPI 并与理想值对比

---

## 交付物

1. 完整 Tomasulo 模拟器代码
2. WAW/WAR 场景的逐周期执行日志
3. 对比表：Tomasulo vs Scoreboard vs 顺序流水线
