# Week 13 实验：Python 记分板模拟器

## 实验目标

用 Python 实现 CDC 6600 记分板算法，手工跟踪指令执行的 4 个阶段。

---

## 代码框架

```python
class Scoreboard:
    def __init__(self):
        # 功能单元: {name: (busy, op, Fi, Fj, Fk, Qj, Qk, Rj, Rk)}
        self.fus = {
            "Integer": [False, "", "", "", "", "", "", False, False],
            "Mult1":   [False, "", "", "", "", "", "", False, False],
            "Mult2":   [False, "", "", "", "", "", "", False, False],
            "Add":     [False, "", "", "", "", "", "", False, False],
            "Divide":  [False, "", "", "", "", "", "", False, False],
        }
        # 寄存器结果状态: 哪个 FU 将写入此寄存器（空=就绪）
        self.reg_status = {}
        self.cycle = 0
        self.instructions = []

    def add_instruction(self, op, dest, src1, src2, latency):
        self.instructions.append({
            "op": op, "dest": dest, "src1": src1, "src2": src2,
            "latency": latency, "issue": None, "read_ops": None,
            "exec_done": None, "write": None
        })

    def step(self):
        """简化版：单周期推进"""
        self.cycle += 1
        # 1. Write Result（检查 WAR）
        # 2. Read Operands（检查 RAW）
        # 3. Issue（检查结构冒险 + WAW）
        # 4. Execution 计数递减
        # (实际应倒序检查以体现优先权)
        pass  # 学生实现


# === 测试用例 ===
sb = Scoreboard()
# 假设: Mult=10c, Add=2c, Divide=40c
sb.add_instruction("MULT", "F0", "F2", "F4", 10)  # MULT F0, F2, F4
sb.add_instruction("SUB",  "F8", "F0", "F6", 2)    # SUB  F8, F0, F6
sb.add_instruction("ADD",  "F10","F8", "F2", 2)    # ADD  F10, F8, F2
```

---

## 任务

1. **实现 step()**：按 Issue→ReadOps→Exec→Write 顺序推进，处理 WAW/RAW/WAR
2. **追踪一例**：用以上代码运行，输出每周期三张表的状态
3. **观察 WAR**：构造一个 WAR 场景，展示记分板如何通过 stall Write 处理 WAR
4. **对比 CPI**：同样指令序列，记分板 vs 顺序流水线（无前推）的 CPI 对比
5. **构造 WAW**：让一条指令在 Issue 阶段因 WAW 而被阻塞

---

## 交付物

1. 完整可运行的 Python 记分板模拟器
2. 每周期状态表（至少 6 个周期）
3. CPI 对比分析
