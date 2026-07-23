# Week 29 实验：WSC 总拥有成本（TCO）建模

## 实验目标

构建一个 Python 电子表格模型，计算假设的仓库级计算机（WSC）的总拥有成本。
输入服务器规格、功耗、冷却参数，输出详细的 TCO 分解。

## 实验背景

假设你要为一家云计算公司设计一个包含 50,000 台服务器的数据中心。
你需要计算 5 年生命周期内的总拥有成本。

## 任务：构建 TCO 计算模型

```python
"""
WSC TCO Calculator - 仓库级计算机总拥有成本计算器
基于 CAQA 6th Edition, Chapter 6
"""

from dataclasses import dataclass
from typing import List
import math

@dataclass
class ServerSpec:
    """单台服务器规格"""
    name: str
    cpu_cores: int
    ram_gb: int
    storage_tb: int
    peak_power_w: float          # 峰值功耗 (W)
    idle_power_ratio: float       # 空闲功耗比例 (0-1)
    unit_cost_usd: float          # 单台采购成本 ($)
    
@dataclass
class DataCenterConfig:
    """数据中心配置"""
    num_servers: int
    pue: float                    # Power Usage Effectiveness
    electricity_price_per_kwh: float  # 电价 ($/kWh)
    cooling_cost_ratio: float     # 冷却占非IT能耗比例
    network_cost_per_server: float # 每服务器网络设备成本 ($)
    facility_cost_per_watt: float  # 每瓦基础设施建设成本 ($/W)
    annual_maintenance_per_server: float  # 每服务器年维护费 ($)
    staffing_cost_per_month: float  # 运维人员月薪 ($)
    num_staff: int                # 运维人员数量
    lifetime_years: int           # 生命周期 (年)
    discount_rate: float          # 折现率

def calculate_tco(server: ServerSpec, dc: DataCenterConfig) -> dict:
    """
    计算仓库级计算机的总拥有成本 (TCO)
    
    返回包含 CapEx, OpEx 和 TCO 明细的字典
    """
    # ===== CapEx 资本支出 =====
    
    # 服务器采购成本
    server_cost = server.unit_cost_usd * dc.num_servers
    
    # 网络设备成本
    network_cost = dc.network_cost_per_server * dc.num_servers
    
    # 基础设施成本（按总峰值功率计算）
    total_peak_power = server.peak_power_w * dc.num_servers
    facility_cost = dc.facility_cost_per_watt * total_peak_power
    
    total_capex = server_cost + network_cost + facility_cost
    
    # ===== OpEx 运营支出（每年） =====
    
    annual_opex_list = []
    for year in range(1, dc.lifetime_years + 1):
        # 假设服务器平均利用率为 60%
        avg_utilization = 0.60
        
        # 单台服务器实际功耗（考虑能量比例性）
        actual_power_per_server = server.peak_power_w * (
            server.idle_power_ratio + 
            (1 - server.idle_power_ratio) * avg_utilization
        )
        
        # 总 IT 功耗
        total_it_power_w = actual_power_per_server * dc.num_servers
        
        # 总数据中心功耗（含 PUE）
        total_facility_power_w = total_it_power_w * dc.pue
        
        # 年电力成本
        annual_kwh = total_facility_power_w * 24 * 365 / 1000
        electricity_cost = annual_kwh * dc.electricity_price_per_kwh
        
        # 维护成本
        maintenance_cost = dc.annual_maintenance_per_server * dc.num_servers
        
        # 人员成本
        staffing_cost = dc.staffing_cost_per_month * 12 * dc.num_staff
        
        # 年 OpEx 合计
        annual_opex = electricity_cost + maintenance_cost + staffing_cost
        annual_opex_list.append(annual_opex)
    
    # OpEx 净现值 (NPV)
    npv_opex = sum(
        annual_opex_list[i] / ((1 + dc.discount_rate) ** (i + 1))
        for i in range(dc.lifetime_years)
    )
    
    # ===== 汇总 =====
    total_tco = total_capex + npv_opex
    annual_avg_opex = sum(annual_opex_list) / dc.lifetime_years
    
    return {
        "CapEx 明细": {
            "服务器采购": server_cost,
            "网络设备": network_cost,
            "基础设施": facility_cost,
            "CapEx 合计": total_capex,
        },
        "年度 OpEx 明细": {
            f"第{y+1}年": round(annual_opex_list[y], 2)
            for y in range(dc.lifetime_years)
        },
        "OpEx NPV": round(npv_opex, 2),
        "TCO 总计": round(total_tco, 2),
        "年均 OpEx": round(annual_avg_opex, 2),
        "年均每服务器 TCO": round(total_tco / dc.num_servers / dc.lifetime_years, 2),
        "总 IT 峰值功率 (MW)": round(total_peak_power / 1_000_000, 3),
        "数据中心总峰值功率 (MW)": round(total_peak_power * dc.pue / 1_000_000, 3),
    }


# ========== 运行示例 ==========

if __name__ == "__main__":
    # 定义一台典型云计算服务器
    server = ServerSpec(
        name="Cloud Node v2",
        cpu_cores=64,
        ram_gb=256,
        storage_tb=4,
        peak_power_w=300,
        idle_power_ratio=0.45,     # 空闲时消耗 45% 峰值功率
        unit_cost_usd=4500,
    )
    
    # 定义数据中心配置
    dc = DataCenterConfig(
        num_servers=50000,
        pue=1.15,                   # 较先进的 PUE
        electricity_price_per_kwh=0.08,  # $0.08/kWh
        cooling_cost_ratio=0.40,
        network_cost_per_server=800,
        facility_cost_per_watt=8.0,      # $8/W 基础设施建设
        annual_maintenance_per_server=120,
        staffing_cost_per_month=8000,
        num_staff=30,
        lifetime_years=5,
        discount_rate=0.05,         # 5% 折现率
    )
    
    result = calculate_tco(server, dc)
    
    print("=" * 60)
    print("仓库级计算机 TCO 分析报告")
    print("=" * 60)
    
    print("\n📊 CapEx 资本支出:")
    for key, val in result["CapEx 明细"].items():
        if "合计" in key:
            print(f"  ├─ {key}: ${val:,.0f}")
        else:
            print(f"  │  {key}: ${val:,.0f}")
    
    print(f"\n📈 OpEx (NPV): ${result['OpEx NPV']:,.0f}")
    print(f"\n💰 TCO 总计: ${result['TCO 总计']:,.0f}")
    print(f"📅 年均 OpEx: ${result['年均 OpEx']:,.0f}")
    print(f"🖥️  年均每服务器 TCO: ${result['年均每服务器 TCO']:,.0f}")
    print(f"⚡ 总峰值功率: {result['总 IT 峰值功率 (MW)']:.2f} MW (IT)")
    print(f"⚡ 数据中心总功率: {result['数据中心总峰值功率 (MW)']:.2f} MW")
```

## 分析任务

1. **PUE 敏感性分析**：将 PUE 从 1.1 变化到 1.8，观察 TCO 变化幅度
2. **电价影响**：模拟电价从 $0.05/kWh 到 $0.20/kWh 时的 TCO 变化
3. **能量比例性改进**：将 idle_power_ratio 从 0.45 降至 0.15，计算节省的 TCO
4. **规模效应**：变化服务器数量（10K, 50K, 100K, 500K），分析每服务器 TCO 变化

## 思考题

1. 为什么 CapEx 通常在 TCO 中占比不到 50%？
2. 如果利用率为 30% 和 80%，TCO 差异有多大？这说明了什么？
3. 自建数据中心 vs 租用云服务的 TCO 比较需要考虑哪些额外因素？
