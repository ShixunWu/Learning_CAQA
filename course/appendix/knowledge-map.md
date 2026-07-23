# 计算机架构知识图谱

> 基于 CAQA 6th Edition | 概念关系全览

## 总览

```
计算机架构 (Computer Architecture)
│
├── 性能基础 (Performance Foundations)
│   ├── Iron Law: CPU Time = IC × CPI × Cycle Time
│   │   ├── IC (指令数) → ISA 设计, 编译器优化
│   │   ├── CPI (每指令周期数) → 微架构, 流水线, 缓存
│   │   └── Cycle Time (周期时间) → 工艺技术, 微架构复杂度
│   ├── Amdahl 定律
│   │   ├── Speedup = 1 / ((1-f) + f/S)
│   │   └── 推论: 加速常见情况, 瓶颈决定上限
│   ├── 基准测试与评估
│   │   ├── SPEC CPU (整数/浮点)
│   │   ├── 报告指标: 均值, 几何均值, 调和均值
│   │   └── 陷阱: MIPS, MFLOPS 的误导性
│   └── 成本与功耗
│       ├── 动态功耗: P_dyn = α × C × V² × f
│       ├── 静态功耗: P_static = V × I_leak
│       └── Energy = Power × Time (关注能效而非仅功耗)
│
├── 指令集架构 (ISA)
│   ├── RISC (MIPS, ARM, RISC-V)
│   │   ├── Load/Store 架构
│   │   ├── 固定长度指令, 简单寻址模式
│   │   └── 大量通用寄存器 (32 GPR)
│   ├── CISC (x86)
│   │   ├── 变长指令, 复杂寻址模式
│   │   ├── 内存-内存操作
│   │   └── 内部翻译为 RISC-like μops
│   └── ISA 设计权衡
│       ├── 寄存器数量 vs 指令编码空间
│       ├── 寻址模式 vs 指令复杂度
│       └── 兼容性 vs 创新自由度
│
├── 流水线与 ILP (Pipelining & Instruction-Level Parallelism)
│   ├── 经典 5 级流水线: IF → ID → EX → MEM → WB
│   ├── 冒险 (Hazards)
│   │   ├── 结构冒险 → 资源复制
│   │   ├── 数据冒险 → 转发 (Forwarding/Bypassing)
│   │   │   ├── RAW (真相关)
│   │   │   ├── WAR (反相关) → 寄存器重命名
│   │   │   └── WAW (输出相关) → 寄存器重命名
│   │   └── 控制冒险 → 分支预测
│   ├── 动态调度
│   │   ├── Scoreboarding (CDC 6600, 1964)
│   │   └── Tomasulo 算法 (IBM 360/91, 1967)
│   │       ├── 保留站 (Reservation Stations)
│   │       ├── 公共数据总线 (CDB)
│   │       ├── 寄存器重命名 (隐式)
│   │       └── 乱序执行 (Out-of-Order)
│   ├── 分支预测
│   │   ├── 静态: Always Taken, BTFN
│   │   ├── 动态 1-bit / 2-bit 饱和计数器
│   │   ├── 两级自适应 (Two-Level): Gshare, Gselect
│   │   ├── 竞赛预测器 (Tournament/Hybrid)
│   │   ├── 返回地址栈 (RAS)
│   │   └── 间接分支预测
│   └── 推测执行 (Speculation)
│       ├── 硬件推测: ROB (Reorder Buffer)
│       ├── 软件推测: 编译器辅助
│       └── 安全影响: Spectre, Meltdown
│
├── 超标量与 VLIW (Superscalar & VLIW)
│   ├── 多发射: 每周期发射多条指令
│   ├── 超标量: 硬件动态调度 (x86, ARM Cortex-A)
│   │   ├── 发射宽度 (Issue Width): 2-8 way
│   │   ├── ROB 大小, 物理寄存器文件大小
│   │   └── 调度器 (Scheduler), 唤醒 & 选择逻辑
│   └── VLIW/EPIC: 编译器静态调度 (Itanium)
│       ├── 捆绑指令 (Bundle)
│       └── 谓词执行 (Predication)
│
├── 存储层次 (Memory Hierarchy)
│   ├── 局部性原理
│   │   ├── 时间局部性 → 缓存复用
│   │   └── 空间局部性 → 缓存行/预取
│   ├── 缓存 (Cache)
│   │   ├── 结构: 直接映射, 组相联, 全相联
│   │   ├── AMAT = Hit Time + Miss Rate × Miss Penalty
│   │   ├── 3C 模型: Compulsory, Capacity, Conflict
│   │   ├── 替换策略: LRU, RRIP, SRRIP, DIP, DRRIP
│   │   ├── 写策略: Write-Through vs Write-Back
│   │   │   ├── Write-Allocate vs No-Write-Allocate
│   │   │   └── Write Buffer
│   │   ├── 预取 (Prefetching)
│   │   │   ├── 硬件: Next-Line, Stride, GHB
│   │   │   └── 软件: Prefetch 指令
│   │   └── 多级缓存: L1 → L2 → L3 (LLC)
│   ├── 虚拟内存 (Virtual Memory)
│   │   ├── 页表 (Page Table) & 页表遍历 (PTW)
│   │   ├── TLB (Translation Lookaside Buffer)
│   │   ├── 大页 (Huge Pages): 2MB, 1GB
│   │   ├── 虚拟化: 嵌套页表 (EPT/NPT)
│   │   └── 页错误 (Page Fault) 处理
│   └── 存储与 I/O
│       ├── SSD (NAND Flash): FTL, 写放大
│       ├── HDD: 寻道时间, 旋转延迟
│       ├── RAID: 0/1/5/6
│       └── 可靠性: ECC, Chipkill, 擦除编码
│
├── 并行架构 (Parallel Architectures)
│   ├── SIMD & 向量
│   │   ├── SSE, AVX, AVX-512, SVE
│   │   ├── 向量长度 (VL), 向量寄存器
│   │   └── 向量化加速比: 依赖 vs 独立循环
│   ├── GPU 架构
│   │   ├── SIMT 模型: Warp/Wavefront
│   │   ├── 存储层次: 全局 → L2 → 共享内存 → 寄存器
│   │   ├── CUDA 编程: Grid, Block, Thread
│   │   ├── 优化: 合并访问, Bank Conflict, Occupancy
│   │   └── Tensor Core: 混合精度矩阵乘法
│   ├── 缓存一致性 (Cache Coherence)
│   │   ├── 协议: MSI → MESI → MOESI
│   │   ├── 实现: Snooping (总线) vs Directory (NoC)
│   │   ├── 目录: Full-Map, Limited, Chained
│   │   └── 状态转换 & 瞬态状态
│   ├── 内存一致性 (Memory Consistency)
│   │   ├── Sequential Consistency (SC)
│   │   ├── Total Store Order (TSO / x86)
│   │   ├── Relaxed (ARM, RISC-V RVWMO)
│   │   └── Fence/Barrier 指令
│   ├── 片上网络 (Network-on-Chip)
│   │   ├── 拓扑: Mesh, Torus, Flattened Butterfly, Crossbar
│   │   ├── 路由: Dimension-Order, Adaptive
│   │   ├── 流控: Wormhole, Virtual Channels
│   │   └── 指标: 延迟-吞吐量曲线, 零负载延迟
│   └── SMT (Simultaneous Multithreading)
│       ├── 细粒度 MT vs 粗粒度 MT vs SMT
│       ├── 资源共享: 静态分区 vs 动态竞争
│       └── 前端 vs 后端瓶颈
│
├── 现代架构话题 (Modern Architecture Topics)
│   ├── 仓库级计算机 (WSC)
│   │   ├── TCO = CapEx + NPV(OpEx)
│   │   ├── PUE = 总能耗 / IT 能耗
│   │   ├── 能量比例性 (Energy Proportionality)
│   │   ├── 尾延迟 (Tail Latency) & 对冲请求
│   │   └── 故障处理: MTTF, MTTR, 可用性
│   ├── 领域专用架构 (DSA)
│   │   ├── 动机: 摩尔定律放缓, Dark Silicon
│   │   ├── TPU: 256×256 脉动阵列, INT8
│   │   └── 量化: INT8 vs FP16 vs BF16 vs FP32
│   └── DNN 加速器
│       ├── 数据流: WS, OS, RS, NLR, N 输出
│       ├── Eyeriss (MIT): Row Stationary
│       ├── DaDianNao (中科院): eDRAM 集成
│       ├── 算术强度: ops/byte → Roofline
│       └── 推理 vs 训练需求差异
│
└── 历史与趋势 (History & Trends)
    ├── 大时代
    │   ├── 1960s: 大型机 (IBM S/360)
    │   ├── 1980s: RISC 革命 (MIPS, SPARC)
    │   ├── 1990s: 超标量乱序 (Pentium Pro)
    │   ├── 2005: 多核转折 (功耗墙)
    │   └── 2015+: DSA 兴起 (TPU, NPU)
    ├── 永恒原则
    │   ├── 加速常见情况
    │   ├── Amdahl 定律
    │   ├── 定量设计
    │   └── 层次化设计
    └── 未来方向
        ├── Chiplet & 2.5D/3D 封装
        ├── CXL (Compute Express Link)
        ├── Processing-in-Memory (PIM)
        └── 量子计算 & 神经形态计算
```

## 核心公式速查

| 类别     | 公式                                        | 意义                  |
| -------- | ------------------------------------------- | --------------------- |
| 性能     | CPU Time = IC × CPI × T_cycle             | Iron Law              |
| 加速比   | Speedup = 1 / ((1-f) + f/S)                 | Amdahl 定律           |
| 动态功耗 | P_dyn = α C V² f                          | 频率/电压对功耗的影响 |
| 缓存     | AMAT = Hit Time + Miss Rate × Miss Penalty | 平均访存时间          |
| 一致性   | M → S (BusRd), S → M (BusRdX)             | MSI 协议              |
| 算术强度 | AI = FLOPs / Bytes                          | Roofline 模型         |
| PUE      | PUE = P_total / P_IT                        | 数据中心能效          |
| 可用性   | A = MTTF / (MTTF + MTTR)                    | 系统可靠性            |

## 使用方法

1. **学习导航**：从上到下按层次学习，同一层次的内容可并行学习
2. **复习检查**：随机选取一个节点，检查自己能否解释其上下层关系
3. **面试准备**：任一叶子节点都是潜在的面试题
