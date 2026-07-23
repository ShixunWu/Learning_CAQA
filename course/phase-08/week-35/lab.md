# Week 35 实验：项目实现指导与进度检查

## 实验目标

完成顶点项目的核心实现，运行实验套件，并完成中期进度检查。

## 通用实验框架

```python
"""
Capstone Project Experiment Runner
通用实验框架 - 适用于所有5个项目类型
"""
import json
import csv
import time
import os
from datetime import datetime
from typing import List, Dict, Callable
import statistics


class ExperimentRunner:
    """通用实验运行器，支持参数扫描、多次重复、结果持久化"""
    
    def __init__(self, project_name: str, output_dir: str = "./results"):
        self.project_name = project_name
        self.output_dir = output_dir
        self.results: List[Dict] = []
        os.makedirs(output_dir, exist_ok=True)
    
    def run_parameter_sweep(self, 
                            param_grid: Dict[str, list],
                            run_fn: Callable[[Dict], Dict],
                            repetitions: int = 5):
        """参数扫描：穷举参数组合，每个组合重复多次"""
        
        keys = list(param_grid.keys())
        values = list(param_grid.values())
        
        total_configs = 1
        for v in values:
            total_configs *= len(v)
        
        print(f"参数扫描: {total_configs} 配置 × {repetitions} 次重复 = "
              f"{total_configs * repetitions} 次运行")
        
        from itertools import product
        config_idx = 0
        
        for combo in product(*values):
            params = dict(zip(keys, combo))
            config_idx += 1
            
            for rep in range(repetitions):
                print(f"  [{config_idx}/{total_configs}] {params}  rep {rep+1}/{repetitions}")
                
                try:
                    result = run_fn(params)
                    result['config_id'] = config_idx
                    result['repetition'] = rep + 1
                    result['timestamp'] = datetime.now().isoformat()
                    result['params'] = params
                    self.results.append(result)
                except Exception as e:
                    print(f"    ❌ 错误: {e}")
                    self.results.append({
                        'config_id': config_idx,
                        'repetition': rep + 1,
                        'params': params,
                        'error': str(e),
                    })
        
        self._save_results()
        print(f"\n✅ 完成! 总运行次数: {len(self.results)}")
    
    def aggregate_results(self) -> List[Dict]:
        """聚合多次重复的结果：计算均值和标准差"""
        from collections import defaultdict
        
        grouped = defaultdict(list)
        for r in self.results:
            if 'error' in r:
                continue
            key = r['config_id']
            grouped[key].append(r)
        
        aggregated = []
        for config_id, reps in grouped.items():
            params = reps[0]['params']
            agg = {'config_id': config_id, 'params': params, 'n': len(reps)}
            
            # 聚合数值字段
            numeric_keys = set()
            for r in reps:
                for k, v in r.items():
                    if isinstance(v, (int, float)) and k not in ['config_id', 'repetition']:
                        numeric_keys.add(k)
            
            for nk in numeric_keys:
                vals = [r[nk] for r in reps if nk in r]
                if len(vals) >= 2:
                    agg[f'{nk}_mean'] = statistics.mean(vals)
                    agg[f'{nk}_std'] = statistics.stdev(vals)
                    agg[f'{nk}_cv'] = statistics.stdev(vals) / statistics.mean(vals) * 100
                elif len(vals) == 1:
                    agg[f'{nk}_mean'] = vals[0]
            
            aggregated.append(agg)
        
        return aggregated
    
    def _save_results(self):
        """保存原始结果到 JSON"""
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filepath = os.path.join(self.output_dir, 
                                f"{self.project_name}_{timestamp}.json")
        with open(filepath, 'w') as f:
            json.dump(self.results, f, indent=2, default=str)
        print(f"结果已保存: {filepath}")
    
    def export_aggregated_csv(self, aggregated: List[Dict]):
        """导出聚合结果为 CSV"""
        if not aggregated:
            return
        
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filepath = os.path.join(self.output_dir,
                                f"{self.project_name}_agg_{timestamp}.csv")
        
        # 收集所有列名
        all_keys = set()
        for a in aggregated:
            all_keys.update(a.keys())
        
        with open(filepath, 'w', newline='') as f:
            writer = csv.DictWriter(f, fieldnames=sorted(all_keys))
            writer.writeheader()
            for a in aggregated:
                # 将 params 展开
                row = {}
                for k, v in a.items():
                    if k == 'params' and isinstance(v, dict):
                        for pk, pv in v.items():
                            row[f'param_{pk}'] = pv
                    else:
                        row[k] = v
                writer.writerow(row)
        
        print(f"聚合 CSV 已保存: {filepath}")


# ===== 各项目类型的实验函数示例 =====

def example_cpu_sweep():
    """项目1: CPU 微架构参数扫描示例"""
    runner = ExperimentRunner("cpu-microarch")
    
    param_grid = {
        "issue_width": [1, 2, 4, 8],
        "rob_size": [32, 64, 128, 256],
        "lq_size": [16, 32, 64],
    }
    
    def simulate(params):
        # 这里调用 gem5 并解析结果
        # 以下是模拟数据
        base_ipc = 1.0
        ipc = base_ipc * (params['issue_width'] ** 0.4) * \
              (params['rob_size'] ** 0.15) * (params['lq_size'] ** 0.05)
        # 添加噪声
        import random
        ipc *= random.gauss(1.0, 0.02)
        
        return {
            'ipc': round(ipc, 3),
            'sim_seconds': round(random.uniform(0.001, 0.005), 6),
        }
    
    runner.run_parameter_sweep(param_grid, simulate, repetitions=3)
    agg = runner.aggregate_results()
    runner.export_aggregated_csv(agg)
    
    # 输出最佳配置
    best = max(agg, key=lambda x: x.get('ipc_mean', 0))
    print(f"\n🏆 最佳配置: {best['params']}")
    print(f"   IPC = {best['ipc_mean']:.3f} ± {best.get('ipc_std', 0):.3f}")


def example_cache_sweep():
    """项目2: 缓存参数扫描"""
    runner = ExperimentRunner("cache-optimization")
    
    param_grid = {
        "l1_size_kb": [16, 32, 64],
        "l1_assoc": [4, 8],
        "l2_size_kb": [128, 256, 512],
        "l2_assoc": [8, 16],
    }
    
    def simulate(params):
        import random
        # 简化的缓存性能模型
        l1_miss = 0.10 * (16 / params['l1_size_kb']) ** 0.5 * \
                  (4 / params['l1_assoc']) ** 0.3
        l2_miss = l1_miss * (128 / params['l2_size_kb']) ** 0.4 * \
                  (8 / params['l2_assoc']) ** 0.2
        
        amat = 1 + l1_miss * (10 + l2_miss * 100)  # cycles
        amat *= random.gauss(1.0, 0.03)
        
        return {
            'l1_miss_rate': round(l1_miss, 4),
            'l2_miss_rate': round(l2_miss, 4),
            'amat_cycles': round(amat, 2),
        }
    
    runner.run_parameter_sweep(param_grid, simulate, repetitions=5)
    agg = runner.aggregate_results()
    runner.export_aggregated_csv(agg)


if __name__ == "__main__":
    import sys
    
    if len(sys.argv) > 1:
        project_type = sys.argv[1]
    else:
        project_type = "cpu"
    
    print(f"运行顶点项目实验: {project_type}")
    print("=" * 50)
    
    if project_type == "cpu":
        example_cpu_sweep()
    elif project_type == "cache":
        example_cache_sweep()
    else:
        print("用法: python experiment_runner.py [cpu|cache|gpu|noc|dnn]")

```

## 进度检查清单

请在本周末完成以下检查，并回答每个问题：

### 实现进度
- [ ] 核心代码/模型已实现并能运行
- [ ] 至少完成一个完整的参数扫描（≥ 10 个配置点）
- [ ] 结果可以复现（别人运行你的代码能得到相同结论）
- [ ] 原始数据已保存，准备用于 Week 36 的报告

### 关键发现（至少 3 条）
1. _________________________________________________________________
2. _________________________________________________________________
3. _________________________________________________________________

### 遇到的困难及解决方案
- 困难：__________________________________________________________
- 解决：__________________________________________________________

### 下一步计划
- Week 36 需要完成：_______________________________________________

## 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 仿真结果波动大 | 系统噪声 | 增加重复次数，隔离 CPU 核心 |
| 工具链报错 | 版本不兼容 | 使用 Docker 或降级/升级版本 |
| 参数扫描太慢 | 配置点过多 | 先用粗粒度扫描，缩小范围后精扫 |
| 结果与预期不符 | 模型/配置有误 | 检查输入参数，对比文献基线 |
