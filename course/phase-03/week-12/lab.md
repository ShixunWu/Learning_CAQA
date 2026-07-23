# Week 12 实验：Python 分支预测器实现

## 实验目标

用 Python 实现 1-bit 和 2-bit 分支预测器，使用真实分支 trace 测试准确率。

---

## 代码框架

```python
class BranchPredictor:
    def __init__(self, predictor_type, table_size=1024):
        self.type = predictor_type
        self.table_size = table_size
        # 2-bit: 0=StrongNT, 1=WeakNT, 2=WeakT, 3=StrongT
        # 1-bit: 0=NT, 1=T
        self.table = [0] * table_size
        self.predictions = []
        self.outcomes = []

    def predict(self, pc):
        idx = (pc >> 2) % self.table_size
        if self.type == "1bit":
            return self.table[idx]  # 0=NT, 1=T
        else:  # 2bit
            return 1 if self.table[idx] >= 2 else 0

    def update(self, pc, taken):
        idx = (pc >> 2) % self.table_size
        self.outcomes.append(taken)
        self.predictions.append(self.predict(pc))

        if self.type == "1bit":
            self.table[idx] = 1 if taken else 0
        else:  # 2-bit saturating counter
            if taken:
                self.table[idx] = min(3, self.table[idx] + 1)
            else:
                self.table[idx] = max(0, self.table[idx] - 1)

    def accuracy(self):
        correct = sum(1 for p, o in zip(self.predictions, self.outcomes) if p == o)
        return correct / len(self.outcomes) * 100


# === 测试：模拟简单循环分支模式 ===
def generate_loop_trace(iterations=100):
    """生成循环分支 trace: 循环体内 always taken, 最后一次 NT"""
    trace = []
    for i in range(iterations):
        for j in range(10):  # 10 次内部循环
            trace.append((0x1000, 1 if j < 9 else 0))  # (PC, taken)
    return trace

# 1-bit 测试
trace = generate_loop_trace(100)
bp1 = BranchPredictor("1bit")
bp2 = BranchPredictor("2bit")

for pc, taken in trace:
    bp1.predict(pc)
    bp1.update(pc, taken)
    bp2.predict(pc)
    bp2.update(pc, taken)

print(f"1-bit accuracy: {bp1.accuracy():.2f}%")
print(f"2-bit accuracy: {bp2.accuracy():.2f}%")
```

---

## 任务

1. **运行代码**：观察 1-bit vs 2-bit 在循环 trace 上的准确率差
2. **修改循环**：把内循环改成 2 次迭代，再次对比
3. **实现 BTFNT**：写一个 `predict_btfnt(pc, target_pc)` 函数，PC 增长取 NT，减少取 T
4. **实现 (2,2) 相关预测器**：用 2-bit 全局历史索引 4 个 2-bit 计数器
5. **构造交替模式**：T/NT/T/NT... 的 trace，对比 1-bit、2-bit、(1,2) 相关预测器的准确率

---

## 交付物

1. 完整的 Python 代码（包含所有预测器实现）
2. 不同预测器在不同 trace 上的准确率对比表
3. 一句话解释：为什么 2-bit 在循环上优于 1-bit？
