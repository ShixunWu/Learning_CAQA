# Week 8 实验：TLB 行为观察

## 实验：gem5 TLB 配置

```bash
# 查看 gem5 中的 TLB 统计
build/RISCV/gem5.opt configs/example/se.py \
  --cmd=/path/to/benchmark \
  --caches

# 在 m5out/stats.txt 中搜索 TLB 相关统计：
# system.cpu.dtb.hits / misses
# system.cpu.itb.hits / misses
```

## Python 模拟：TLB 覆盖范围计算

```python
def tlb_coverage(entries, page_size_kb):
    return entries * page_size_kb  # KB

# 典型配置
configs = [
    ("Small pages", 64, 4),
    ("Large pages", 64, 2048),   # 2MB pages
    ("More entries", 256, 4),
    ("Ideal", 1024, 2048),
]

for name, entries, ps in configs:
    coverage_mb = tlb_coverage(entries, ps) / 1024
    print(f"{name}: {entries} entries × {ps}KB = {coverage_mb:.0f} MB")

# 工作集分析：
workloads = [
    ("Streaming audio", 0.5),    # MB
    ("Web browser", 200),
    ("Database index", 2000),
    ("Scientific simulation", 8000),
]
print("\nWorkload vs TLB coverage:")
for wl_name, wl_size in workloads:
    covered = any(tlb_coverage(e, ps)/1024 >= wl_size
                  for _, e, ps in configs[:-1])
    print(f"  {wl_name} ({wl_size}MB): {'Fits' if covered else 'TLB thrashing'}")
```

## 交付物

1. gem5 TLB 统计数据（hits/misses）
2. 不同 TLB 配置的覆盖范围对比表
3. 一句话解释：为什么数据库和大科学计算喜欢大页？
