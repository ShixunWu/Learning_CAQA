# Week 30 实验：脉动阵列矩阵乘法仿真

## 实验目标

用 Python 仿真一个简单的脉动阵列（Systolic Array），实现矩阵乘法，
并与传统 CPU 三重循环实现进行吞吐量对比分析。

## 实验背景

脉动阵列是 TPU 等 DNN 加速器的核心计算单元。本实验模拟一个 4×4 的
脉动阵列，理解其数据流和时序行为。

## 任务 1：实现脉动阵列仿真

```python
"""
Systolic Array Simulator - 脉动阵列矩阵乘法仿真
基于 TPU v1 256×256 脉动阵列的简化模型
"""
import numpy as np
import time

class SystolicArray:
    """简化的权重固定（Weight-Stationary）脉动阵列"""
    
    def __init__(self, size: int):
        self.size = size  # 脉动阵列大小 N×N
        self.pe_array = np.zeros((size, size))  # PE 中的累加值
        self.weights = np.zeros((size, size))    # 存储在各 PE 中的权重
        self.total_cycles = 0
        self.utilization = 0.0
    
    def load_weights(self, weight_matrix: np.ndarray):
        """预加载权重到脉动阵列（权重固定策略）"""
        assert weight_matrix.shape == (self.size, self.size)
        self.weights = weight_matrix.copy()
        self.pe_array.fill(0.0)
    
    def compute(self, input_matrix: np.ndarray) -> np.ndarray:
        """
        执行 C = input_matrix @ weights（input 从左流入，weights 已固定在各 PE）
        
        数据流：input 从左向右脉动，partial sums 从上向下流动
        """
        rows_A, cols_A = input_matrix.shape
        K = self.size  # B 的 rows = 脉动阵列大小
        
        assert cols_A == K, f"input_matrix cols ({cols_A}) must == array size ({K})"
        
        output_rows = rows_A
        output_cols = self.size
        result = np.zeros((output_rows, output_cols))
        
        # 计算总周期数：setup + compute + drain
        self.total_cycles = K + (output_rows * output_cols) + K
        
        # 计算 MAC 利用率
        total_mac_possible = self.total_cycles * self.size * self.size
        total_mac_useful = output_rows * K * output_cols  # 实际的 MAC 数
        self.utilization = total_mac_useful / total_mac_possible
        
        # 模拟计算（简化版：按数学结果直接计算）
        # 实际硬件中数据需逐周期脉动流过
        for i in range(output_rows):
            for j in range(output_cols):
                for k in range(K):
                    result[i, j] += input_matrix[i, k] * self.weights[k, j]
        
        return result
    
    def get_stats(self) -> dict:
        return {
            "array_size": self.size,
            "total_cycles": self.total_cycles,
            "mac_utilization": self.utilization,
            "peak_mac_per_cycle": self.size * self.size,
            "total_mac_operations": self.total_cycles * self.size * self.size,
        }


def cpu_matmul(A: np.ndarray, B: np.ndarray) -> tuple:
    """CPU 标准三重循环矩阵乘法，返回结果和执行时间"""
    start = time.perf_counter()
    C = np.zeros((A.shape[0], B.shape[1]))
    for i in range(A.shape[0]):
        for j in range(B.shape[1]):
            for k in range(A.shape[1]):
                C[i, j] += A[i, k] * B[k, j]
    elapsed = time.perf_counter() - start
    return C, elapsed


def systolic_matmul(A: np.ndarray, B: np.ndarray, array_size: int) -> tuple:
    """使用脉动阵列完成矩阵乘法，返回结果和等效周期数"""
    sa = SystolicArray(array_size)
    sa.load_weights(B)
    result = sa.compute(A)
    stats = sa.get_stats()
    return result, stats


# ========== 主程序 ==========
if __name__ == "__main__":
    print("=" * 55)
    print("脉动阵列 vs CPU 矩阵乘法性能对比")
    print("=" * 55)
    
    # 测试配置
    M, K, N = 256, 256, 256  # C[M×N] = A[M×K] × B[K×N]
    
    A = np.random.randn(M, K).astype(np.float32)
    B = np.random.randn(K, N).astype(np.float32)
    
    # CPU 实现
    C_cpu, cpu_time = cpu_matmul(A, B)
    cpu_flops = 2 * M * K * N  # 乘法和加法各算一次
    cpu_gflops = cpu_flops / cpu_time / 1e9
    
    print(f"\n📊 CPU 三重循环 (Python):")
    print(f"   时间: {cpu_time:.4f} s")
    print(f"   吞吐量: {cpu_gflops:.2f} GFLOPS")
    
    # 脉动阵列 - 不同大小
    print(f"\n📊 脉动阵列 (固定频率 700 MHz):")
    freq_mhz = 700
    
    for sa_size in [16, 32, 64, 128, 256]:
        # 需要分块计算（矩阵大于脉动阵列时）
        C_sa = np.zeros((M, N))
        total_cycles = 0
        total_mac = 0
        
        # 分块策略
        for i_start in range(0, M, sa_size):
            i_end = min(i_start + sa_size, M)
            for j_start in range(0, N, sa_size):
                j_end = min(j_start + sa_size, N)
                
                # 对每个块使用脉动阵列
                A_block = A[i_start:i_end, :]
                B_block = B[:, j_start:j_end]
                
                sa = SystolicArray(sa_size)
                # 需要所有 K 维度，所以直接调 compute
                if A_block.shape[1] == sa_size and B_block.shape[0] == sa_size:
                    sa.load_weights(B_block[:sa_size, :])
                    C_sa[i_start:i_end, j_start:j_end] = sa.compute(A_block[:, :sa_size])
                    total_cycles += sa.total_cycles
                    total_mac += 2 * A_block.shape[0] * sa_size * min(sa_size, B_block.shape[1])
        
        compute_time_s = total_cycles / (freq_mhz * 1e6)
        sa_gflops = total_mac / compute_time_s / 1e9 if compute_time_s > 0 else 0
        
        print(f"   {sa_size}×{sa_size} 阵列: "
              f"周期={total_cycles:,}, "
              f"等效时间={compute_time_s*1e6:.1f} μs, "
              f"吞吐量={sa_gflops:.1f} GFLOPS")
    
    # 验证正确性
    C_numpy = A @ B
    if np.allclose(C_cpu, C_numpy, atol=1e-3):
        print(f"\n✅ CPU 实现结果验证通过")
    else:
        print(f"\n❌ CPU 结果有误")
    
    # 分析总结
    print(f"\n📝 分析总结:")
    print(f"   脉动阵列通过数据复用和流水线并行，大幅减少内存访问。")
    print(f"   对于大规模矩阵（M,K,N >> array_size），需分块处理。")
    print(f"   256×256 脉动阵列 @ 700MHz 峰值 ≈ {256*256*2*700e6/1e9:.0f} GFLOPS")

```

## 任务 2：吞吐量对比分析

修改上述代码，完成以下对比：

1. **阵列大小扫描**：变化阵列大小 4→256，记录 MAC 利用率变化
2. **矩阵形状影响**：测试方阵 (N×N)、瘦矩阵 (1×N)、胖矩阵 (N×1)
3. **与 NumPy 对比**：使用 `np.dot()` 作为 BLAS 优化实现的参考

## 任务 3：量化模拟

```python
def quantize_int8(x_fp32: np.ndarray) -> tuple:
    """将 FP32 张量量化为 INT8"""
    x_min, x_max = x_fp32.min(), x_fp32.max()
    scale = (x_max - x_min) / 255.0
    zero_point = round(-x_min / scale)
    x_int8 = np.clip(np.round(x_fp32 / scale) + zero_point, 0, 255).astype(np.uint8)
    return x_int8, scale, zero_point

def dequantize(x_int8: np.ndarray, scale: float, zero_point: int) -> np.ndarray:
    """从 INT8 反量化回 FP32"""
    return (x_int8.astype(np.float32) - zero_point) * scale

# 测试量化精度损失
A_int8, scale_A, zp_A = quantize_int8(A)
B_int8, scale_B, zp_B = quantize_int8(B)
A_recovered = dequantize(A_int8, scale_A, zp_A)
B_recovered = dequantize(B_int8, scale_B, zp_B)

quantization_error = np.mean(np.abs(A - A_recovered))
print(f"量化平均绝对误差: {quantization_error:.6f}")
```

## 思考题

1. 脉动阵列的 MAC 利用率何时达到 100%？何时最低？
2. 权重固定 vs 输出固定 vs 行固定数据流各有何优劣？
3. INT8 量化的 SNR 理论值是多少？实际应用中为什么常低于理论值？
