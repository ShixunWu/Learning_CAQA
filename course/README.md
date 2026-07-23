# 计算机体系结构：量化研究方法 - 36周深度学习课程

> 基于《Computer Architecture: A Quantitative Approach》第6版（Hennessy & Patterson）

## 课程哲学

这本书难啃有三个原因：**密度太大**（每页信息量极高）、**缺乏反馈**（纯阅读无验证）、**理论悬浮**（没有动手锚点）。本课程通过三个机制解决：

- **粒度拆分**：每章拆成3-6个小模块，每次只攻克一个子主题
- **实践先行**：每个模块先跑仿真看现象，再回头读理论，形成"现象->理论"的正向反馈
- **三级检验**：每周自测 -> 每阶段项目 -> 最终毕业设计，层层验证

## 适用人群

已有计组/体系结构基础知识（了解流水线、Cache等概念但不成体系），每周可投入4-6小时。

## 课程路线图

```
Phase 1 (Week 1-4)   Phase 2 (Week 5-9)   Phase 3 (Week 10-12)
定量设计基础           存储层次               流水线基础
[Ch1 + App A/K]       [App B + Ch2 + App D]  [App C]
      |                     |                      |
      v                     v                      v
Phase 4 (Week 13-18)  Phase 5 (Week 19-23)  Phase 6 (Week 24-28)
指令级并行             数据级并行              线程级并行
[Ch3 + App H]         [Ch4 + App G]           [Ch5 + App F/I]
      |                     |                      |
      v                     v                      v
Phase 7 (Week 29-33)  Phase 8 (Week 34-36)
现代体系结构            毕业设计
[Ch6 + Ch7 + App E/J/L]
```

## 目录结构

```
course/
├── README.md                    # 本文件
├── week-00-setup/               # 环境搭建
│   └── guide.md
├── phase-01/                    # 定量设计基础
│   ├── week-01/  week-02/  week-03/  week-04/
│   └── phase-review.md
├── phase-02/                    # 存储层次
│   ├── week-05/ ... week-09/
│   └── phase-review.md
├── ...                          # Phase 3-7
├── phase-08/                    # 毕业设计
│   ├── week-34/  week-35/  week-36/
│   └── capstone.md
└── appendix/
    ├── knowledge-map.md         # 知识地图模板
    └── certification-checklist.md # 自我认证清单
```

## 每周学习模板

每周包含三个文件：
- `reading-guide.md` - 阅读目标和核心概念
- `lab.md` - 动手实验指导
- `self-test.md` - 自测题目（附答案）

## 配套资料

- 课本 PDF：`Computer Architecture A Quantitative Approach (6th Edition).pdf`
- 习题解答：`Solutions Manual`
- 课件 PPT：`CAQA6e_ch1.pptx` ~ `CAQA6e_ch7.pptx`
- 在线附录：`References Appendices/Appendix_D_online.pdf` ~ `Appendix_L_online.pdf`

---

## 开始学习

请从 **[Week 0 环境搭建](../week-00-setup/guide.md)** 开始。
