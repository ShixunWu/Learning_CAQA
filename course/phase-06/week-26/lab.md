# 第26周 实验：gem5 多核同步与内存一致性

> 在 gem5 多核环境中测试同步原语与内存一致性模型

## 实验环境

```bash
# 编译多线程同步测试
gcc -O2 -pthread -o sync_test sync_test.c
gcc -O2 -pthread -o producer_consumer pc_test.c
```

## 实验 1：自旋锁实现与测试

```c
// spinlock_test.c — TAS 自旋锁 vs pthread mutex
#include <pthread.h>
#include <stdio.h>
#include <stdatomic.h>
#define ITERS 1000000

// Test-and-Set 自旋锁
typedef atomic_flag spinlock_t;
#define SPIN_INIT ATOMIC_FLAG_INIT

void spin_lock(spinlock_t *l) {
    while (atomic_flag_test_and_set(l))
        ;  // 自旋
}
void spin_unlock(spinlock_t *l) {
    atomic_flag_clear(l);
}

spinlock_t my_lock = SPIN_INIT;
long counter = 0;

void *worker_spinlock(void *arg) {
    for (int i = 0; i < ITERS; i++) {
        spin_lock(&my_lock);
        counter++;
        spin_unlock(&my_lock);
    }
    return NULL;
}
```

## 实验 2：生产者-消费者（内存一致性测试）

```c
// pc_test.c — 生产者-消费者 (测试内存顺序)
#include <pthread.h>
#include <stdio.h>
#include <stdatomic.h>
#define BUF_SIZE 1024

int buffer[BUF_SIZE];
atomic_int data_ready = 0;  // 0 = 空, 1 = 有数据
atomic_int data_consumed = 0;

// 生产者
void *producer(void *arg) {
    for (int i = 0; i < 1000; i++) {
        buffer[i % BUF_SIZE] = i;         // (A)
        atomic_store_explicit(&data_ready, 1, memory_order_release); // (B)
        while (atomic_load_explicit(&data_consumed, memory_order_acquire) == 0)
            ;
        atomic_store(&data_consumed, 0);
    }
    return NULL;
}

// 消费者
void *consumer(void *arg) {
    int sum = 0;
    for (int i = 0; i < 1000; i++) {
        while (atomic_load_explicit(&data_ready, memory_order_acquire) == 0)
            ;
        sum += buffer[i % BUF_SIZE];      // (C) 确保看到 (A)
        atomic_store_explicit(&data_ready, 0, memory_order_release);
        // 信号生产者继续
        atomic_store(&data_consumed, 1);
    }
    printf("Consumer sum = %d\n", sum);
    return NULL;
}
```

## 实验 3：内存一致性模型对比脚本

```python
#!/usr/bin/env python3
"""模拟 SC vs TSO vs RC 下的程序输出"""
from itertools import permutations

def sc_possible_outcomes(prog):
    """
    SC: 所有操作的全局交织序, 保持每线程程序序
    prog = [(thread_id, [(label, action), ...]), ...]
    action: 'S'(store), 'L'(load), 格式 'var=value'
    """
    outcomes = set()
    # 简化: 生成所有合法交织
    # (此为学生需要完成的实现细节)
    return outcomes

def tso_possible_outcomes(prog):
    """
    TSO: 允许 store→load 重排 (每个线程内的 store 可能延迟)
    """
    # Store 可被推后到后续 load 之后
    outcomes = set()
    return outcomes

# Dekker 算法示例
# Thread 0: S(flag0=1); L(flag1)
# Thread 1: S(flag1=1); L(flag0)
print("=== Dekker 算法 ===")
print("SC 下: (L0=0, L1=1) 或 (L0=1, L1=0) 或 (L0=1, L1=1)")
print("     不可能: (L0=0, L1=0) — 两个都进入临界区")
print("TSO 下: (L0=0, L1=0) 可能发生! Store 被缓冲, Load 看到旧值")
print("      需要 MFENCE 在 store 与 load 之间来修复")

# 内存屏障效果
print("\n=== 添加 MFENCE ===")
print("Thread 0: S(flag0=1); MFENCE; L(flag1)")
print("Thread 1: S(flag1=1); MFENCE; L(flag0)")
print("MFENCE 清空 Store Buffer → TSO 行为接近 SC")
```

## 实验 4：CAS 无锁数据结构

```c
// cas_counter.c — CAS 无锁计数器
typedef atomic_ullong atomic_counter_t;

void cas_increment(atomic_counter_t *c) {
    unsigned long long old, new;
    do {
        old = atomic_load(c);
        new = old + 1;
    } while (!atomic_compare_exchange_weak(c, &old, new));
}

// 对比: 在 gem5 上测 CAS 自旋 vs TAS 自旋锁的吞吐量/延迟
```

## 实验报告要求

1. 在 gem5 双核上运行 spinlock 测试，记录每核平均每次临界区的周期数
2. 分析自旋锁的 cache line 迁移模式（从一致性协议角度）
3. 生产者-消费者：将 `memory_order_release/acquire` 改为 `memory_order_relaxed`，观察是否出现错误
4. 画出 Dekker 算法在 SC/TSO/RC 下的可行输出集合
