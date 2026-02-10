---
title: xray-safety-attention-heads
date: 2026-02-09 20:02
tags: [read, xray, paper]
identifier: 20260209T200234
source: ICLR 2025
authors: Zhenhong Zhou, Haiyang Yu, Xinghua Zhang, et al.
venue: ICLR 2025
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Ships(q_H, θ_H^l) = KL(p(reject|θ_O) || p(reject|θ_\θ_H^l))   |
|                                                          |
|   Safety = Few Critical Heads, Not All Parameters        |
|                                                          |
+----------------------------------------------------------+
```

通过注意力头消融后的拒绝概率KL散度,定位模型安全机制中的关键注意力头。只需修改0.006%参数(单个头)即可让ASR从0.04提升到0.64。

# PROBLEM

**痛点定义**: LLM安全对齐后仍可被越狱攻击,但没人知道安全机制到底藏在多头注意力的哪个头里

**前人困境**:
- 现有方法要么分析整个神经元(粒度太粗),要么分析整层(改5%参数)
- 直接消融注意力输出无法区分Query/Key/Value矩阵的不同作用
- 无法定位"协同工作"的安全头群组

# INSIGHT

**核心直觉**: 安全能力不是均匀分布的,而是集中在极少数"安全警卫"注意力头中

**关键步骤**:
1. Undifferentiated Attention消融法: 将Query或Key矩阵乘以极小系数ε,让注意力权重退化为均匀分布矩阵A(下三角全1/n),相当于"关闭"该头的特征提取能力
2. Ships度量(Safety Head Important Score): 用KL散度量化"消融该头→拒绝概率下降多少",Ships越大=该头对安全越关键
3. Sahara算法(Safety Attention Head Attribution): 贪心搜索找出"协同作用的安全头群组"(消融后ASR上升最多的组合)

# DELTA

**vs SOTA**:
- 定位精度: 从层级(ActSVD)→头级(Ours),参数修改量从5%→0.018%(3个头)
- 效率提升: 6 GPU小时 vs 850小时(全量生成对比法)
- 发现洞察: 证明了"修改Wq或Wk等价,但修改Wv无效"(安全在注意力权重,不在值矩阵)

**新拼图**:
- LLM安全是"稀疏机制":Llama-2-7b的安全由Head 2-26主导,消融它ASR从0.04→0.64
- 预训练基座模型已塑造安全头布局,对齐只是微调强化
- 不同模型的安全头有重叠(特征共性),但具体位置因base model而异

# CRITIQUE

**隐形假设**:
- 假设安全头的作用是线性可叠加的(实际可能有复杂交互)
- 假设消融单头不会触发其他头的补偿机制
- Ships度量依赖"有害数据集质量",数据集偏差会影响结果
- Sahara的贪心搜索可能陷入局部最优(未尝试全局搜索)

**未解之谜**:
- 为什么这些特定头会负责安全?预训练时学到了什么模式?
- 安全头是否也参与有用能力(消融后helpfulness下降)?
- 攻击者能否反向利用Ships定位并精准攻击安全头?
- 长上下文中安全头的行为会变化吗(论文只测了短查询)?

# LOGIC FLOW

```
问题: LLM被越狱 --> 安全机制在哪?
          |
          v
假设: 安全集中在少数注意力头
          |
          v
方法1: Ships度量(单query级)
   消融head_i --> 计算ASR变化 --> 排序找关键头
          |
          v
方法2: Sahara算法(dataset级)
   SVD分解激活 --> 计算主角度差异 --> 贪心搜索头群组
          |
          v
验证: 消融Llama-2 Head-2-26
   ASR: 0.04 -> 0.64 (16x worse)
   参数改动: 仅0.006%
          |
          v
洞察1: Wq消融 = Wk消融 (安全在权重)
洞察2: Wv消融无效 (值矩阵不重要)
洞察3: 安全头重叠于不同fine-tune版本
```

# NAPKIN SKETCH

```
Multi-Head Attention (32 heads)
+---+---+---+---+     +---+
| 0 | 1 |26*| 3 | ... |31 |   * = Safety Head
+---+---+---+---+     +---+
  |   |   |   |         |
  v   v   v   v         v
 Normal Heads    <-- Feature Extraction

      |26| <-- Safety Guard
       |
       v
  [Harmful?] --> Yes: Reject Token
       |
       No
       v
  [Generate]


Undifferentiated Attention:
Original:              Ablated:
+-------+              +-------+
|   *   |              |1/n 1/n|
| *   * |  -epsilon->  |1/n 1/n|  (uniform)
|*  *  *|              |1/n 1/n|
+-------+              +-------+
Sharp Attention        No Feature Selection
```

# ADDITIONAL INSIGHTS

**实验亮点**:
1. 消融1个头(Llama-2): ASR 0.04→0.64,但BoolQ/MMLU几乎不变(安全头专注安全)
2. Scaling Contribution对比: 修改Wv无效,证明"安全信息在注意力模式,不在内容传递"
3. 预训练重要性: 将对齐模型的其他参数换回base model,安全能力消失→安全头依赖整体上下文
4. 头群组协同: 消融3个头比消融1个头效果更显著(非线性叠加)

**方法论价值**:
- Ships可迁移到其他可解释性任务(找"有用能力头"而非安全头)
- Undifferentiated Attention是通用消融技术,优于直接置零(保持结构完整性)
- 避免了全量生成(850 GPU hours),只需前向传播计算KL散度

**潜在风险**:
- 论文公开了精准定位安全头的方法,攻击者可能:
  1. 设计针对特定头的对抗样本
  2. 微调时故意破坏安全头
  3. 模型窃取攻击中优先提取安全头参数
- 作者未讨论防御措施(如安全头冗余设计)
