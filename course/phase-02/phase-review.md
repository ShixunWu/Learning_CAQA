# Phase 2 综合测验：存储层次

## 选择题（每题5分，共60分）

**1.** 直接映射 Cache 的主要缺点是：
A) 访问延迟最高  B) 冲突缺失最多  C) 容量最小  D) 功耗最大

**2.** 32KB Cache，块大小 64B，8 路组相联。共有多少组？
A) 32  B) 64  C) 128  D) 256

**3.** L1 hit=1c, miss=10%, L2 hit=10c, L2 local miss=20%, Mem=200c。AMAT=?
A) 3c  B) 5c  C) 6c  D) 8c

**4.** 以下哪个对减少 Conflict Miss 最有效？
A) 增加块大小  B) 增加相联度  C) 增加 Cache 容量  D) 使用 write-back

**5.** TLB 本质上是：
A) 一级 Cache  B) 页表项的 Cache  C) 分支预测器  D) 寄存器文件

**6.** HP 是 87.5%，MP=100c。访问 1000 次的总 penalties 是多少 cycles？
A) 12500  B) 1250  C) 8750  D) 125

## 分析题（每题20分）

**7.** 设计题：一个嵌入式 SoC 有 2MB SRAM budget。请设计 Cache 层次方案（L1/L2 分配），论证你的选择。考虑：嵌入式 workload 通常工作集小但延迟敏感。

**8.** 某程序 50% load, 20% store, 30% ALU。Cache 配置：write-back + write-allocate。计算 write-back 相比 write-through 的带宽节省。

---

## 答案

1. B
2. B (32KB/64B=512块, 512/8=64组)
3. C (1+0.1×(10+0.2×200)=6)
4. B
5. B
6. A (1000×(1-0.875)×100=12500)

**7.** 建议 L1=32KB(I)+32KB(D), L2=剩余~1.9MB。嵌入式对延迟敏感→L1不能太小。L2可统一。

**8.** Write-through: 20% store每次都写→20%额外写流量。Write-back: 仅在replace dirty时写。设miss rate 5%，50% dirty→额外写流量=5%×50%=2.5%。节省=20%-2.5%=17.5%总访存带宽。
