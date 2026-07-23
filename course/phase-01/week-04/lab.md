# Week 4 实验：综合性能分析

## 实验：用 Iron Law 分析真实处理器

### 任务

选择三款真实处理器（如 Apple M1, Intel Core i7-13700, AMD Ryzen 7950X），收集它们的规格并分析：

```python
# 从公开数据收集以下参数
processors = {
    "M1":    {"freq": 3.2, "cores": 8,  "tdp": 15,  "process": 5},
    "i7":    {"freq": 5.0, "cores": 16, "tdp": 125, "process": 10},
    "Ryzen": {"freq": 5.7, "cores": 16, "tdp": 170, "process": 5},
}

def perf_index(freq, cores):
    """简化的性能指标"""
    return freq * cores

def efficiency(perf, tdp):
    return perf / tdp

for name, spec in processors.items():
    perf = perf_index(spec["freq"], spec["cores"])
    eff = efficiency(perf, spec["tdp"])
    print(f"{name}: Perf={perf:.0f}, Efficiency={eff:.2f} perf/W")
```

### 分析

1. 如果 M1 的 SIMD 宽度是 i7 的 2 倍（影响 IC），这如何改变性能排名？
2. 用 Amdahl 定律分析：如果能把 M1 中 80% 的代码向量化且加速 4x，性能提升多少？

---

## 交付物

1. 三个处理器的性能对比表
2. 基于 Iron Law 的分解分析（IC/CPI/Cycle Time 的定性贡献）
3. 一句话：为什么能效比在现代处理器比较中越来越重要？
