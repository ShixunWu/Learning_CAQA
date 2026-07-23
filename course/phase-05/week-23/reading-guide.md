# 第23周 阅读指南：向量处理器深入与阶段项目

> 对应教材：CAQA 6th Edition, Appendix G + 综合 Chapter 4

## 学习目标

1. 深入理解向量 ISA 设计的关键权衡
2. 理解向量化编译器的工作原理
3. 掌握 Scatter/Gather 指令在稀疏计算中的应用
4. 综合比较向量/SIMD/GPU 三种数据并行方案的优劣

## 核心概念

### 向量 ISA 设计空间（App G）

**寄存器设计**：
- 向量寄存器数量：8（VMIPS）→ 32（RVV）→ 更多（SX-Aurora）
- 向量长度：固定（Cray-1: 64）vs 可变（RVV: VLEN≥128）

**指令类型**：
- 算术：VV（向量-向量）、VS（向量-标量）、SVM（标量-向量-掩码）
- 访存：Unit-stride、Constant-stride、Indexed（Scatter/Gather）

**条件执行**：
- 向量掩码寄存器（VM）：控制哪些元素参与运算
- 替代 SIMD 的手动 blend/permute

### 向量化编译器

编译器自动向量化的关键步骤：
1. **依赖分析**：检测循环携带依赖（RAW/WAR/WAW）
2. **循环变换**：Loop Interchange、Loop Distribution、Loop Fusion
3. **向量化合法性**：无向后依赖（方向向量 ≥）
4. **Strip Mining**：自动生成循环分段代码

### Scatter/Gather

```
; 稀疏向量加法: C[idx[i]] = A[idx[i]] + B[idx[i]]
LV      V1, (Ra)          # 加载索引
LVI     V2, (Rb + V1)     # Gather: V2[i] = A[idx[i]]
LVI     V3, (Rc + V1)     # Gather: V3[i] = B[idx[i]]
ADDVV   V4, V2, V3
SVI     (Rd + V1), V4     # Scatter: C[idx[i]] = V4[i]
```

### 三种方案综合对比

| 维度 | 向量处理器 | x86 SIMD | GPU |
|------|-----------|----------|-----|
| 硬件开销 | 中（专用寄存器+流水线） | 低（扩展已有指令集） | 高（独立芯片） |
| 编程难度 | 低（编译器友好） | 中（intrinsics） | 高（显式管理层次） |
| 峰值性能 | 中 | 中高 | 极高 |
| 能效比 | 高 | 中 | 中 |
| 适用场景 | 科学计算 | 通用加速 | 大规模并行 |

## 关键计算

**向量化收益**（考虑 Strip Mining 开销）：
\[
\text{加速比} = \frac{n \times T_{scalar}}{ \lceil n/MVL \rceil \times T_{strip\_overhead} + n \times T_{element} }
\]

## 阅读任务

1. 精读 App G §G.1-G.3：向量 ISA 完整设计
2. 精读 App G §G.4-G.5：向量化编译器实现
3. 综合 Chapter 4 与 App G：建立完整的 DLP 知识图谱

## 思考题

1. 可变向量长度（RVV）相比固定长度（Cray）的优势是什么？
2. Scatter/Gather 在什么场景下能显著超越标量代码？
3. 如果让你设计下一代数据并行架构，你会借鉴三种方案各自的哪一点？
