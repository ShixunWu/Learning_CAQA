# 第25周 实验：Python MSI Snooping 协议模拟器

> 实现一个多核 MSI 缓存一致性协议模拟器，追踪状态转换

## 实验目标

编写 Python 模拟器，模拟 4 核系统通过共享总线执行 MSI 协议的状态转换。

## 完整模拟器代码

```python
#!/usr/bin/env python3
"""MSI Snooping 缓存一致性协议模拟器"""

from enum import Enum
from dataclasses import dataclass

class State(Enum):
    M = "Modified"
    S = "Shared"
    I = "Invalid"

@dataclass
class CacheLine:
    tag: int = -1
    state: State = State.I

class Core:
    def __init__(self, core_id, cache_size=8):
        self.id = core_id
        self.cache = [CacheLine() for _ in range(cache_size)]
        self.stats = {"reads": 0, "writes": 0, "misses": 0,
                      "invalidations": 0}

class MSIBus:
    def __init__(self, n_cores, memory):
        self.cores = [Core(i) for i in range(n_cores)]
        self.memory = memory
        self.event_log = []

    def _cache_index(self, addr):
        return addr % len(self.cores[0].cache)

    def _find_owner(self, addr):
        """查找 M 状态的持有者"""
        for core in self.cores:
            line = core.cache[self._cache_index(addr)]
            if line.tag == addr and line.state == State.M:
                return core
        return None

    def _invalidate_others(self, requester_id, addr):
        """失效其他核心的副本"""
        for core in self.cores:
            if core.id == requester_id: continue
            line = core.cache[self._cache_index(addr)]
            if line.tag == addr and line.state != State.I:
                line.state = State.I
                core.stats["invalidations"] += 1

    def read(self, core_id, addr):
        core = self.cores[core_id]
        core.stats["reads"] += 1
        line = core.cache[self._cache_index(addr)]

        if line.tag == addr and line.state != State.I:
            self.event_log.append(f"  [HIT]  Core{core_id} read addr{addr} (state={line.state.name})")
            return self.memory[addr]

        core.stats["misses"] += 1
        owner = self._find_owner(addr)

        if owner and owner.id != core_id:
            # M → S downgrade
            oline = owner.cache[self._cache_index(addr)]
            oline.state = State.S
            self.event_log.append(f"  [BUS]  Core{owner.id} M→S downgrade for addr{addr}")

        line.tag = addr
        line.state = State.S
        self.event_log.append(f"  [MISS] Core{core_id} BusRd addr{addr} → S")
        return self.memory[addr]

    def write(self, core_id, addr, value):
        core = self.cores[core_id]
        core.stats["writes"] += 1
        line = core.cache[self._cache_index(addr)]
        self.memory[addr] = value

        if line.tag == addr and line.state == State.M:
            self.event_log.append(f"  [HIT]  Core{core_id} write addr{addr}={value} (M, silent)")
            return

        # BusRdX
        self._invalidate_others(core_id, addr)
        line.tag = addr
        line.state = State.M
        core.stats["misses"] += 1
        self.event_log.append(f"  [MISS] Core{core_id} BusRdX addr{addr}={value} → M")

    def print_stats(self):
        print("\n=== 统计 ===")
        for core in self.cores:
            c = core.stats
            miss_rate = c["misses"] / max(c["reads"]+c["writes"], 1) * 100
            print(f"Core{core.id}: {c['reads']}R {c['writes']}W "
                  f"{c['misses']}miss ({miss_rate:.1f}%) "
                  f"{c['invalidations']}inv")

# ====== 测试场景 ======
print("=== MSI 协议模拟: 4 核共享内存 ===")
mem = {i: 0 for i in range(16)}
bus = MSIBus(4, mem)

# 场景 1: 多核读同一地址 → 都应进入 S
print("\n--- 场景 1: 多核读共享 ---")
bus.read(0, 0)
bus.read(1, 0)
bus.read(2, 0)

# 场景 2: 写 → 失效其他副本
print("\n--- 场景 2: Core1 写 addr0 ---")
bus.write(1, 0, 42)
bus.read(0, 0)  # Core0 read miss → BusRd

# 场景 3: 回写同一地址
print("\n--- 场景 3: Core2 写 addr0 ---")
bus.write(2, 0, 99)
bus.read(1, 0)

# 场景 4: False Sharing
print("\n--- 场景 4: False Sharing (addr4 和 addr5 可能同 cache line) ---")
bus.write(0, 4, 1)
bus.write(1, 5, 2)  # 同一 cache line → 可能 ping-pong

bus.print_stats()

# 打印事件日志
print("\n=== 事件日志 ===")
for e in bus.event_log:
    print(e)
```

## 扩展任务

### 1. 添加 MESI 支持

```python
class State(Enum):
    M = "Modified"
    E = "Exclusive"  # 新增
    S = "Shared"
    I = "Invalid"

# 修改 read: Read miss 无其他共享者时 → E（而非 S）
# 修改 write: E → M 不发射 BusRdX（静默升级）
```

### 2. 协议状态验证

```python
def verify_coherence(bus):
    """验证一致性不变量"""
    for addr in bus.memory:
        m_count = sum(1 for c in bus.cores 
                     if c.cache[addr % 8].tag == addr 
                     and c.cache[addr % 8].state == State.M)
        if m_count > 1:
            print(f"错误: addr{addr} 在多个核心处于 M 状态!")
            return False
    print("一致性验证通过")
    return True
```

### 3. 统计分析

```python
def analyze_traffic(bus):
    """分析总线流量"""
    busrd = sum(1 for e in bus.event_log if "BusRd" in e)
    busrdx = sum(1 for e in bus.event_log if "BusRdX" in e)
    total = busrd + busrdx
    print(f"总线事务: BusRd={busrd}, BusRdX={busrdx}, 总计={total}")
    print(f"Snooping 广播总数: {total * len(bus.cores)}")
```

## 实验报告要求

1. 针对 3 种典型访存模式（只读共享、乒乓读写、生产者-消费者），画出 MSI 状态转换序列
2. 添加 MESI E 状态，重新运行场景 2，对比 MSI vs MESI 的总线事务数
3. 模拟 8 核时统计 invalidations 总量，分析 Snooping 的可扩展性瓶颈
