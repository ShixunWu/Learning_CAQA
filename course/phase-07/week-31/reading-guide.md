# Week 31 阅读指南：DNN 加速器深度解析

> 教材章节：Ch7 §7.5-7.8 | 核心主题：Eyeriss、DaDianNao、数据流、推理 vs 训练

## 学习目标

- 理解 DNN 加速器的三种核心数据流策略及其能耗特征
- 掌握 Eyeriss（MIT）和 DaDianNao（中科院）等代表性加速器架构
- 理解推理（Inference）与训练（Training）对加速器的不同需求
- 掌握数据复用效率的定量分析方法

## 核心概念

### 1. 三种数据流策略

DNN 计算的核心是嵌套循环：
```
for m in [0, M]:        # 输出通道
  for n in [0, N]:      # 输入通道  
    for e in [0, E]:    # 输出行
      for f in [0, F]:  # 输出列
        O[m][e][f] += I[n][e+r][f+s] × W[m][n][r][s]
```

| 数据流 | 固定不变的部分 | DRAM 访问优化目标 | 适用场景 |
|--------|---------------|-------------------|----------|
| **Weight Stationary** | 权重存在 PE 中 | 最小化权重读取 | 推理（权重固定） |
| **Output Stationary** | 部分和累加在 PE | 最小化部分和读写 | 小批次训练 |
| **Row Stationary** | 一行数据复用在 PE | 最大化所有类型复用 | Eyeriss 使用 |

### 2. Eyeriss 架构特点
- **MIT 2016 年设计**：首个展示 Row Stationary 数据流的芯片
- **核心创新**：将高维卷积映射到 2D PE 阵列，最大化数据复用
- **12×14 PE 阵列**：每个 PE 有独立的小型寄存器文件
- **能量效率**：相比 GPU 有 10-30× 能效提升

### 3. DaDianNao 架构
- **中科院计算所**：针对大规模神经网络的加速器
- **eDRAM 集成**：在芯片上集成大量 eDRAM 存储权重
- **多 chip 扩展**：通过 Node 互联实现多芯片扩展

### 4. 推理 vs 训练
| 特性 | 推理 | 训练 |
|------|------|------|
| 数据精度 | INT8/FP16 即可 | 通常需要 FP32/BF16 |
| 数据流 | 前向传播 | 前向+反向+权重更新 |
| 批次大小 | 1 或小批次 | 大批次（64-1024） |
| 权重 | 固定不变 | 持续更新 |
| 吞吐量延迟 | 延迟敏感 | 吞吐量优先 |

### 5. 数据复用分析
- **权重复用**：同一滤波器在输入特征图上滑动，被多次使用
- **输入复用**：同一输入激活值参与多个输出通道的计算
- **输出复用**：部分和在累加完成前保持在寄存器中

## 阅读建议

1. **重点**：§7.5 的数据流分类和能量分析模型
2. **对比阅读**：Eyeriss JSSC 2017 论文 vs 教材描述
3. **批判思考**：不同的数据流在实际 DNN 模型（如 ResNet、Transformer）上表现有何差异？

## 补充资源

- Chen et al., "Eyeriss: A Spatial Architecture for Energy-Efficient Dataflow for CNNs", ISCA 2016
- Chen et al., "DaDianNao: A Machine-Learning Supercomputer", MICRO 2014
