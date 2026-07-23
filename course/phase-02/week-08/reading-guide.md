# Week 8：虚拟内存

> **阅读范围**：Ch2 §2.7-2.8
> **核心问题**：虚拟地址如何映射到物理地址？TLB 如何加速这一过程？

---

## 阅读目标

1. 理解页表的结构和多级页表的动机
2. 掌握 TLB 的工作原理（本质是页表项的专用 Cache）
3. 理解大页（Huge Page）如何减少 TLB miss
4. 能用 gem5 观察 TLB miss 对性能的影响

## 核心概念

### 虚拟内存 = 地址翻译 + 保护

- **页**：虚拟内存的基本单元（通常 4KB）
- **页表**：存储虚拟页→物理页的映射
- **TLB**：页表项的 Cache，通常只有几十到几百个条目

### TLB 作为特殊 Cache

| 属性 | 常规 Cache | TLB |
|------|-----------|-----|
| 存储内容 | 数据/指令 | 页表项（地址映射） |
| 容量 | MB级别 | 几十到几百条目 |
| 相联度 | 通常4-16路 | 通常全相联 |
| 覆盖范围 | TLB 覆盖范围 = 条目数 × 页大小 |

### 大页的优势

4KB 页 × 64 条 TLB = 覆盖 256KB。如果工作集 >256KB → TLB thrashing。
2MB 页 × 64 条 TLB = 覆盖 128MB → 大幅减少 TLB miss。

代价：内部碎片（分配的最小单元就是 2MB）、换页开销大。

### 虚拟机与内存虚拟化（Ch2 §2.4）

现代云计算的基石是虚拟机技术，它对内存系统提出新的挑战：

- **Hypervisor 类型**：Type 1（裸机型，如 VMware ESXi、KVM）直接运行在硬件上，性能最优；Type 2（宿主型，如 VirtualBox）运行在宿主 OS 之上，开销较大
- **嵌套页表（Nested Page Tables / EPT）**：传统影子页表（Shadow Page Table）由 Hypervisor 维护 Guest VA→Host PA 的映射，每次 Guest 页表更新都需 VM Exit，开销极大。Intel EPT / AMD NPT 引入硬件两级翻译——Guest VA→Guest PA→Host PA，消除大部分 VM Exit，大幅提升虚拟化环境下的内存性能
- **TLB 影响**：虚拟化下每次 Guest VA 访问需完成两级页表遍历，TLB miss 代价加倍，促使更大 TLB 和 ASID（Address Space ID）设计以避免 TLB flush

> 理解虚拟化对内存系统的影响，是分析云服务性能开销的关键。

## 动手实验

详见 [lab.md](lab.md)

## 自测

详见 [self-test.md](self-test.md)
