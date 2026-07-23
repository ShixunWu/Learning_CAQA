# Phase 6 阶段复习：线程级并行 (Thread-Level Parallelism)

> 覆盖教材：CAQA Chapter 5 + Appendix F + Appendix I，周 24-28

## 阶段测验（12 题，闭卷 45 分钟）

### Q1：Amdahl 定律（多核）
程序 92% 可并行化，在 16 核 SMP 上的理论加速比是多少？

### Q2：NUMA 延迟
4 节点 NUMA，本地 100ns，远程 350ns。程序 85% 本地访问。平均延迟？若调度优化使本地率达到 97%，延迟改善多少？

### Q3：MSI 状态转换
两核系统：Core0 地址 X 处于 M 状态，Core1 写 X。列出状态转换序列与总线事务。

### Q4：MESI 优势
Core0 首次加载地址 Y（其他核均无副本）。MESI vs MSI 下 Y 分别处于什么状态？若 Core0 随后写 Y，哪种协议产生更多总线事务？多多少？

### Q5：Snooping 可扩展性
n 核 Snooping 系统，每个 shared write miss 需发送 n-1 条 invalidation。每核每秒产生 2×10^6 次 shared write miss。互连最多处理 10^8 消息/秒。求最大 n。

### Q6：内存一致性
x=0, y=0。Core0: x=1; r0=y。Core1: y=1; r1=x。在 SC 和 x86-TSO 下，(r0,r1) 各自可能取值是什么？

### Q7：CAS 原子性
```c
do { old = *ptr; new = old + 1;
} while (!CAS(ptr, old, new));
```
若两个线程同时执行此代码，说明为什么至少一个 CAS 会失败。

### Q8：对分带宽计算
16×16 2D Torus（256 节点），每链路 50 GB/s 双向。对分带宽是多少？

### Q9：虫洞路由延迟
20-flit 消息，6-hop 路径，每 flit 0.5ns，路由延迟 3ns/hop。计算存储转发和虫洞路由延迟。

### Q10：Fat-Tree 设计
64 节点 Fat-Tree，每节点上行 1 条链路。要使对分带宽达到 32 条链路，每层的上行带宽倍数应如何设计？

### Q11：Dragonfly vs Mesh
10000 节点系统。Mesh (100×100) vs Dragonfly (100 组×100 节点，组内全连接)。计算两者的直径。

### Q12：综合：64 核方案选型
为 64 核处理器选择缓存一致性方案。从 Snooping + MESI vs Directory + MOESI 中选一个。给出两个定量理由。

---

## 答案

### A1
\( S = 1/(0.08 + 0.92/16) = 1/(0.08+0.0575) = 1/0.1375 \approx \mathbf{7.27\times} \)。

### A2
优化前：0.85×100 + 0.15×350 = 85+52.5 = **137.5ns**。
优化后：0.97×100 + 0.03×350 = 97+10.5 = **107.5ns**。
改善：137.5/107.5 ≈ **1.28×**（延迟降低 21.8%）。

### A3
Core0: M。Core1 write miss → BusRdX（请求独占+数据）。
Core0 snoop BusRdX → I（失效，若有脏数据则写回）。
Core1 进入 M。事务：**BusRdX + (可能的 WriteBack)**。

### A4
MESI：load → **E 状态**；store → **静默 M**，无总线事务。
MSI：load → S；store → 需 **BusRdX**。
MESI 比 MSI 少 **1 次 BusRdX 事务**（从 S 到 M 的那次）。

### A5
总消息 = n × 2×10^6 × (n-1) ≈ 2×10^6 × n² ≤ 10^8。
n² ≤ 50 → n ≤ **≈7 核**。Snooping 在 7 核以上就会饱和互连。

### A6
SC: (0,1), (1,0), (1,1)。**不可能 (0,0)**。
TSO: 以上三种 + **(0,0)**（因 store buffer 允许 load 越过 store）。

### A7
假设 `*ptr` 初始为 0。每个线程读 old=0，算出 new=1，同时执行 CAS。
硬件串行化 CAS：先到的 CAS 成功（*ptr=1），后到的 CAS 发现 *ptr≠old (1≠0) → **失败**。失败的 CAS 必须重试（读 new old=1, new=2, CAS 成功）。

### A8
Torus 对分带宽 = 2k × 50 = 2×16×50 = **1600 GB/s**（双向，每条链路计一次）。
若计双向每方向：**1600 GB/s**。

### A9
存储转发：6 × (20×0.5 + 3) = 6 × 13 = **78 ns**。
虫洞：head (6×(0.5+3)=21ns) + tail (19×0.5=9.5ns) = **30.5 ns**。

### A10
对分带宽 = 32 条链路。切中间时，N/2 条上行穿越对分线。
第 1 层：64 → 32 条（×1）
第 2 层：32 → 16 条（需 ×2 才能对分 = 32）→ 第 2 层链路宽需为 **2x**。
或等效：每层上行带宽翻倍导致对分处带宽 = 根层上行 × 层数/2。

### A11
Mesh 100×100：直径 = 2×(100-1) = **198 hops**。
Dragonfly：组内 1 + 跨组 1 + 组内 1 = **3 hops**。
Dragonfly 有极大的直径优势（198 vs 3）。

### A12
选 **Directory + MOESI**：
1. 64 核 Snooping 总线/互连广播消息量 = O(64²) ≈ 4000 消息/周期峰值 → 互连需 >10 TB/s 带宽；Directory 仅精确路由 O(持有者数)。
2. MOESI O 状态允许 dirty data forwarding 无需写回内存 → 减少 30-40% 内存带宽（对于 shared-memory 工作负载）。
