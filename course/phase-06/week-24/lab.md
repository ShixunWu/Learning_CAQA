# 第24周 实验：gem5 多核仿真与 NUMA 效应

> 使用 gem5 模拟器配置双核/四核系统，观察多线程性能与 NUMA 效应

## 实验环境

```bash
# 安装 gem5 (若未安装)
git clone https://github.com/gem5/gem5.git
cd gem5
scons build/X86/gem5.opt -j$(nproc)
```

## 实验 1：配置 gem5 双核 SMP 系统

```python
# dualcore_smp.py — 双核 SMP 配置
import m5
from m5.objects import *

system = System()
system.clk_domain = SrcClockDomain(clock='3GHz', voltage_domain=VoltageDomain())
system.mem_mode = 'timing'
system.mem_ranges = [AddrRange('2GB')]

# 两个 TimingSimpleCPU
system.cpu = [TimingSimpleCPU() for _ in range(2)]

# 共享总线
system.membus = SystemXBar()

# 每核私有 L1
for i, cpu in enumerate(system.cpu):
    cpu.icache = L1ICache(size='32kB')
    cpu.dcache = L1DCache(size='64kB')
    cpu.icache_port = system.membus.cpu_side_ports
    cpu.dcache_port = system.membus.cpu_side_ports
    cpu.createInterruptController()

# 共享 L2
system.l2bus = SystemXBar()
system.l2cache = L2Cache(size='512kB')
system.l2cache.cpu_side = system.l2bus.cpu_side_ports
system.l2cache.mem_side = system.membus.mem_side_ports
system.system_port = system.membus.cpu_side_ports

# 内存控制器
system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR3_1600_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports

root = Root(full_system=False, system=system)
m5.instantiate()
m5.simulate()
```

## 实验 2：多线程 Benchmark

```bash
#!/bin/bash
# 运行 PARSEC/Splash 风格的共享内存 benchmark
# 使用 gem5 SE 模式运行多线程程序

# 编译多线程测试
gcc -O2 -pthread -o mc_bench mc_bench.c

# 2 线程在 gem5 上运行
build/X86/gem5.opt dualcore_smp.py \
    --cmd="./mc_bench" --options="2"

# 4 线程
build/X86/gem5.opt quadcore_smp.py \
    --cmd="./mc_bench" --options="4"
```

多线程测试程序：

```c
// mc_bench.c — 多线程共享计数器竞争
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#define ITERS 10000000

volatile long shared_counter = 0;  // 无锁竞争

void *worker(void *arg) {
    for (long i = 0; i < ITERS; i++)
        shared_counter++;  // 触发缓存一致性流量
    return NULL;
}

int main(int argc, char *argv[]) {
    int nthreads = atoi(argv[1]);
    pthread_t *threads = malloc(nthreads * sizeof(pthread_t));
    for (int i = 0; i < nthreads; i++)
        pthread_create(&threads[i], NULL, worker, NULL);
    for (int i = 0; i < nthreads; i++)
        pthread_join(threads[i], NULL);
    printf("counter=%ld\n", shared_counter);
}
```

## 实验 3：NUMA 延迟模拟

```python
#!/usr/bin/env python3
"""NUMA 延迟模型：模拟本地 vs 远程访问性能差异"""
import numpy as np

def simulate_numa(n_cores, local_lat=100, remote_lat=300, 
                  local_ratio=0.9):
    """
    计算 NUMA 系统的平均内存延迟
    n_cores: 总核数
    local_lat: 本地访问延迟 (ns)
    remote_lat: 远程访问延迟 (ns)
    local_ratio: 本地访问比例（取决于调度质量）
    """
    avg_lat = local_ratio * local_lat + (1 - local_ratio) * remote_lat
    print(f"核数={n_cores}, 本地比={local_ratio:.0%}: "
          f"平均延迟={avg_lat:.0f}ns")
    return avg_lat

def scalability_analysis(p, f_values):
    """Amdahl 定律可扩展性分析"""
    print(f"\n{'F':>5}  ", end="")
    for p_cores in [2,4,8,16,32]:
        print(f"P={p_cores:<5}", end=" ")
    print()
    for f in f_values:
        print(f"{f:>5.1%} ", end="")
        for p_cores in [2,4,8,16,32]:
            speedup = 1.0 / ((1-f) + f/p_cores)
            print(f"{speedup:<5.1f}", end=" ")
        print()

# 分析不同 NUMA 调度质量
print("=== NUMA 延迟分析 ===")
for ratio in [0.99, 0.90, 0.75, 0.50]:
    simulate_numa(8, local_ratio=ratio)

# Amdahl 扩展性
print("\n=== Amdahl 可扩展性 ===")
scalability_analysis(32, [0.99, 0.95, 0.90, 0.80])
```

## 实验 4：cache 一致性流量观测

```python
#!/usr/bin/env python3
"""估计缓存一致性协议的流量开销"""
def coherence_traffic(n_cores, miss_rate=0.05, 
                      shared_fraction=0.3, cache_line=64):
    """
    估算多核一致性流量
    每 1000 次访存中的一致性消息数
    """
    accesses = 1000
    misses = accesses * miss_rate
    shared_misses = misses * shared_fraction
    # 每次 shared miss → invalidations(N-1) + response
    traffic = shared_misses * (n_cores - 1 + 1) * cache_line
    print(f"核数={n_cores}: 每1000次访存产生 {traffic/1024:.1f}KB 一致性流量")
    return traffic

for p in [2, 4, 8, 16]:
    coherence_traffic(p)
```

## 实验报告要求

1. 记录不同核心数下 `mc_bench` 的执行时间与加速比
2. 解释为何无锁竞争下 `shared_counter++` 的加速比可能低于 1（甚至慢于单核）
3. NUMA 模拟：讨论 "本地比例" 从 99% 降到 50% 对系统性能的影响
