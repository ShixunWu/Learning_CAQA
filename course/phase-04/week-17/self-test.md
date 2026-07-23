# Week 17 自测：VLIW 与 EPIC

## Q1：VLIW 代码膨胀

VLIW 处理器有 4 个 slot（2 ALU, 1 FPU, 1 MEM）。一段代码编译后平均每个 bundle 利用 2.5 个 slot。代码膨胀率（相比单发射）是多少？NOP 占比是多少？

## Q2：VLIW vs 超标量

以下场景 VLIW 和超标量哪个更有优势？
A) 循环展开后的密集数值计算
B) 包含大量分支的业务逻辑代码
C) 嵌入式低功耗设备上的信号处理

## Q3：EPIC 的谓词执行

IA-64 的谓词执行如何消除分支？将以下 if-else 转换为谓词形式：
```c
if (a > b) x = c + d; else x = e - f;
```
假设有 `cmp.gt p1,p2 = a,b` 指令（p1 为真谓词，p2 为假谓词）。

## Q4：VLIW 的 Cache Miss 问题

VLIW 处理器发生 L1 cache miss 时，为什么整个 bundle 都被阻塞？超标量如何处理类似情况？

## Q5：调度器计算

VLIW 有 2 ALU（1c）、1 MUL（3c）。调度以下依赖链：
```
I0: ALU result=R1 (1c)
I1: MUL R1, R2 → R3 (3c, 依赖 I0)
I2: ALU R3, R4 → R5 (1c, 依赖 I1)
I3: ALU - (独立)
I4: ALU - (独立)
```
画出各周期 bundle 内容，计算总周期数和 slot 利用率。

---

## 答案

### A1
- 单发射：每条指令 = 1 指令宽度
- VLIW：每 bundle = 4 slots，利用率 = 2.5/4 = 62.5%
- 代码膨胀率 = 4（每条 bundle 相当于 4 条单发射指令的宽度）
  - 实际有效指令密度 = 2.5/bundle，单发射 = 1/指令
  - 膨胀率 = 4/2.5 = 1.6x（即 VLIW 代码体积是单发射的 1.6 倍）
- NOP 占比 = (4-2.5)/4 = 37.5%

### A2
A) VLIW：循环展开后 ILP 高且可预测，编译器能有效调度
B) 超标量：分支多且不可预测，硬件动态调度更灵活
C) VLIW：低功耗是关键优势，信号处理有大量可预测并行性

### A3
```asm
cmp.gt p1, p2 = a, b    ; p1 = (a>b), p2 = !(a>b)
(p1) add x = c, d       ; 仅当 p1=1 时执行
(p2) sub x = e, f       ; 仅当 p2=1 时执行
```
两条指令可放入同一个 VLIW bundle，无分支跳转。

### A4
VLIW 中一个 bundle 的所有 slot 在同一周期执行。当 MEM slot 的 load 发生 cache miss 时，整个流水线（包括 ALU 和 FPU slot）都 stall——因为下个 bundle 必须等当前 bundle 全部完成。超标量中，不同指令独立发射，load 的 cache miss 不一定阻塞其他不依赖该 load 的指令（通过乱序执行继续推进）。

### A5
C1: [I0:ALU, I3:ALU, -] → 2/3 slot
C2: [I4:ALU, -, -] → 1/3 slot
C3: [I1:MUL, -, -] → 1/3 slot
C4-5: MUL 执行中 → 0/3
C6: [I2:ALU, -, -] → 1/3
总 6 周期，利用率 = (2+1+1+0+0+1)/(6×3) = 5/18 ≈ 27.8%
