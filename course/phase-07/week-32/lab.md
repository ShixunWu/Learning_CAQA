# Week 32 实验：IEEE 754 单精度浮点加法器实现

## 实验目标

用 Python 实现 IEEE 754 单精度浮点数的加法运算，模拟硬件中的
对阶、尾数运算、规格化和舍入流程。

## 实验背景

浮点加法器是 CPU 中最复杂的算术单元之一。本实验将逐步骤模拟
硬件实现 IEEE 754 FP32 加法的完整流程。

## 任务 1：FP32 解析与编码

```python
"""
IEEE 754 Single-Precision Floating-Point Adder
SPDX: 模拟硬件 FP32 加法器的完整流程
"""
import struct
from enum import Enum

class RoundMode(Enum):
    NEAREST_EVEN = 0   # 最近舍入到偶数（默认）
    TOWARD_ZERO = 1    # 向零舍入
    TOWARD_POS = 2     # 向正无穷
    TOWARD_NEG = 3     # 向负无穷

class FP32:
    """IEEE 754 单精度浮点数表示"""
    
    BIAS = 127
    MANTISSA_BITS = 23
    
    def __init__(self, sign: int, exponent: int, mantissa: int):
        self.sign = sign & 1
        self.exponent = exponent & 0xFF
        self.mantissa = mantissa & 0x7FFFFF
    
    @classmethod
    def from_float(cls, value: float) -> 'FP32':
        """从 Python float 创建 FP32"""
        if value == 0.0:
            return cls(0, 0, 0)
        # 使用 struct 获取 IEEE 754 位表示
        packed = struct.pack('>f', value)
        bits = struct.unpack('>I', packed)[0]
        sign = (bits >> 31) & 1
        exponent = (bits >> 23) & 0xFF
        mantissa = bits & 0x7FFFFF
        return cls(sign, exponent, mantissa)
    
    def to_float(self) -> float:
        """转换为 Python float"""
        bits = (self.sign << 31) | (self.exponent << 23) | self.mantissa
        return struct.unpack('>f', struct.pack('>I', bits))[0]
    
    def is_zero(self) -> bool:
        return self.exponent == 0 and self.mantissa == 0
    
    def is_inf(self) -> bool:
        return self.exponent == 0xFF and self.mantissa == 0
    
    def is_nan(self) -> bool:
        return self.exponent == 0xFF and self.mantissa != 0
    
    def is_denormal(self) -> bool:
        return self.exponent == 0 and self.mantissa != 0
    
    def __repr__(self):
        value = self.to_float()
        return (f"FP32(sign={self.sign}, exp={self.exponent:#04x}, "
                f"mant={self.mantissa:#07x}) = {value}")


def fp32_add(a: FP32, b: FP32, round_mode=RoundMode.NEAREST_EVEN) -> FP32:
    """
    模拟硬件 FP32 加法器
    
    步骤:
    1. 处理特殊情况 (NaN, Inf, Zero)
    2. 对阶 (Alignment)
    3. 尾数加/减
    4. 规格化 (Normalization)
    5. 舍入 (Rounding)
    """
    # ===== Step 1: 特殊情况处理 =====
    if a.is_nan() or b.is_nan():
        # NaN 传播
        return FP32(0, 0xFF, 0x400000)  # 返回 quiet NaN
    
    if a.is_inf() and b.is_inf():
        if a.sign != b.sign:
            return FP32(0, 0xFF, 0x400000)  # ∞ + (-∞) = NaN
        return a
    
    if a.is_inf():
        return a
    if b.is_inf():
        return b
    
    if a.is_zero():
        return b
    if b.is_zero():
        return a
    
    # ===== Step 2: 对阶 =====
    # 提取隐含的 1（或 0，对于非规格化数）
    mant_a = (a.mantissa | (0x800000 if not a.is_denormal() else 0))
    mant_b = (b.mantissa | (0x800000 if not b.is_denormal() else 0))
    
    exp_a = a.exponent if a.exponent > 0 else 1  # 非规格化数指数视为 1
    exp_b = b.exponent if b.exponent > 0 else 1
    
    # 确保 exp_a >= exp_b
    if exp_a < exp_b:
        a, b = b, a
        mant_a, mant_b = mant_b, mant_a
        exp_a, exp_b = exp_b, exp_a
    elif exp_a == exp_b and mant_a < mant_b:
        a, b = b, a
        mant_a, mant_b = mant_b, mant_a
        exp_a, exp_b = exp_b, exp_a
    
    # 对阶：小指数的尾数右移
    shift = exp_a - exp_b
    if shift > 24:
        # 位移过大，b 对结果无影响
        return a
    
    # 保留 guard, round, sticky bits
    sticky = 0
    if shift > 0:
        sticky = 1 if (mant_b & ((1 << shift) - 1)) != 0 else 0
        mant_b = mant_b >> shift
    
    # ===== Step 3: 尾数运算 =====
    if a.sign == b.sign:
        # 同号：相加
        result_mant = mant_a + mant_b
        result_sign = a.sign
    else:
        # 异号：大的减小的（已确保 mant_a >= mant_b）
        result_mant = mant_a - mant_b
        result_sign = a.sign if result_mant != 0 else 0
    
    if result_mant == 0:
        return FP32(0, 0, 0)  # 结果为零
    
    result_exp = exp_a
    
    # ===== Step 4: 规格化 =====
    # 前导 1 检测
    lead = result_mant.bit_length()
    target_lead = 24  # 1 位隐含 + 23 位尾数
    
    if lead > target_lead:
        # 溢出，右移
        shift = lead - target_lead
        sticky |= 1 if (result_mant & ((1 << shift) - 1)) != 0 else 0
        result_mant >>= shift
        result_exp += shift
    elif lead < target_lead:
        # 不足，左移
        shift = target_lead - lead
        result_mant <<= shift
        result_exp -= shift
    
    # 检查指数范围
    if result_exp >= 255:
        # 上溢
        return FP32(result_sign, 0xFF, 0)  # 返回 ±∞
    if result_exp <= 0:
        # 下溢：转为非规格化数
        shift = 1 - result_exp
        if shift >= 24:
            return FP32(result_sign, 0, 0)  # 下溢为 0
        sticky |= 1 if (result_mant & ((1 << shift) - 1)) != 0 else 0
        result_mant >>= shift
        result_exp = 0
    
    # ===== Step 5: 舍入 =====
    # 提取最低位、round 位、sticky 位
    L = (result_mant >> 1) & 1  # 最低有效位
    G = result_mant & 1          # guard bit
    # sticky 已在上面维护
    
    round_up = False
    if round_mode == RoundMode.NEAREST_EVEN:
        if G == 1 and (sticky == 1 or L == 1):
            round_up = True
    elif round_mode == RoundMode.TOWARD_ZERO:
        round_up = False
    elif round_mode == RoundMode.TOWARD_POS:
        round_up = (result_sign == 0 and (G == 1 or sticky == 1))
    elif round_mode == RoundMode.TOWARD_NEG:
        round_up = (result_sign == 1 and (G == 1 or sticky == 1))
    
    # 舍入前去除 guard bit
    result_mant >>= 1
    
    if round_up:
        result_mant += 1
        # 舍入可能引起进位，再次规格化
        if result_mant >= 0x1000000:
            result_mant >>= 1
            result_exp += 1
            if result_exp >= 255:
                return FP32(result_sign, 0xFF, 0)
    
    # 移除隐含位
    if result_exp > 0:
        result_mant &= 0x7FFFFF
    
    return FP32(result_sign, result_exp, result_mant)


# ========== 测试与验证 ==========

if __name__ == "__main__":
    print("=" * 55)
    print("IEEE 754 FP32 加法器测试")
    print("=" * 55)
    
    test_cases = [
        (1.0, 2.0, "1.0 + 2.0"),
        (1.5, 2.75, "1.5 + 2.75"),
        (-1.5, 2.5, "-1.5 + 2.5"),
        (1e38, 1e38, "1e38 + 1e38 (overflow)"),
        (1e-38, 1e-38, "1e-38 + 1e-38 (underflow)"),
        (0.0, -0.0, "0 + (-0)"),
        (0.1, 0.2, "0.1 + 0.2 (famous case)"),
    ]
    
    all_pass = True
    for a_val, b_val, desc in test_cases:
        expected = a_val + b_val
        fp_a = FP32.from_float(a_val)
        fp_b = FP32.from_float(b_val)
        result_fp = fp32_add(fp_a, fp_b)
        result = result_fp.to_float()
        
        # 计算 ULP 误差
        if expected != 0:
            rel_err = abs(result - expected) / abs(expected)
        else:
            rel_err = abs(result - expected)
        
        status = "✅" if (rel_err < 1e-6 or (expected == 0 and result == 0)) else "❌"
        if status == "❌":
            all_pass = False
        
        print(f"\n{desc}:")
        print(f"  预期: {expected},  实际: {result}")
        print(f"  相对误差: {rel_err:.2e}  {status}")
    
    # 精度分析
    print(f"\n{'='*55}")
    print("舍入误差分析")
    
    # 测试非结合性
    a, b, c = 1e20, -1e20, 1.0
    result1 = (a + b) + c  # float
    result2 = a + (b + c)
    print(f"(1e20 + (-1e20)) + 1.0 = {result1}")
    print(f"1e20 + ((-1e20) + 1.0) = {result2}")
    print(f"浮点加法不满足结合律!")
    
    # 测试灾难性抵消
    a = 1.0000001
    b = 1.0000000
    print(f"\n灾难性抵消: {a:.10f} - {b:.10f} = {a-b:.10e}")
    print(f"有效精度损失: 原 ~7 位有效数字 → 结果仅 ~1 位")
```

## 任务 2：舍入模式对比分析

修改 fp32_add 的舍入逻辑，对以下场景测试四种舍入模式：

```python
# 连续加 0.1 十次，观察不同舍入模式的累积误差
values = [0.1] * 10
for mode in RoundMode:
    total = FP32.from_float(0.0)
    for v in values:
        total = fp32_add(total, FP32.from_float(v), round_mode=mode)
    exact = 1.0
    error = total.to_float() - exact
    print(f"{mode.name}: 10×0.1 = {total.to_float()}, 误差 = {error:.1e}")
```

## 分析任务

1. 验证 0.1 + 0.2 ≠ 0.3 的原因（二进制表示精度限制）
2. 分析灾难性抵消的数值根源
3. 比较四种舍入模式在 1000 次随机加法的累积误差分布

## 思考题

1. 为什么 IEEE 754 默认舍入模式选择 "最近偶数" 而非 "四舍五入"？
2. 浮点加法器中的 sticky bit 有什么作用？
3. 芯片设计中选择 Ripple Carry Adder vs Carry Lookahead Adder 的权衡是什么？
