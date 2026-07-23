# Week 6 实验：扩展 Cache 模拟器

## 任务：支持组相联和多种替换策略

在 Week 5 的直接映射 Cache 基础上扩展：

```python
class SetAssociativeCache:
    def __init__(self, capacity_kb, block_size_b, ways):
        self.block_size = block_size_b
        self.ways = ways
        self.num_sets = capacity_kb * 1024 // (block_size_b * ways)
        # 每个 set 是一个列表，包含 ways 个 [tag, valid, last_used_time]
        self.sets = [[[None, False, 0] for _ in range(ways)]
                      for _ in range(self.num_sets)]
        self.time = 0

    def find_way(self, set_idx, tag):
        for w in range(self.ways):
            if self.sets[set_idx][w][1] and self.sets[set_idx][w][0] == tag:
                return w
        return -1

    def access(self, address, write=False):
        block_addr = address // self.block_size
        set_idx = block_addr % self.num_sets
        tag = block_addr // self.num_sets
        self.time += 1

        way = self.find_way(set_idx, tag)
        if way >= 0:
            self.sets[set_idx][way][2] = self.time  # 更新时间戳(LRU)
            return True  # Hit
        else:
            # 找一个空way，或用LRU替换
            target = None
            for w in range(self.ways):
                if not self.sets[set_idx][w][1]:
                    target = w; break
            if target is None:  # 全满，LRU替换
                target = min(range(self.ways),
                           key=lambda w: self.sets[set_idx][w][2])
            self.sets[set_idx][target] = [tag, True, self.time]
            return False
```

### 任务

1. 用同一个地址序列，测试 ways=1/2/4/8 时的命中率
2. 修改替换策略为 FIFO（用插入顺序替代 LRU），对比差异

---

## 交付物

1. 完整的组相联 Cache 代码
2. 不同路数下的命中率对比图
