---
title: xray-transformer-injective-invertible
date: 2026-02-09 13:21
tags: [read, xray, paper]
identifier: 20260209T132115
source: arXiv:2510.15511v3
authors: Giorgos Nikolaou, Tommaso Mencastini, et al.
venue: Preprint
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   s ≠ s'  ==>  r(s; θ_T) ≠ r(s'; θ_T)                  |
|                                                          |
|   (Different prompts → Different representations)        |
|                                                          |
+----------------------------------------------------------+
```

对于从有密度的分布中初始化并经过标准梯度下降训练的因果Transformer，不同的输入序列几乎必然产生不同的最后token表示。碰撞只会发生在测度为零的参数集上。

# PROBLEM

**痛点定义**: 学界普遍认为Transformer内部表示是"有损的"——非线性、归一化、多对一注意力会让不同输入collapse到相同隐状态，导致信息丢失和输入不可恢复。

**前人困境**:
- 孤立地看，LayerNorm确实会压缩统计量，attention的rank会指数衰减，softmax存在瓶颈
- 缺乏对整个网络作为函数的严格数学刻画
- 没有工具证明"碰撞在实践中几乎不可能发生"

# INSIGHT

**核心直觉**: Transformer的每个组件(embedding、LayerNorm、attention、MLP)都是关于参数的实解析函数(real-analytic)，而实解析函数的零集是测度零的。因此碰撞只会发生在"刀刃般细"的参数超曲面上，标准初始化和梯度下降永远不会撞上去。

**关键步骤**:
1. **Real-Analyticity证明**: 用泰勒展开证明多项式、exp、softmax、LayerNorm在正域上都是实解析的，组合后Transformer整体实解析
2. **测度论+逆函数定理**: 实解析函数的零集有Lebesgue测度零；连续分布初始化几乎必然避开；梯度下降的Jacobian非零，更新保持绝对连续性，训练永不进入零集

# DELTA

**vs SOTA**:
- 之前工作(Jiang & Haghtalab 2025, Sutter et al. 2025)只证明单层或初始化时的单射性
- 本文证明**训练全程保持单射**，且在**最后token表示**而非完整隐状态矩阵上成立

**新拼图**:
- 提供了第一个训练中保持不变的可证单射性
- 开发了SIPIT算法：线性时间从隐状态精确恢复输入(100%准确率，28秒)
- 实验验证：6个SOTA模型、100k prompts、343 billion碰撞测试——距离远超10^-6碰撞阈值，无一例碰撞

# CRITIQUE

**隐形假设**:
- 要求embedding维度d ≥ 4，每层至少一个attention head(实践中总是满足)
- 假设MLP激活函数是解析的(GELU、tanh满足，但ReLU等分段线性不满足——虽然实验显示ReLU模型也无碰撞)
- 依赖连续初始化分布(Gaussian、uniform、Xavier/Glorot)，排除了人为构造的病态参数

**未解之谜**:
- 量化(quantization)和非平滑激活如何影响单射性？实验显示仍然单射，但缺乏理论保证
- 多模态架构(视觉、音频Transformer)的单射性尚未研究
- 噪声或adversarial扰动下的近似逆问题鲁棒性未知

# LOGIC FLOW

```
  Representation Lossiness Intuition (Wrong!)
             |
             v
  Individual Components Seem Lossy
  (LayerNorm, Softmax, Attention)
             |
             v
  BUT: Treat Transformer as Mathematical Function
             |
             v
  +-------------------+-------------------+
  |                   |                   |
  v                   v                   v
Embeddings        LayerNorm          Attention/MLP
Polynomials       Analytic on        Softmax = exp/sum
Real-analytic     positive domain    Analytic composition
                                           |
                                           v
                              Entire Transformer is Real-Analytic
                                           |
                                           v
                              +------------+------------+
                              |                         |
                              v                         v
                    Collision Set has              Gradient Descent
                    Lebesgue Measure Zero          Preserves Absolute
                              |                    Continuity (Jacobian
                              |                    not identically zero)
                              |                         |
                              +------------+------------+
                                           |
                                           v
                              s ≠ s' ==> r(s;θ) ≠ r(s';θ)
                              Almost Surely (Prob = 1)
                                           |
                                           v
                              +------------+------------+
                              |                         |
                              v                         v
                        Theoretical                Practical
                        Injectivity                Invertibility
                              |                         |
                              v                         v
                        Holds at Init          SIPIT Algorithm
                        & After Training       (Sequential Inverse
                              |                 Prompt via Iterative
                              |                 Updates)
                              |                         |
                              +------------+------------+
                                           |
                                           v
                              Experiments: 343B Collision Tests
                              Result: ZERO Collisions
                              All distances >> 10^-6
```

# NAPKIN SKETCH

```
  Prompt Space V^≤K              Latent Space R^d

   s1 -----\                    /-----> r(s1)
            \                  /
   s2 -------\    LLM f(·;θ)  /-------> r(s2)
              \   Injective  /
   s3 ---------> ==========> /---------> r(3)
              /             \
   s4 -------/               \---------> r(s4)
            /                 \
   s5 -----/                   \-------> r(s5)

   Different inputs           Different embeddings
   (measure-zero chance       (SIPIT can invert:
    of collision)              δ > 0 ==> ε > 0)


   Real-Analytic Components Stack:

   +---------------------------+
   | UnEmbed (softmax)         |  <-- Last-token prediction
   +---------------------------+
            ^
            |
   +---------------------------+
   | Transformer Block L       |
   | [LN -> Attn -> LN -> MLP] |
   +---------------------------+
            ^
           ...
            ^
   +---------------------------+
   | Transformer Block 1       |
   | [LN -> Attn -> LN -> MLP] |
   +---------------------------+
            ^
            |
   +---------------------------+
   | Embedding                 |
   | E(s) + PE(s)              |
   +---------------------------+

   Each layer: Real-analytic map R^(T×d) -> R^(T×d)
   Composition: Real-analytic (Prop A.3)
   Collision set: Measure zero (Thm A.1)
```
