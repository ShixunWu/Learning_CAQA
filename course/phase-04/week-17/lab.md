# Week 17 实验：VLIW 编译器调度模拟

## 实验目标

用 Python 模拟 VLIW 编译器的静态调度过程，观察 NOP 填充率和代码膨胀。

---

## 代码框架

```python
class VLIWScheduler:
    """简单的 VLIW 静态调度器"""
    def __init__(self, fu_config):
        self.fu_config = fu_config  # {fu_type: count}
        self.slots = []

    def schedule(self, instructions):
        """
        instructions: list of (op, latency, dependencies)
        返回 VLIW bundle 序列
        """
        bundles = []
        ready_time = {}  # {指令索引: 最早执行时间}

        for i, (op, lat, deps) in enumerate(instructions):
            # 计算该指令的最早可用时间
            avail = 0
            for dep in deps:
                avail = max(avail, ready_time[dep] + instructions[dep][1])
            # 在 bundles[avail:] 中找一个有空闲 slot 的周期
            cycle = avail
            while True:
                if cycle >= len(bundles):
                    bundles.append({fu: 0 for fu in self.fu_config})
                if bundles[cycle][op] < self.fu_config[op]:
                    bundles[cycle][op] += 1
                    break
                cycle += 1
            ready_time[i] = cycle
            self.slots.append((op, cycle))

        return bundles


# 测试指令序列
# (操作, 延迟, [依赖指令索引])
instructions = [
    ("ALU", 1, []),       # I0: ADD  x1, x2, x3
    ("ALU", 1, [0]),      # I1: ADD  x4, x1, x5  ← 依赖 I0
    ("MUL", 3, []),       # I2: MUL  x6, x7, x8
    ("ALU", 1, []),       # I3: ADD  x9, x10, x11
    ("ALU", 1, [2, 3]),   # I4: ADD  x12, x6, x9 ← 依赖 I2 和 I3
]

scheduler = VLIWScheduler({"ALU": 2, "MUL": 1, "MEM": 1})
bundles = scheduler.schedule(instructions)

for i, b in enumerate(bundles):
    total_slots = sum(b.values())
    used_slots = sum(self.fu_config.values())
    print(f"Bundle {i}: {b} (利用率: {total_slots}/{used_slots})")
```

---

## 任务

1. **完善调度器**：支持不同操作类型、不同延迟
2. **测试不同 FU 配置**：ALU×2+MUL×1 vs ALU×4+MUL×2，观察 NOP 率
3. **计算代码膨胀**：VLIW bundle 的 NOP 占比
4. **对比指令序列**：有依赖 vs 无依赖序列的 slot 利用率差异
5. **模拟循环展开**：对循环体展开 4 次后调度，对比展开前后的利用率

---

## 交付物

1. 完整调度器代码
2. 不同 FU 配置下的 slot 利用率表
3. 循环展开前后对比分析
