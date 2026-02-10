---
title: xray-memory-layers-at-scale
date: 2026-02-09 19:51
tags: [read, xray, paper]
identifier: 20260209T195130
source: https://arxiv.org/abs/2412.09764v2
authors: Vincent-Pierre Berges, Barlas Oguz, Daniel Haziza, Wen-tau Yih, Luke Zettlemoyer, Gargi Ghosh
venue: Meta FAIR (December 23, 2024)
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Memory(x) = Softmax(K_I * q) * V_I                    |
|                                                          |
|   where K_I, V_I = trainable lookup table (not FFN)     |
|                                                          |
+----------------------------------------------------------+
```

用可训练的key-value查找表替换部分FFN层，将"存储"从"计算"中解耦出来，实现参数量翻倍但计算量不变。

# PROBLEM

**痛点定义**: LLM的参数增长与计算增长紧密耦合——每增加一个参数（权重矩阵），就必须增加一次乘法运算，导致存储知识的成本=计算成本。

**前人困境**:
- Dense模型：参数全是矩阵权重，每个参数都参与前向计算，扩展知识库=扩展计算量
- MoE模型：虽然能增加参数但不增加计算，但需要复杂的路由机制和专家组合
- 早期Memory Networks：概念存在但从未在现代大规模场景下验证可行性

# INSIGHT

**核心直觉**: 人脑的记忆存储（海马体）和计算处理（前额叶）是分离的。LLM也应该有一个"纯存储"的模块，只负责记住事实（名人生日、国家首都），而不参与复杂计算。

**关键步骤**:
1. **Product-Key查找机制**: 不用暴力搜索百万个keys，而是将query和key都拆成两半，分别在两个sqrt(N)大小的子空间中找top-k，然后取交集。O(N) -> O(sqrt(N))。
2. **跨GPU并行化EmbeddingBag**: 将百万级keys的embedding查找分散到多个GPU，每个GPU只存储部分embedding，通过all-gather同步结果，突破单卡内存限制。
3. **SiLU门控+qk-norm稳定训练**: 在memory输出上加input-dependent gating和query-key归一化，解决大规模memory层训练不稳定问题。

# DELTA

**vs SOTA**:
- 相同计算预算下，Memory模型比Dense模型参数量翻倍（1.3B -> 3.3B参数，但FLOPs相同）
- 在事实性QA任务上，Memory模型（937M参数）超越Dense模型（1.3B参数），准确率提升7-10个百分点
- 扩展到8B base + 128B memory参数，在知识密集型任务上超越Llama3.1 8B（训练token数仅1T vs 15T）

**vs MOE**: 在相同参数量下，Memory模型在事实性任务上显著优于MOE（尤其是小模型时），因为MOE的"专家"仍然是计算单元而非纯存储单元。

**新拼图**: 证明了"轻量计算+重量存储"的架构在现代规模下可行，为未来AI系统提供了"记忆U盘"式的模块化扩展路径。

# CRITIQUE

**隐形假设**:
- **事实性知识占主导的任务才有优势**: Memory层擅长存储离散关联（Paris是法国首都），但对需要深度推理的任务（数学证明、因果链）可能不如dense FFN。论文主要测试QA和编码任务，未覆盖复杂推理场景。
- **需要高带宽GPU互联**: 跨GPU的all-gather操作依赖快速通信（NVLink/Infiniband），在带宽受限环境下可能成为瓶颈。
- **预训练阶段必须有memory**: 论文未测试在已训练dense模型上"插入"memory层的迁移学习能力。

**未解之谜**:
- **如何自动决定哪些层该用memory？**: 论文通过消融实验发现"中间层+大步幅"最优，但缺乏理论指导原则。
- **Memory层学到的知识如何可解释？**: 百万个key-value对应什么语义？是否可以像知识图谱一样查询和编辑？
- **与RAG的关系未明确**: Memory层是"内置的RAG"吗？两者如何协同？论文未对比纯RAG方案。
- **长期记忆更新机制**: 如果训练后世界知识过时（新总统上任），如何高效更新memory而不重新训练？

# LOGIC FLOW

```
Problem: Dense Model Scaling
   |
   | Parameter Growth = Compute Growth
   |                    (tightly coupled)
   v
Observation: Factual Knowledge != Deep Reasoning
   |
   | Facts: discrete associations (can be stored)
   | Reasoning: compositional transforms (needs compute)
   |
   v
Insight: Decouple Storage from Compute
   |
   +---> Replace some FFN layers with Memory Layers
   |
   v
Challenge 1: Naive KNN is O(N) - too slow
   |
   +---> Solution: Product-Key Lookup
   |               O(sqrt(N)) via sub-key decomposition
   v
Challenge 2: Million keys don't fit in one GPU
   |
   +---> Solution: Parallel EmbeddingBag
   |               Shard keys across GPUs + all-gather
   v
Challenge 3: Training instability at scale
   |
   +---> Solution: SiLU gating + qk-normalization
   |
   v
Result: 2x Parameters, Same FLOPs
   |
   +---> Factual QA: +7-10% accuracy
   +---> Code/Knowledge: matches 2x dense model
   +---> 8B+128B memory ~ Llama3 8B (1T vs 15T tokens)
```

# NAPKIN SKETCH

```
Traditional FFN Layer:              Memory Layer:

  Input (x)                           Input (x)
    |                                   |
    v                                   v
+--------+                          +--------+
|  W1*x  |  <-- Computation         | Query  |  <-- Lightweight MLP
+--------+      tied to storage     |  MLP   |
    |                                +--------+
    v                                   |
+--------+                              v
|  W2*x  |                          +---------+
+--------+                          | Product |  <-- O(sqrt(N)) lookup
    |                               |   Key   |      in million-scale table
    v                               | Lookup  |
  Output                            +---------+
                                        |
  Every parameter                       v
  adds 1 multiply!                 +----------+
                                   | top-k    |
                                   | Values   |
                                   +----------+
                                        |
                                        v
                                   +--------+
                                   | SiLU   |  <-- Gating for stability
                                   | Gate   |
                                   +--------+
                                        |
                                        v
                                     Output

                                   Parameters stored,
                                   but NOT computed every time!


Storage vs Compute:

Dense FFN:          Memory Layer:
+----------+        +----------+
| Storage  |        | Storage  |  <-- 128B params
|    +     |        +----------+
| Compute  |             |
+----------+             v
  (coupled)         +----------+
                    | Compute  |  <-- Only 1.3B FLOPs
                    +----------+
                      (decoupled)
```
