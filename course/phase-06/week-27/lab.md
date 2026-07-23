# 第27周 实验：互连拓扑延迟与吞吐量建模

> 使用 Python 实现 Mesh、Torus、Butterfly 的延迟与吞吐量模型

## 实验 1：拓扑建模框架

```python
#!/usr/bin/env python3
"""互连网络拓扑建模与比较"""
import math
from abc import ABC, abstractmethod

class Topology(ABC):
    @abstractmethod
    def diameter(self):
        pass

    @abstractmethod
    def bisection_bandwidth(self, link_bw=1.0):
        pass

    @abstractmethod
    def avg_distance(self):
        pass

    @abstractmethod
    def zero_load_latency(self, hop_lat=1.0):
        """零负载延迟: 平均距离 × 每跳延迟"""
        return self.avg_distance() * hop_lat

class Mesh2D(Topology):
    def __init__(self, k):
        self.k = k  # k×k mesh

    @property
    def N(self):
        return self.k * self.k

    def diameter(self):
        return 2 * (self.k - 1)

    def bisection_bandwidth(self, link_bw=1.0):
        return self.k * link_bw  # 切中间需要 k 条链路

    def avg_distance(self):
        # 近似: 2k/3
        return 2 * self.k / 3

class Torus2D(Topology):
    def __init__(self, k):
        self.k = k

    @property
    def N(self):
        return self.k * self.k

    def diameter(self):
        return 2 * (self.k // 2)

    def bisection_bandwidth(self, link_bw=1.0):
        return 2 * self.k * link_bw  # 回绕边加倍

    def avg_distance(self):
        return self.k / 2  # 回绕减少一半

class Butterfly(Topology):
    def __init__(self, k, n):
        self.k = k  # radix
        self.n = n  # stages

    @property
    def N(self):
        return self.k ** self.n

    def diameter(self):
        return self.n

    def bisection_bandwidth(self, link_bw=1.0):
        return (self.N // 2) * link_bw

    def avg_distance(self):
        return self.n  # 全网络距离都是 n

class FatTree(Topology):
    def __init__(self, levels):
        self.levels = levels

    @property
    def N(self):
        return 2 ** self.levels

    def diameter(self):
        return 2 * self.levels

    def bisection_bandwidth(self, link_bw=1.0):
        return (self.N // 2) * link_bw  # 全对分带宽

    def avg_distance(self):
        return 2 * self.levels - 1
```

## 实验 2：对比分析

```python
# 比较不同规模的拓扑
def compare_topologies(sizes):
    print(f"{'Size':<8} {'Topo':<8} {'N':<8} {'Diam':<8} "
          f"{'BisBW':<10} {'AvgDist':<10} {'Latency':<10}")
    print("-" * 62)

    for k in sizes:
        topologies = [
            ("Mesh", Mesh2D(k)),
            ("Torus", Torus2D(k)),
        ]
        for name, topo in topologies:
            print(f"{k}x{k:<4} {name:<8} {topo.N:<8} "
                  f"{topo.diameter():<8} {topo.bisection_bandwidth():<10.0f} "
                  f"{topo.avg_distance():<10.2f} {topo.zero_load_latency():<10.2f}")

    # FatTree
    for levels in [3, 4, 5, 6]:
        ft = FatTree(levels)
        print(f"{2**levels:<8} {'FatTree':<8} {ft.N:<8} "
              f"{ft.diameter():<8} {ft.bisection_bandwidth():<10.0f} "
              f"{ft.avg_distance():<10.2f} {ft.zero_load_latency():<10.2f}")

compare_topologies([4, 8, 16, 32])
```

## 实验 3：饱和吞吐量模型

```python
def saturation_throughput(topo, link_bw, injection_rate):
    """
    计算给定注入率下的饱和吞吐量
    当注入率 × N × avg_distance / bisection_bw > 1 时，网络饱和
    """
    load_on_bisection = (injection_rate * topo.N * topo.avg_distance()
                         / topo.bisection_bandwidth(link_bw))
    is_saturated = load_on_bisection > 1.0
    effective_tput = min(injection_rate * topo.N,
                         topo.bisection_bandwidth(link_bw) / topo.avg_distance())
    return {
        "bisection_load": load_on_bisection,
        "saturated": is_saturated,
        "effective_tput": effective_tput
    }

# 测试: 16x16 Mesh, 25 GB/s 链路
mesh = Mesh2D(16)
for rate in [0.01, 0.05, 0.1, 0.2]:
    result = saturation_throughput(mesh, link_bw=25, injection_rate=rate)
    print(f"注入率={rate:.2f}: 对分负载={result['bisection_load']:.2f}, "
          f"饱和={result['saturated']}, 有效吞吐={result['effective_tput']:.0f} GB/s")
```

## 实验 4：死锁检测模拟

```python
def detect_cycle_dependency(routes):
    """
    检测路由表中的循环依赖（可能导致死锁）
    routes: {node: {dest: next_hop}}
    """
    # 构建通道依赖图
    edges = set()
    for src, table in routes.items():
        for dst, next_hop in table.items():
            if next_hop != dst:
                # 通道: src→next_hop
                edges.add((src, next_hop))

    # Tarjan SCC 检测循环
    def find_cycle(graph):
        visited = set()
        stack = []
        for node in graph:
            if node not in visited:
                if dfs(node, graph, visited, stack, set()):
                    return True
        return False

    def dfs(node, graph, visited, stack, rec_stack):
        visited.add(node)
        rec_stack.add(node)
        for neighbor in graph.get(node, []):
            if neighbor not in visited:
                if dfs(neighbor, graph, visited, stack, rec_stack):
                    return True
            elif neighbor in rec_stack:
                return True  # 发现循环
        rec_stack.remove(node)
        return False

    adj = {}
    for u, v in edges:
        adj.setdefault(u, []).append(v)

    if find_cycle(adj):
        print("警告: 检测到循环依赖 - 可能死锁!")
    else:
        print("无循环依赖 - 路由死锁安全")

# 示例: 维序路由在 2×2 Mesh 上的路由表 (无循环)
routes = {
    (0,0): {(1,1): (0,1)},
    (0,1): {(1,1): (1,1)},
}
detect_cycle_dependency(routes)
```

## 实验报告要求

1. 绘制 Mesh vs Torus 在 k=4,8,16,32 时的直径和对分带宽对比曲线
2. 计算 16×16 Mesh 何时达到饱和（假设均匀随机流量）
3. 比较 Butterfly (k=2, n=6) 与 8×8 Torus（同为 64 节点）的零负载延迟
