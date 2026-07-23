# 第28周 阅读指南：大规模多处理器与阶段项目

> 对应教材：CAQA 6th Edition, Appendix I + 综合 Chapter 5

## 学习目标

1. 了解大规模多处理器系统（MPP、Cray、BlueGene、集群）的架构演进
2. 理解 TOP500 中的性能趋势与技术变革
3. 综合 Phase 6 知识设计完整的多核缓存一致性方案

## 核心概念

### 大规模多处理器分类

**MPP (Massively Parallel Processor)**：
- 数千至数万处理器，分布式内存，专用互连
- 例：Cray T3E（1995, 2048 核 Alpha）
- 访问模型：消息传递（MPI），非共享地址空间

**Cluster（集群）**：
- 商用服务器+商用互连（InfiniBand/以太网）
- Beowulf 集群（1994）：廉价 PC 通过以太网组成
- 优势：性价比高、可增量扩展

**BlueGene 系列**：
- IBM BlueGene/L（2004）：65536 核，PowerPC 440，3D Torus
- BlueGene/Q（2012）：18 核/节点，5D Torus
- 极致能效比（GFLOPS/W）设计哲学

**Cray 系统**：
- Cray X1（2003）：向量+MPP 混合
- Cray XK7（2012）：CPU+GPU 混合（Titan, 18,688 K20x GPU）
- Cray XC 系列：Dragonfly 拓扑（分层全连接）

### TOP500 趋势

| 年代 | #1 系统 | 性能 | 架构特征 |
|------|---------|------|---------|
| 1993 | CM-5 | 131 GFLOPS | MPP |
| 2008 | Roadrunner | 1 PFLOPS | 混合 CPU+Cell |
| 2012 | Titan | 17.6 PFLOPS | CPU+GPU |
| 2018 | Summit | 148 PFLOPS | CPU+GPU (NVLINK) |
| 2022 | Frontier | 1.1 EFLOPS | CPU+GPU (HPE Cray) |

### 大规模系统设计权衡

1. **强缩放 vs 弱缩放**：固定问题 vs 问题随核数增大
2. **紧耦合 vs 松耦合**：共享内存 vs 消息传递
3. **胖节点 vs 瘦节点**：每节点多核多内存 vs 每节点少量核
4. **同构 vs 异构**：纯 CPU vs CPU+加速器

### 集群互连技术

- **InfiniBand**：低延迟 (~1μs)、高带宽 (200 Gb/s HDR)
- **Cray Slingshot**：Dragonfly + 自适应路由，200 Gb/s
- **NVIDIA NVLINK**：GPU-GPU 直连，900 GB/s

### 片上网络（NoC）

- 多核芯片内部互连
- 常用拓扑：2D Mesh（Tilera 64 核）, Ring（Intel Xeon Phi）, Crossbar（小规模）

## 关键计算

**强缩放加速比**（固定问题大小）：
\[
S_{strong} = \frac{T_1}{T_P}
\]

**弱缩放加速比**（固定每核问题大小）：
\[
S_{weak} = \frac{T_1(P)}{T_P(P)} = \frac{P \times T_{per\_core}}{T_P}
\]

## 阅读任务

1. 精读 App I §I.1-I.3：大规模系统架构
2. 精读 App I §I.4：集群与云计算
3. 浏览 top500.org 最新榜单，分析前 10 系统的架构共性

## 思考题

1. Cluster 相比 MPP 的核心优势是什么？
2. Dragonfly 拓扑如何在大规模下同时实现低直径和高对分带宽？
3. Frontier 达到 Exascale，其功耗约 21MW。计算其能效比 (GFLOPS/W) 并与 10 年前的系统对比，分析趋势。
