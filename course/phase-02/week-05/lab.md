# Week 5 实验：Python Cache 模拟器

---

## 实验：从零实现一个直接映射 Cache

### 代码框架

```python
class DirectMappedCache:
    def __init__(self, capacity_kb, block_size_b):
        self.block_size = block_size_b
        self.num_blocks = capacity_kb * 1024 // block_size_b
        self.tags = [None] * self.num_blocks
        self.valid = [False] * self.num_blocks
        self.hits = 0
        self.misses = 0

    def access(self, address):
        block_addr = address // self.block_size
        index = block_addr % self.num_blocks
        tag = block_addr // self.num_blocks

        if self.valid[index] and self.tags[index] == tag:
            self.hits += 1
            return True  # Hit
        else:
            self.misses += 1
            self.tags[index] = tag
            self.valid[index] = True
            return False  # Miss

# 测试：遍历一个数组
cache = DirectMappedCache(capacity_kb=8, block_size_b=64)
array_size = 1024 * 1024  # 1M integers = 4MB

for i in range(0, array_size, 8):  # stride=8 (32 bytes between accesses)
    cache.access(i * 4)

hit_rate = cache.hits / (cache.hits + cache.misses) * 100
print(f"Hits: {cache.hits}, Misses: {cache.misses}")
print(f"Hit Rate: {hit_rate:.1f}%")
```

### 任务

1. **改变步长**：将 stride 从 8 改为 1、2、4、16、64。观察命中率如何变化，解释原因。
2. **改变 Cache 大小**：将 capacity 改为 4KB、32KB、256KB。观察对命中率的影响。
3. **改变块大小**：将 block_size 改为 16B、128B。哪种应用场景受益？

### 思考

- 为什么 stride=1 时命中率最高？
- 多大的 stride 会让命中率降到最低？为什么？

---

## 交付物

1. 完整的 Cache 模拟器代码
2. 不同 stride 下的命中率变化曲线（用 matplotlib 画图）
3. 一句话解释：为什么 stride 影响命中率？
