# Week 31 实验：DNN 加速器 MAC 阵列与数据流仿真

## 实验目标

用 Python 建模 DNN 加速器的 MAC 阵列，分析不同数据流策略下的
数据复用效率和存储器访问次数。

## 实验背景

DNN 加速器的能耗主要由数据搬运决定（通常占总能耗的 60-80%）。
不同的数据流策略显著影响 DRAM 和 SRAM 的访问次数。

## 任务：数据流效率分析器

```python
"""
DNN Accelerator Dataflow Simulator
分析 Weight Stationary, Output Stationary, Row Stationary 三种数据流
"""
import numpy as np
from dataclasses import dataclass
from enum import Enum

class Dataflow(Enum):
    WEIGHT_STATIONARY = "WS"
    OUTPUT_STATIONARY = "OS"
    ROW_STATIONARY = "RS"

@dataclass
class EnergyModel:
    """各级存储器访问能耗 (pJ)"""
    dram_read: float = 200.0     # DRAM 读 (pJ/byte)
    dram_write: float = 200.0    # DRAM 写 (pJ/byte)
    sram_read: float = 5.0       # 片上 SRAM 读 (pJ/byte)
    sram_write: float = 5.0      # 片上 SRAM 写 (pJ/byte)
    rf_read: float = 1.0         # 寄存器文件读 (pJ/byte)
    mac_op: float = 0.5          # MAC 计算 (pJ/op)
    data_bytes: int = 2          # 默认 FP16 = 2 bytes

@dataclass
class ConvLayerConfig:
    """卷积层参数"""
    H: int  # 输入高度
    W: int  # 输入宽度
    C: int  # 输入通道
    K: int  # 输出通道
    R: int  # 滤波器高度
    S: int  # 滤波器宽度
    stride: int = 1
    batch: int = 1

class MACArray:
    """MAC 阵列 + 数据流分析器"""
    
    def __init__(self, rows: int, cols: int, energy: EnergyModel):
        self.rows = rows
        self.cols = cols
        self.energy = energy
        self.access_stats = {}
    
    def analyze_conv2d(self, conv: ConvLayerConfig, df: Dataflow) -> dict:
        """
        分析给定卷积层在特定数据流下的访存特征
        返回能耗分解和访存次数
        """
        pad = 0
        H_out = (conv.H + 2*pad - conv.R) // conv.stride + 1
        W_out = (conv.W + 2*pad - conv.S) // conv.stride + 1
        
        # 数据量 (bytes)
        input_size = conv.H * conv.W * conv.C * self.energy.data_bytes * conv.batch
        weight_size = conv.K * conv.C * conv.R * conv.S * self.energy.data_bytes
        output_size = H_out * W_out * conv.K * self.energy.data_bytes * conv.batch
        
        total_macs = conv.batch * conv.K * conv.C * conv.R * conv.S * H_out * W_out
        
        if df == Dataflow.WEIGHT_STATIONARY:
            stats = self._weight_stationary(conv, input_size, weight_size, 
                                             output_size, total_macs)
        elif df == Dataflow.OUTPUT_STATIONARY:
            stats = self._output_stationary(conv, input_size, weight_size, 
                                             output_size, total_macs)
        elif df == Dataflow.ROW_STATIONARY:
            stats = self._row_stationary(conv, input_size, weight_size, 
                                          output_size, total_macs)
        else:
            raise ValueError("Unknown dataflow")
        
        self.access_stats[df.value] = stats
        return stats
    
    def _weight_stationary(self, conv, input_sz, weight_sz, output_sz, macs):
        """Weight Stationary: 权重固定在各 PE 中"""
        # 权重只从 DRAM 加载一次
        weight_dram_reads = weight_sz
        
        # 输入激活值通过 PE 阵列广播
        # 每个输入被 C 个通道使用，复用在 PE 间
        input_dram_reads = input_sz
        
        # 部分和累加后写回 DRAM
        output_dram_writes = output_sz
        
        dram_energy = (weight_dram_reads + input_dram_reads) * self.energy.dram_read \
                      + output_dram_writes * self.energy.dram_write
        mac_energy = macs * self.energy.mac_op
        
        # 估计输入在 SRAM 间的数据搬移（广播浪费）
        sram_moves = input_sz * conv.K  # 输入广播到 K 个 PE
    
        sram_energy = sram_moves * self.energy.sram_read
        
        total_energy = dram_energy + sram_energy + mac_energy
        
        return {
            "total_macs": macs,
            "dram_reads_bytes": weight_dram_reads + input_dram_reads,
            "dram_writes_bytes": output_dram_writes,
            "dram_energy_pJ": dram_energy,
            "sram_energy_pJ": sram_energy,
            "mac_energy_pJ": mac_energy,
            "total_energy_pJ": total_energy,
            "energy_per_mac_pJ": total_energy / macs,
            "arithmetic_intensity": macs / (weight_dram_reads + input_dram_reads + output_dram_writes),
        }
    
    def _output_stationary(self, conv, input_sz, weight_sz, output_sz, macs):
        """Output Stationary: 部分和固定在各 PE 中累加"""
        input_dram_reads = input_sz
        weight_dram_reads = weight_sz * conv.batch  # 每个样本需重新加载权重
        output_dram_writes = output_sz
        
        dram_energy = (input_dram_reads + weight_dram_reads) * self.energy.dram_read \
                      + output_dram_writes * self.energy.dram_write
        mac_energy = macs * self.energy.mac_op
        
        total_energy = dram_energy + mac_energy
        total_dram = input_dram_reads + weight_dram_reads + output_dram_writes
        
        return {
            "total_macs": macs,
            "dram_reads_bytes": input_dram_reads + weight_dram_reads,
            "dram_writes_bytes": output_dram_writes,
            "dram_energy_pJ": dram_energy,
            "mac_energy_pJ": mac_energy,
            "total_energy_pJ": total_energy,
            "energy_per_mac_pJ": total_energy / macs,
            "arithmetic_intensity": macs / total_dram,
        }
    
    def _row_stationary(self, conv, input_sz, weight_sz, output_sz, macs):
        """Row Stationary: Eyeriss 风格，最大化 1D 复用"""
        # Row Stationary 最大化所有类型的数据复用
        # 1D row of filter + 1D row of ifmap → 1D row of psum
        
        # 相比 WS/OS，RS 通过寄存器级别的复用显著减少 SRAM 访问
        reuse_factor = conv.R * conv.S  # 权重在 PE 内的复用
        
        input_dram_reads = input_sz * 0.5   # 每个输入被多次复用，DRAM 减半
        weight_dram_reads = weight_sz * 0.7  # 权重复用减少 DRAM 访问
        output_dram_writes = output_sz
        
        dram_energy = (input_dram_reads + weight_dram_reads) * self.energy.dram_read \
                      + output_dram_writes * self.energy.dram_write
        
        # RS 的 RF 访问更多，但 SRAM 访问更少
        rf_access = macs * 3 * self.energy.data_bytes  # 3 个操作数
        rf_energy = rf_access * self.energy.rf_read
        
        mac_energy = macs * self.energy.mac_op
        
        total_energy = dram_energy + rf_energy + mac_energy
        
        return {
            "total_macs": macs,
            "dram_reads_bytes": input_dram_reads + weight_dram_reads,
            "dram_writes_bytes": output_dram_writes,
            "dram_energy_pJ": dram_energy,
            "rf_energy_pJ": rf_energy,
            "mac_energy_pJ": mac_energy,
            "total_energy_pJ": total_energy,
            "energy_per_mac_pJ": total_energy / macs,
            "arithmetic_intensity": macs / (input_dram_reads + weight_dram_reads + output_dram_writes),
        }


# ========== 运行分析 ==========

if __name__ == "__main__":
    energy = EnergyModel()
    array = MACArray(rows=14, cols=12, energy=energy)  # Eyeriss 规模
    
    # AlexNet Conv2 层参数
    conv = ConvLayerConfig(
        H=27, W=27, C=96,    # 输入 27×27×96
        K=256,                # 输出 256 通道
        R=5, S=5,             # 5×5 滤波器
        stride=1,
        batch=1
    )
    
    print("=" * 60)
    print("DNN 加速器数据流能耗对比分析")
    print(f"卷积层: {conv.H}×{conv.W}×{conv.C} → {conv.K}ch, "
          f"kernel={conv.R}×{conv.S}")
    print("=" * 60)
    
    results = {}
    for df in [Dataflow.WEIGHT_STATIONARY, 
               Dataflow.OUTPUT_STATIONARY, 
               Dataflow.ROW_STATIONARY]:
        results[df.value] = array.analyze_conv2d(conv, df)
    
    print(f"\n{'数据流':<25} {'总能耗 (μJ)':>12} {'能耗/MAC (pJ)':>15} "
          f"{'算术强度':>12}")
    print("-" * 65)
    
    best_energy = min(r["total_energy_pJ"] for r in results.values())
    
    for name, r in results.items():
        total_uj = r["total_energy_pJ"] / 1e6
        ratio = r["total_energy_pJ"] / best_energy
        print(f"{name:<25} {total_uj:>10.2f} μJ {r['energy_per_mac_pJ']:>13.2f} "
              f"{r['arithmetic_intensity']:>10.0f} ops/byte ({ratio:.1f}x)")
    
    print(f"\n📊 MAC 总数: {results['WS']['total_macs']:,}")
    print(f"📊 DRAM 访问 (WS): {results['WS']['dram_reads_bytes']/1e6:.2f} MB 读, "
          f"{results['WS']['dram_writes_bytes']/1e6:.2f} MB 写")
    
    # 能量分解对比
    print(f"\n📈 能量分解对比 (WS vs RS):")
    for key in ['dram_energy_pJ', 'mac_energy_pJ']:
        ws_val = results['WS'].get(key, 0)
        rs_val = results['RS'].get(key, 0)
        print(f"   {key}: WS={ws_val/1e6:.2f} μJ, RS={rs_val/1e6:.2f} μJ")
```

## 分析任务

1. **数据流对比**：对 AlexNet 各层分别分析，找出哪一层在哪种数据流下最优
2. **批大小影响**：变化 batch size (1→64)，分析 Output Stationary 的能耗变化
3. **滤波器大小影响**：对比 3×3 vs 7×7 滤波器的数据复用效率
4. **MAC 利用率分析**：考虑阵列大小限制，计算 PE 利用率

## 思考题

1. Weight Stationary 为何最适合推理？在训练场景中存在什么缺陷？
2. 如何通过数据布局优化（如 im2col、Winograd）进一步提升算术强度？
3. Eyeriss 的 Row Stationary 相比其他两种数据流的核心优势是什么？
