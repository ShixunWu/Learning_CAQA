# 第28周 阶段项目：64 核 NoC 缓存一致性方案设计

> 综合 Phase 6 所学，设计一款 64 核处理器的片上网络与缓存一致性方案

## 项目目标

设计一个 64 核处理器的完整 TLP 方案，包括：
1. NoC 拓扑选择与理由
2. 缓存一致性协议设计
3. 性能预估与瓶颈分析

## Part 1：约束与假设

- 64 个 RISC-V 核心，每核私有 L1 I/D (32KB each)
- 每 4 核共享 L2 (512KB)，共 16 个 L2 bank
- 共享 L3 (32MB)，分 8 个 bank，分布在各 tile
- 芯片面积约 400 mm²（7nm 工艺）
- 目标频率 3 GHz

## Part 2：拓扑选型分析

```python
#!/usr/bin/env python3
"""64核 NoC 拓扑选型分析"""
import math

def analyze_topology(name, nodes, links, diameter, bisection_bw,
                     link_power, router_power, total_power):
    """统一分析框架"""
    print(f"\n--- {name} ---")
    print(f"  节点数: {nodes}, 链路数: {links}")
    print(f"  直径: {diameter}, 对分带宽: {bisection_bw}")
    print(f"  链路功耗: {link_power:.1f}W, 路由器功耗: {router_power:.1f}W")
    print(f"  总NoC功耗: {total_power:.1f}W")

# 8×8 Mesh
k = 8
mesh_links = 2 * k * (k - 1)  # 112
mesh_diam = 2 * (k - 1)       # 14
mesh_bis = k                   # 8

# 8×8 Torus
torus_links = 2 * k * k        # 128
torus_diam = 2 * (k // 2)      # 8
torus_bis = 2 * k              # 16

# 4×4×4 Hypercube 近似 (CEB: Concentrated)
# 4 核/tile × 16 tiles: 4×4 Tile Mesh
tile_mesh_links = 2 * 4 * 3
tile_diam = 2 * 3

# Fat-Tree (集中式)
ft_links = 64 * 2  # 每叶 2 条上行链路
ft_diam = 2 * math.ceil(math.log2(64))
ft_bis = 32

analyze_topology("8×8 Mesh", 64, mesh_links, mesh_diam, mesh_bis,
                  mesh_links*0.5, 64*0.3, mesh_links*0.5+64*0.3)
analyze_topology("8×8 Torus", 64, torus_links, torus_diam, torus_bis,
                  torus_links*0.5, 64*0.4, torus_links*0.5+64*0.4)
analyze_topology("Fat-Tree", 64, ft_links, ft_diam, ft_bis,
                  ft_links*0.3, 32*0.6, ft_links*0.3+32*0.6)

print("\n=== 推荐方案 ===")
print("选择 8×8 Mesh + 目录协议（有限指针=4）")
print("理由: (1) 链路数最少 (112), 面积最小")
print("      (2) 直径 14 可接受 (平均 5.3 hops)")
print("      (3) 每 L2 bank 4 个目录指针覆盖典型共享模式")
```

## Part 3：协议设计

```python
#!/usr/bin/env python3
"""64核目录协议设计"""
from enum import Enum

class DirState(Enum):
    UNCACHED = 0  # 无核心缓存此块
    SHARED = 1    # 1-4 个核心在 S 状态
    EXCLUSIVE = 2 # 1 个核心在 E/M 状态

class DirectoryEntry:
    def __init__(self):
        self.state = DirState.UNCACHED
        self.sharers = set()    # 最多 4 个
        self.owner = -1         # E/M 所有者
        self.overflow = False   # 超过 4 个共享者 → 广播

class CoherenceProtocol:
    def __init__(self, n_cores):
        self.n_cores = n_cores
        self.directory = {}
        self.traffic_count = {"read_miss": 0, "write_miss": 0,
                              "invalidation": 0, "fwd_request": 0}

    def read_miss(self, core_id, addr):
        entry = self.directory.get(addr, DirectoryEntry())
        self.traffic_count["read_miss"] += 1

        if entry.state == DirState.UNCACHED:
            entry.state = DirState.EXCLUSIVE
            entry.owner = core_id
            return "MEMORY→CORE (E)"  # 数据来自内存

        elif entry.state == DirState.EXCLUSIVE:
            # 降级 owner 到 S, 发送数据给请求者
            owner = entry.owner
            entry.state = DirState.SHARED
            entry.sharers.add(owner)
            entry.sharers.add(core_id)
            self.traffic_count["fwd_request"] += 1
            return f"CORE{owner}→CORE{core_id} (S), owner→S"

        elif entry.state == DirState.SHARED:
            if len(entry.sharers) < 4:
                entry.sharers.add(core_id)
                return f"MEMORY→CORE{core_id} (S)"
            else:
                entry.overflow = True
                return f"MEMORY→CORE{core_id} (S, overflow)"

        self.directory[addr] = entry

    def write_miss(self, core_id, addr):
        entry = self.directory.get(addr, DirectoryEntry())
        self.traffic_count["write_miss"] += 1

        if entry.state == DirState.SHARED:
            # 失效所有共享者
            for s in entry.sharers:
                if s != core_id:
                    self.traffic_count["invalidation"] += 1

        elif entry.state == DirState.EXCLUSIVE:
            if entry.owner != core_id:
                self.traffic_count["invalidation"] += 1

        entry.state = DirState.EXCLUSIVE
        entry.owner = core_id
        entry.sharers.clear()
        self.directory[addr] = entry
        return f"CORE{core_id} in M"

    def print_stats(self):
        total = sum(self.traffic_count.values())
        print(f"总事务: {total}")
        for k, v in self.traffic_count.items():
            print(f"  {k}: {v} ({v/total*100:.1f}%)")

# 测试: 10 核读同一地址 → 写
proto = CoherenceProtocol(64)
for i in range(10):
    proto.read_miss(i, 0x1000)
proto.write_miss(5, 0x1000)
proto.print_stats()
```

## Part 4：可扩展性预估

```python
def scalability_model(n_cores, local_access=0.85, remote_overhead=3.0):
    """
    预估多核性能缩放
    local_access: 本地 L2 命中率
    remote_overhead: 远程访问延迟/本地延迟比
    """
    effective_latency = (local_access * 1.0 +
                         (1-local_access) * remote_overhead)
    ideal_speedup = n_cores
    actual_speedup = ideal_speedup / effective_latency
    efficiency = actual_speedup / n_cores * 100

    print(f"核数={n_cores}: 有效延迟={effective_latency:.2f}×, "
          f"加速比={actual_speedup:.1f}×, 效率={efficiency:.1f}%")
    return actual_speedup

for p in [1, 4, 16, 64]:
    scalability_model(p, local_access=1.0 - 0.02*(p-1))
```

## 实验报告要求

1. 给出最终方案：拓扑 + 协议 + 理由（< 500 字）
2. 计算 64 核下缓存一致性流量开销（假设 5% miss rate, 20% shared）
3. 与 16 核 Snooping 方案对比，论证 Directory 在 64 核时的必要性
