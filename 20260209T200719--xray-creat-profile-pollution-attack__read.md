---
title: xray-creat-profile-pollution-attack
date: 2026-02-09 20:07
tags: [read, xray, paper]
identifier: 20260209T200719
source: arXiv:2152.09390v3
authors: Jiajie Su, Zihan Nan, Yunshan Ma, Xiaobo Xia, et al.
venue: AAAI 2026
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Polluted_Seq = argmax{Pattern_Inversion_Reward}       |
|                  s.t. Distribution_Shift < threshold     |
|                                                          |
|   where Pattern_Inversion = Directionality + Diversity  |
|         Stealth = -DLOT(Original, Polluted)             |
|                                                          |
+----------------------------------------------------------+
```

在序列推荐的"模式空间"而非"序列空间"做手脚，既能放大攻击效果（模式的级联传播），又能降低检测风险（局部扰动而非整体改变）

# PROBLEM

**痛点定义**: 现有的推荐系统污染攻击（PPA）都在"序列层面"做文章，需要修改大量用户序列才能见效，且容易被检测到

**前人困境**:
- 基于梯度的方法（SimAlter, Replace）只能看到"整条序列"对目标的影响，无法精细控制哪些item转换最关键
- 大规模修改用户兴趣会造成明显的分布偏移，防御系统能轻易识别
- 长尾商品攻击几乎无效，因为缺乏足够的模式支撑

# INSIGHT

**核心直觉**: 推荐模型的预测不是被"整条序列"驱动，而是被"序列内的pattern依赖关系"驱动。就像破解密码不需要改掉整串字符，只需要翻转几个关键的"字符组合模式"

**关键步骤**:
1. **模式反转机制**: 不修改整体兴趣，只反转那些与目标item语义距离最大的sub-pattern，构造"虚假的归因路径"
   - Directionality: 让target前后的pattern与target语义距离最大化
   - Diversity: 用Gram矩阵让攻击模式彼此异构，避免被聚类检测

2. **双层OT约束**: 用Unbalanced Co-Optimal Transport同时约束"序列级语义"和"pattern级转换"，让污染后的序列在表示空间里"看起来"和原序列无差别
   - 序列级: 对齐整体embedding
   - Pattern级: 对齐k-gram子序列的embedding分布

3. **动态屏障强化学习**: 用bi-level RL同步优化
   - Upper level: 最大化pattern反转效果（攻击强度）
   - Lower level: 最小化分布偏移（隐蔽性）
   - 通过动态调整Lagrangian乘子δ，在梯度冲突时优先攻击，在违反隐蔽性时优先防御

# DELTA

**vs SOTA**:
- 相比Replace/SimAlter，在Beauty数据集HR@10提升138%（0.26 vs 0.11）
- 在t-SNE可视化中，CREAT的污染样本与原始样本不可区分，而SOTA仍有明显异常团簇
- 对长尾item攻击成功率首次达到可用水平（10% tail item HR vs 0% for baselines）

**新拼图**:
- 首次提出"模式视域"（pattern horizon）作为攻击锚点，区别于传统的"序列视域"
- 证明了模式级扰动可以通过协同效应（synergistic cascading）在模型训练时泛化到相似模式
- 建立了双层优化框架：上层目标是攻击强度，下层约束是隐蔽性

# CRITIQUE

**隐形假设**:
- 假设攻击者能获取模型架构和损失函数，或能通过model extraction构造替代模型
- 假设推荐系统的pattern依赖关系是可被反向工程的（依赖于attention/GRU的可解释性）
- 假设防御方没有专门的pattern分布监控机制（只检测序列级异常）

**未解之谜**:
- 论文承认目前没有有效防御方法，带来安全风险
- 动态屏障的threshold ρ_st 设置依赖group-wise统计，在小样本或极端分布下可能失效
- 对于更深、更复杂的Transformer backbone（如LLM-based推荐），pattern extraction的计算成本和有效性未知
- k-gram pattern的k值如何自适应选择？论文未给出原则性方法

# LOGIC FLOW

```
Problem: Sequence-level PPA
    |
    v
[Limitation Analysis]
    |
    +---> Low intensity: 只能影响特定兴趣子集
    |
    +---> High detectability: 整体兴趣偏移明显
    |
    v
[Insight: Pattern Horizon]
    |
    +---> Finer-grained: 修改item transition而非整体序列
    |
    +---> Cascading effect: 一个pattern影响多个相似pattern
    |
    v
[Method: CREAT = PBRP + C-GRRL]
    |
    +---> PBRP (Pattern Balanced Rewarding Policy)
    |       |
    |       +---> R_dir: 最大化target与前后context的语义距离
    |       |
    |       +---> R_div: 最大化攻击pattern的多样性（Gram行列式）
    |       |
    |       +---> R_dist: 最小化DLOT距离（隐蔽性）
    |
    +---> C-GRRL (Constrained Group Relative RL)
            |
            +---> Stage 1: 纯攻击训练，找到最有效的pattern位置
            |
            +---> Stage 2: 加入动态屏障约束，微调隐蔽性
                    |
                    +---> delta = [violation + gradient_conflict] / ||grad||^2
    |
    v
[Result]
    |
    +---> 10x exposure increase vs pure model
    |
    +---> 20%+ improvement over SOTA
    |
    +---> Indistinguishable in t-SNE (stealth verified)
```

# NAPKIN SKETCH

```
Original Sequence:
    [item1] --> [item2] --> [item3] --> [item4] --> [item5]
         |           |           |           |
         P0          P1          P2          P3     (patterns)

Attack Strategy (CREAT):
    Step 1: Identify semantic-divergent positions

    [item1] --> [item2] --> [TARGET] --> [item4] --> [TARGET]
         |           |           |            |            |
         P0          P1         P_inv        P2          P_inv
                                  ^                       ^
                                  |                       |
                           Max distance from TARGET
                           + Max diversity among P_inv

    Step 2: Constrain via DLOT

    Original Dist:     {h_orig, p_orig}
                              |
                       minimize DLOT
                              |
    Polluted Dist:     {h_pert, p_pert}

    +------------------------------------+
    |  Dual-Level Co-Optimal Transport   |
    |                                    |
    |  π_s: align sequence embeddings    |
    |  π_f: align pattern embeddings     |
    |                                    |
    |  Constraint: m(π_s) = m(π_f)       |
    +------------------------------------+

    Step 3: RL with dynamic barrier

    Reward at step i:
        r_i = R_dir + R_div - delta * R_dist
              ^       ^         ^        ^
              |       |         |        |
          directional diverse  dynamic  stealth
          inversion  paths     penalty  measure

    delta increases when:
        - Constraint violated: J_dist > rho_st
        - Gradients aligned: grad_J_dist · grad_J_reward > 0
```
