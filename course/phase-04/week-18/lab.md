# Week 18 实验：ILP 极限探索

## 实验目标

通过修改 Tomasulo+ROB 模拟器的资源配置，观察 ILP 随资源增加的变化曲线，验证收益递减规律。

---

## 代码框架

基于 Week 15 的 Tomasulo+ROB 模拟器，添加参数配置：

```python
def run_experiment(config):
    """
    config = {
        "rob_size": 16,
        "issue_width": 2,
        "int_alu": 2, "fp_alu": 1, "ls_unit": 1,
        "predictor": "2bit"
    }
    返回 IPC
    """
    # 运行 trace → 计算 IPC
    pass

# 扫描 ROB 大小
for rob_size in [4, 8, 16, 32, 64, 128]:
    ipc = run_experiment({"rob_size": rob_size, ...})
    print(f"ROB={rob_size}: IPC={ipc:.2f}")

# 扫描发射宽度
for width in [1, 2, 4, 8]:
    ipc = run_experiment({"issue_width": width, ...})
    print(f"Width={width}: IPC={ipc:.2f}")

# 组合扫描
results = []
for width in [1, 2, 4, 8]:
    for rob in [16, 32, 64, 128]:
        ipc = run_experiment({"issue_width": width, "rob_size": rob, ...})
        results.append((width, rob, ipc))
```

---

## 任务

1. **ROB 大小扫描**：固定其他参数，改变 ROB 大小 4→128，记录 IPC。找到 IPC 饱和点
2. **发射宽度扫描**：固定 ROB=64，改变发射宽度 1→8
3. **FU 数量扫描**：固定其他参数，改变整数 ALU 数量 1→8
4. **绘制收益递减曲线**：用 matplotlib 画出 IPC vs 资源（ROB/Width/FU）
5. **分析瓶颈**：哪个参数增大对 IPC 提升最大？哪个最先饱和？

---

## 交付物

1. 完整的参数扫描代码
2. IPC vs 资源曲线图
3. 分析报告：ILP 是否达到了 Wall 研究所说的极限？
