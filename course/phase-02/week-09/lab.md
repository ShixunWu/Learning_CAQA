# Week 9 实验：CACTI Cache 建模与 RAID 计算

## 实验 1：CACTI Cache 建模

```bash
# CACTI 输入文件示例 (cache.cfg)
# 32KB, 64B blocks, 4-way set-associative L1 data cache
-size 32768
-block_size 64
-associativity 4
-technology 45  # nm
-cache_type "data"

# 运行
cd cacti
./cacti -infile cache.cfg > l1_result.txt
```

提取关键指标：
```python
# 解析 CACTI 输出
def parse_cacti(output_file):
    with open(output_file) as f:
        text = f.read()
    import re
    access_time = re.search(r'Access time.*?(\d+\.\d+).*?ns', text)
    energy = re.search(r'Total dynamic.*?read.*?energy.*?(\d+\.\d+).*?nJ', text)
    area = re.search(r'Total area.*?(\d+\.\d+).*?mm\^2', text)
    return {
        'access_time_ns': float(access_time.group(1)),
        'energy_nj': float(energy.group(1)),
        'area_mm2': float(area.group(1))
    }

# 扫描不同配置
configs = [
    (16, 32, 4),   # 16KB, 32B blocks, 4-way
    (32, 64, 4),   # 32KB, 64B blocks, 4-way
    (64, 64, 8),   # 64KB, 64B blocks, 8-way
]
```

## 实验 2：RAID 可靠性计算

```python
def mttdl_raid5(n, mttf_disk, mttr_hours):
    """RAID 5 的平均数据丢失时间 (MTTDL)"""
    # 近似公式
    return (mttf_disk**2) / (n * (n-1) * mttr_hours)

# 8 块盘，每块 MTTF=1M hours，修复时间 24h
mttdl = mttdl_raid5(8, 1_000_000, 24)
print(f"MTTDL: {mttdl/24/365:.0f} years")
```

## 交付物

1. 不同 Cache 配置的 CACTI 建模结果表（延迟/功耗/面积）
2. RAID 5 vs RAID 6 的 MTTDL 对比
3. 给定 10mm² 面积预算，推荐一个 Cache 层次分配方案
