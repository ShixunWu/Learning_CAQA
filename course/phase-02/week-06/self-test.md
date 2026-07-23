# Week 6 自测

## Q1：组相联计算

16KB Cache，块大小 32B，4 路组相联。有多少组？

## Q2：替换策略

以下场景中，LRU、FIFO、Random 三种策略谁的命中率最差？
A) 访问序列：A, B, C, D, A, B, C, D, A, B, C, D（循环访问 2 路组相联 Cache）
B) 访问序列：A, B, C, D, E, F, G, H（一次性扫描）

## Q3：写策略

某程序 30% 的指令是 store。如果用 write-through，每条 store 产生 1 次额外写流量。改用 write-back 后，store 不再产生即时写流量。假设 miss rate 10%，且只有 dirty block 被替换时才写回（假设 50% 的替换块是 dirty）。write-back 相比 write-through 减少了多少写流量？

---

## 答案

### A1
块数 = 16KB / 32B = 512 块。组数 = 512 / 4 = **128 组**。

### A2
A) LRU 最差：循环长度为 4，Cache 只有 2 路，LRU 每次都替换掉即将被访问的。FIFO 同理。Random 反而可能偶尔命中。
B) 三者一样：一次性扫描，全部 miss，与替换策略无关。

### A3
Write-through 写流量：每 store 1 次写 → 30% × 总访存。
Write-back 写流量：仅在替换 dirty 块时写 → miss_rate × 50% × 总访存 = 10% × 50% = 5%。
减少：(30% - 5%) / 30% ≈ **83%**。
