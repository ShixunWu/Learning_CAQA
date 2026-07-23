# 第26周 自测题：同步与内存一致性模型

> 闭卷完成，时间 30 分钟。内存模型分析与并发正确性判断。

## 题目

### Q1：Dekker 算法分析
```
Core0:  x = 1;  r0 = y;
Core1:  y = 1;  r1 = x;
```
初始 x=0, y=0。在 SC 和 TSO 下，(r0, r1) 的可能取值分别是什么？

### Q2：TSO 与 Store Buffer
在 TSO 模型中，store buffer 允许 store 与后续 load 重排。举例说明这如何影响自旋锁的释放语义，以及为什么 x86 的 `lock` 前缀隐式包含 MFENCE。

### Q3：Fence 位置判断
以下消息传递代码在 ARM (RC) 上需要何处添加 fence 才能保证正确？
```
Thread0:  data = 42;     // (1)
          ready = 1;     // (2)
Thread1:  while (!ready); // (3)
          print(data);   // (4)
```

### Q4：CAS 无锁栈
```c
void push(Node **head, Node *node) {
    do {
        node->next = *head;
    } while (!CAS(head, node->next, node));
}
```
解释为什么这个 CAS 推送操作是无锁的（lock-free），以及为何它不需要额外的 fence（在 x86 上）。

### Q5：LL/SC vs CAS
ARM/RISC-V 用 LL/SC 而非 CAS 实现原子操作。LL/SC 相比 CAS 有什么优势？（提示：ABA 问题、实现复杂度）

### Q6：SC 的性能代价
SC 要求每个核的 store 对所有其他核立即可见。从硬件角度解释：（a）为什么这要求阻止 store→load 重排？(b) 这如何影响流水线性能？

### Q7：内存顺序选择
C11 的 memory_order 有 relaxed / acquire / release / seq_cst。为每个场景选择最弱但正确语义的 memory_order：
(a) 递增一个共享计数器（不依赖旧值）
(b) 生产者-消费者中的 flag 设置
(c) 临界区互斥锁

### Q8：给定代码，输出分析
```
x=0, y=0, z=0 (原子变量)
T0: x.store(1, relaxed);  y.store(1, release);
T1: while(y.load(acquire)==0);  z = x.load(relaxed);
T2: while(z==0);  print(x);
```
T1 中 z 的值可能是多少？T2 最终打印的 x 是多少？（假设所有线程最终完成）

---

## 答案

### A1
SC：(0,1), (1,0), (1,1)。不可能 (0,0)——两个线程不可能同时看到对方未写。
TSO：(0,1), (1,0), (1,1), **(0,0)**——因 store 缓冲，load 可在 store 提交前完成。

### A2
```
// 临界区
spin_lock(L): TAS L; loop
// ... critical ...
spin_unlock(L): L = 0;  // Store 但可能在 buffer 中!
```
若释放锁的 store 在 buffer 中，其他线程的 TAS 可能看不到——锁未真正释放。
x86 lock 前缀强制 store buffer 排空（隐式 MFENCE），保证释放语义。

### A3
Thread0 需要 (1) 和 (2) 之间加 fence：`data=42; fence; ready=1`。
Thread1 需要 (3) 和 (4) 之间加 fence：`while(!ready); fence; print(data)`。
没有 fence 时 ARM 可能对 store-store 和 load-load 重排。

### A4
CAS 是原子操作（硬件保证不可分割）。x86 上 CAS (`cmpxchg`) 带 lock 前缀，隐式具有 full barrier。
无锁：每个线程的失败不会阻塞其他线程的进展（与自旋锁不同，没有临界区）。

### A5
LL/SC 相比 CAS 的优势：
1. **实现更简单**：不需要复杂的比较-交换硬件，只需一个监视位（monitor bit），功耗和面积更优；
2. **可复合**：LL...任意计算...SC 模式允许在原子序列中做复杂计算（如 LL; add; SC），而 CAS 需要软件重试循环；
3. **无活锁风险**：SC 失败时只需重试，而 CAS 在高竞争下可能持续失败。
> 注意：LL/SC 并不能真正避免 ABA 问题（值从 A→B→A 仍可能使 SC 成功），需额外机制（如指针加版本号）来处理 ABA。

### A6
(a) 若允许 store→load 重排，load 可能在线程间的 store 生效前执行 → 破坏全序性。
(b) Store 必须等之前 store 全部完成 → 流水线阻塞（stall）。实际 CPU 用 SFENCE 或隐式 WMO 来避免全停顿，降低但牺牲 SC。

### A7
(a) **relaxed**：`fetch_add(1, relaxed)`——仅需原子自增，无顺序要求。
(b) **release/acquire** 对：写用 release，读用 acquire——确保数据在 flag 可见时也可见。
(c) **seq_cst** 或 **acquire/release**：锁获取用 acquire，释放用 release 足够；mutex 通常用 seq_cst 以保证全局顺序。

### A8
T1 中 z = x.load(relaxed) 在 y.load(acquire)==0 的 while 之后。acquire fence 保证 T1 看到 y==1 时也看到 T0 在 release 之前的 store（包括 x=1）。但 x.load 是 relaxed，acquire 已在上面的 while 中。z **一定为 1**。
T2 的 while(z==0) 之后 print(x)。z 是普通（非原子）变量？这里是伪代码——若 z 被 T1 写入后 T2 可见，则 x=1。**打印 1**。
若代码有歧义：实际需确保 z 对 T2 可见性。若 z 是普通 int，编译器和硬件可能重排/缓存它。但概念上，答案 = **1**。
