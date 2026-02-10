---
title: xray-titans-learning-to-memorize
date: 2026-02-09 13:18
tags: [read, xray, paper]
identifier: 20260209T131836
source: arXiv:2501.00663v1
authors: Ali Behrouz, Peilin Zhong, Vahab Mirrokni
venue: arXiv 2024
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   M_t = M_{t-1} + S_t                                   |
|                                                          |
|   S_t = η_t·S_{t-1} - θ_t·∇ℓ(M_{t-1}; x_t)            |
|                                                          |
|   "Memory = Momentum Update - Surprise Gradient"         |
|                                                          |
+----------------------------------------------------------+
```

记忆更新就是一个带动量的梯度下降，对"意外"的数据学得更多，对已知的数据遗忘更快。

# PROBLEM

**痛点定义**: Transformer 的二次复杂度让它无法处理超长上下文，而线性 RNN 虽然高效但无法有效压缩长历史。

**前人困境**:
- Transformer: 注意力机制需要 O(N×d) 次操作，无法扩展到百万 token 级别
- 线性 RNN: 用固定大小的向量/矩阵状态压缩历史，导致信息过早丢失
- 混合模型: 都基于固定参数设计，测试时无法学习如何遗忘和记忆

# INSIGHT

**核心直觉**: 把"记忆"变成一个在测试时也能学习的元模型——训练时学习"如何记忆"，测试时学习"记住什么"。

**关键步骤**:
1. **惊喜驱动记忆**: 把记忆更新转化为元学习——对当前输入的梯度越大（越意外），记得越深
2. **动量遗忘机制**: 用带权重衰减的动量公式同时实现"渐进遗忘"和"快速并行训练"

# DELTA

**vs SOTA**:
- 语言建模: 在所有规模（340M-760M）上超越所有现代线性 RNN 和 Transformer
- 长文档检索: 在 NIAH 任务上，扩展到 2M+ context 依然保持高准确率（99.4%）
- BABILong: 在超长文档推理上完胜 GPT-4 等大模型（few-shot 设置）

**新拼图**: 首次证明线性 RNN 可以通过深度非线性记忆（≥2 层 MLP）在理论上超越 Transformer 的表达能力（TC^0 vs beyond TC^0）。

# CRITIQUE

**隐形假设**:
- 假设训练数据的"惊喜分布"与测试数据一致——如果测试时遇到全新类型的意外，元学习的记忆策略可能失效
- 动量参数（η_t, θ_t）在每个 chunk 内固定——无法适应 chunk 内部的快速变化
- 深度记忆（L_M ≥ 2）带来表达力，但训练效率下降（图 8 显示线性趋势）

**未解之谜**:
- 为什么 MAC（作为上下文）比 MAL（作为层）更好？论文归因于"架构设计"，但未深入分析注意力与记忆交互的机制
- 遗忘门 α_t 如何自适应？论文用固定权重衰减，但未探索动态遗忘策略
- 持久记忆（persistent memory）作为"元知识"，如何防止在长时间训练后也遗忘？

# LOGIC FLOW

```
Problem: Transformer quadratic cost vs Linear RNN limited memory
    |
    v
Insight: Memory as meta-learning (learn HOW to memorize at train time)
    |
    +---> Design: Surprise Metric = Past Surprise + Momentary Surprise
    |            M_t = M_{t-1} + S_t   (momentum-based update)
    |
    +---> Trick 1: Weight Decay ≈ Forgetting Gate (via math derivation)
    |
    +---> Trick 2: Chunk-wise Training with matmuls (parallel on GPU)
    |
    v
Three Architectures:
    |
    +---> MAC (Memory as Context): Memory || Attention || Core
    |           Best for long-context (NIAH, BABILong)
    |
    +---> MAG (Memory as Gating): SWA(Memory + Core)
    |           Fast inference, good for medium-length
    |
    +---> MAL (Memory as Layer): Memory -> Attention
              Good for LM perplexity, less effective on long-context
    |
    v
Result: Scale to 2M+ context with better accuracy than Transformers
```

# NAPKIN SKETCH

```
+------------------+     +------------------+     +------------------+
|  Persistent      |     |  Long-term       |     |  Short-term      |
|  Memory          |---->|  Memory          |---->|  Memory          |
|  (task params)   |     |  (learned        |     |  (attention)     |
|                  |     |   abstraction)   |     |                  |
|  p1, p2, ..., pN |     |  M_t (key-value) |     |  Q, K, V         |
|  (frozen after   |     |                  |     |                  |
|   meta-training) |     |  Forget: α_t     |     |  Window: N       |
|                  |     |  Remember: θ_t   |     |                  |
+------------------+     +------------------+     +------------------+
        |                        ^                        ^
        |                        |                        |
        v                        v                        v
   +----------------------------------------------------------+
   |               Input Sequence x_1, x_2, ..., x_t         |
   +----------------------------------------------------------+
        |
        v
   +----------------------------------------------------------+
   |  Surprise Metric:                                        |
   |  S_t = η_t·S_{t-1} - θ_t·∇ℓ(M_{t-1}; x_t)             |
   |                                                          |
   |  "Gradient big? Surprising! Remember it."                |
   +----------------------------------------------------------+
        |
        v
   +----------------------------------------------------------+
   |  Memory Update (at test time):                          |
   |  M_t = (1-α_t)·M_{t-1} + S_t                           |
   |                                                          |
   |  "Forget old (α_t) + Add new (S_t)"                     |
   +----------------------------------------------------------+
```

**三层记忆系统的类人脑设计**:
- **Persistent Memory**: 像"程序性记忆"，学会骑自行车后不会忘
- **Long-term Memory**: 像"情景记忆"，需要重复巩固，可以遗忘
- **Short-term Memory**: 像"工作记忆"，容量有限，只关注当前

**核心创新的可视化**:
```
Traditional RNN:
   x_t ---> [Fixed Update Rule] ---> h_t
              (same for all data)

Titans (Meta-Learning Memory):
   x_t ---> [Learned HOW to Update] ---> M_t
              (adapt based on surprise)

   During Training: Learn "surprise metric" (θ_t, η_t)
   During Testing:  Use "surprise" to decide what to remember
```
