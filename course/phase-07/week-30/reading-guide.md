# Week 30 阅读指南：领域专用架构（DSA）— 第一部分

> 教材章节：Ch7 §7.1-7.4 | 核心主题：DSA 动机、TPU 架构、脉动阵列、量化

## 学习目标

- 理解摩尔定律终结对计算机架构的影响与 DSA 兴起的必然性
- 掌握 TPU（Tensor Processing Unit）的架构设计与核心思想
- 理解脉动阵列（Systolic Array）的工作原理与计算模式
- 掌握量化（Quantization）的基本概念及其对精度/效率的权衡

## 核心概念

### 1. DSA 的动机：后摩尔时代的架构创新
- **摩尔定律放缓**：晶体管密度增长趋缓，功耗墙限制频率提升
- **Dennard 缩微定律失效**：电压不能随晶体管尺寸等比降低
- **Dark Silicon**：芯片上大量晶体管无法同时供电使用
- **DSA 答案**：针对特定领域定制硬件，获得 10-100× 能效提升

### 2. TPU v1 架构要点
- **设计目标**：加速神经网络推理（Inference），特别是矩阵乘法
- **核心组件**：
  - **矩阵乘法单元（MXU）**：256×256 脉动阵列，每个周期完成 65,536 次 MAC
  - **统一缓冲区（UB）**：24 MB SRAM，存储中间激活值
  - **权重 FIFO**：从 DDR3 预取权重到 MXU
  - **累加器**：32 位，支持大规模点积运算
- **性能**：92 TOPS（INT8），功耗约 75W

### 3. 脉动阵列（Systolic Array）
- **原理**：数据以有节奏的方式"脉动"流经处理单元阵列
- **结构**：2D PE 阵列，每个 PE 执行乘法并累加，结果向右/向下传递
- **优势**：高数据复用、规则互连、极高吞吐量
- **映射**：矩阵乘法 C = A × B，A 行广播，B 列广播

### 4. 量化技术
- **INT8 量化**：将 FP32 权重和激活值映射到 8 位整数
- **量化公式**：x_int = round(x_float / scale) + zero_point
- **精度损失**：INT8 推理通常 < 1% 精度下降
- **能效收益**：INT8 MAC 比 FP32 MAC 省 4-12× 能耗和面积

## 关键性能公式

| 公式 | 说明 |
|------|------|
| 吞吐量 = N² × f / cycles_per_mac | 脉动阵列 (N×N) 的理论吞吐量 |
| MAC 利用率 = actual_MAC / peak_MAC | 实际计算效率 |
| 量化 SNR ≈ 6.02 × bits + 1.76 dB | 量化信噪比（理想） |
| 能效比 = TOPS / Watt | 加速器关键指标 |

## 阅读建议

1. **精读顺序**：§7.1（DSA 动机）→ §7.3（TPU 架构）→ §7.4（脉动阵列细节）
2. **配套学习**：阅读 Google TPU ISCA 2017 论文原文
3. **对照思考**：TPU 的设计选择与传统 GPU/CPU 有何不同？为什么？

## 补充资源

- Jouppi et al., "In-Datacenter Performance Analysis of a Tensor Processing Unit", ISCA 2017
- Kung, H.T., "Why Systolic Architectures?", IEEE Computer 1982
