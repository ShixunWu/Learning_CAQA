# 第26周 阅读指南：同步与内存一致性模型

> 对应教材：CAQA 6th Edition, Chapter 5 §5.7-5.9

## 学习目标

1. 掌握硬件同步原语：Test-and-Set、CAS、LL/SC
2. 理解内存一致性模型：SC、TSO、RC
3. 能分析给定代码在不同内存模型下的可能输出

## 核心概念

### 硬件同步原语

**原子操作**：
- **Test-and-Set (TAS)**：读并写 1，返回旧值
- **Compare-and-Swap (CAS)**：`CAS(addr, old, new)` → 若 *addr==old，则 *addr=new
- **Load-Linked / Store-Conditional (LL/SC)**：
  - LL：加载值并设置监视位
  - SC：若监视位仍置位，则存储成功（否则失败）
  - RISC-V 的 lr.w / sc.w

**自旋锁（Spinlock）**：
```
lock:   TAS R1, lock_addr
        BNZ R1, lock    ; 自旋等待
        ; 临界区 ...
        ST  0, lock_addr ; 释放
```

### 内存一致性模型

内存一致性模型定义了**多线程程序中的访存操作何时对其他线程可见**。

#### 顺序一致性（SC — Sequential Consistency）

Lamport 定义：多处理器的执行结果与以下情况相同：
1. 所有核的操作以某种全序交错执行
2. 每个核内的操作保持程序顺序

**直观理解**：每个核的 store 对所有其他核同时可见。

#### 总存储顺序（TSO — Total Store Order）

x86 使用的模型：
- 每个核有 **Store Buffer**（写缓冲）
- Store 可延迟提交，Load 可旁路 Store Buffer
- 允许 "store→load 重排"（但 store→store, load→load 保持有序）

#### 宽松一致性（RC — Relaxed Consistency）

ARM/RISC-V 使用的模型：
- 默认不保证任何顺序
- 显式使用 **Fence/Barrier** 来约束顺序
- `FENCE` 指令：之前的所有访存完成后才能开始之后的访存

### 内存栅栏（Memory Fence）

| 指令 | 架构 | 含义 |
|------|------|------|
| `MFENCE` | x86 | 全部访存屏障 |
| `DMB` | ARM | 数据内存屏障 |
| `FENCE` | RISC-V | 可配置屏障 |
| `sync` | Power | 同步指令 |

### 经典并发模式

**Dekker 算法**（验证 SC）：
```
Core0:  flag0 = 1;  if (flag1 == 0) { /* 临界区 */ }
Core1:  flag1 = 1;  if (flag0 == 0) { /* 临界区 */ }
```
SC 下：至多一个核进入。TSO 下：两个核都可能进入（因 store 延迟可见）。

**Peterson 算法**：需要 SC 才能正确工作。

## 关键计算

**锁竞争开销**：
\[
T_{lock} = N_{spins} \times T_{cache\_miss} + T_{critical}
\]

## 阅读任务

1. 精读 §5.7：硬件同步原语的设计与实现
2. 精读 §5.8：内存一致性模型的动机与形式化定义
3. 精读 §5.9：实际处理器的内存模型（x86-TSO vs ARM-RC）

## 思考题

1. 为什么 SC 在硬件上代价昂贵？
2. x86 的 TSO 模型相比 SC 省了什么硬件开销？
3. 编写无锁数据结构时，为什么 ARM 上需要更多 fence 指令？
