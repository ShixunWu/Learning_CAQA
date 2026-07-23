# 第25周 自测题：缓存一致性协议

> 闭卷完成，时间 30 分钟。状态转换与协议分析为主。

## 题目

### Q1：MSI 状态转换
Core0 持有 addrX 在 M 状态。Core1 对 addrX 执行 read。请列出完整的 MSI 状态转换序列和总线事务。

### Q2：MESI vs MSI
MESI 协议中，Core0 首次 load addrX（系统无其他持有者）后，cache line 进入什么状态？之后 Core0 store 同一地址，是否需要总线事务？MSI 协议下同样场景的行为有何不同？

### Q3：Snooping 广播次数
4 核 Snooping 系统，核心 A 执行一次 BusRdX。协议规定所有核都需 snoop。共产生多少次 snoop 响应？若 n 核系统，扩展关系是什么？

### Q4：目录协议消息数
8 核目录协议系统，仅 Core3 持有 addrX 的 S 副本。Core0 要对 addrX 写。目录需要向多少个核发失效消息？Snooping 协议下失效消息的总数是多少？

### Q5：False Sharing 问题
两个变量 `int x, y` 在同一 64B cache line。Core0 循环写 x，Core1 循环写 y。在 MSI 协议下，描述 cache line 在核间的迁移模式。每轮迭代需要多少总线事务？

### Q6：MOESI 中的 Owned 状态
MOESI 中，Core0 持有 M 状态，Core1 读 → Core0 降级到 O 状态。之后 Core2 也读。O 状态如何减少对主存带宽的压力？

### Q7：有限指针目录
有 4 个指针的有限目录，8 核系统。6 个核已共享 addrX（超过指针数）。第 6 个核加入时，目录如何处理？这种设计权衡了什么？

### Q8：协议选型
请为以下场景选择最合适的协议（MSI/MESI/MOESI + Snooping/Directory），并说明理由：
(a) 2 核嵌入式处理器
(b) 64 核服务器 CPU
(c) 256 核多 socket 系统

---

## 答案

### A1
初始：Core0 = M。Core1 read miss → BusRd 广播。
Core0 snoop → M→S（write-back 到内存），提供数据給 Core1。
Core1 进入 S。最终：Core0=S，Core1=S。
事务：BusRd + WriteBack。

### A2
MESI：Core0 load → **E 状态**（干净独占）。Core0 store → E→**M 静默**，**无总线事务**。
MSI：Core0 load → S 状态（不知独占）。Core0 store → 需要 **BusRdX**（多余的广播开销）。
E 状态省去了一次从 S 到 M 必须的总线事务。

### A3
4 核：3 个其他核各产生 1 次 snoop 响应 → **3 次响应**。
n 核：**n-1 次响应**（O(n) 扩展）。

### A4
目录协议：仅 Core3 持有 → 发 **1 条失效消息**。
Snooping：广播到所有 8 核 → 每条 snoop 1 次 → **7 条失效消息**（8 核各 snoop 一次，但仅持有者需失效）。
目录优势：精确失效，消息数正比于持有者数而非总核数。

### A5
Core0 写 x → BusRdX → 进入 M。Core1 写 y → BusRdX (同 cache line!) → Core0 I → Core1 M。
每轮来回：**2 次 BusRdX 事务**（每次 ping-pong）。
这是 False Sharing：两个独立变量共享同一 cache line，产生不必要的失效。

### A6
Core0 M → 当 Core1 读时 → Core0 降级到 O（仍持有脏数据）。
Core2 再读 → Core0（O 状态）直接提供数据，无需访问主存。
节省：**1 次主存访问**。O 状态充当 "脏数据提供者"，减少内存带宽压力。

### A7
目录指针溢出 → **广播失效所有共享者**（或淘汰其中一个共享者使其失效）。
权衡：有限指针节省了目录存储开销（O(n) → O(k)），但溢出时退化为类似广播的行为。

### A8
(a) **MESI + Snooping**：2 核时 Snooping 延迟最低，MESI 减少不必要的事务。
(b) **MESI + Directory**：64 核 Snooping 广播会饱和互连，目录可精确路由。
(c) **MOESI + Directory**：256 核多 socket 跨越 NUMA 域，O 状态可减少跨 socket 的内存访问，目录提供可扩展性。
