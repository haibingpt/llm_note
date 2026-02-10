---
title: xray-cot-serial-computation-depth
date: 2026-02-09 12:05
tags: [read, xray, paper]
identifier: 20260209T120554
source: Chain of Thought Empowers Transformers to Solve Inherently Serial Problems
authors: Zhiyuan Li, Hong Liu, Denny Zhou, Tengyu Ma
venue: arXiv 2024
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   TF^(L+T(n))_θ(x) = Circuit_T(x)                       |
|                                                          |
|   Depth × CoT = Serial Computation                      |
|                                                          |
+----------------------------------------------------------+
```

CoT将Transformer的固定深度L重置为L+T(n)，其中T(n)是CoT步数。每一步CoT相当于一次额外的串行计算，将并行架构转换为能够解决本质串行问题的计算模型。

# PROBLEM

**痛点定义**: 常数深度Transformer只能表达TC^0电路（并行计算），无法解决需要串行计算的问题（如排列群组合、迭代平方、电路求值等NC^1完全问题）。

**前人困境**:

- 标准Transformer的深度L是固定常数，决定了串行计算步数的上界
- 即使增加宽度、精度、embedding维度，常数深度模型仍被困在TC^0表达能力边界
- 无法解决排列群S_5的wording问题（需要本质串行计算）
- 经验发现CoT有效，但理论机制不明

# INSIGHT

**核心直觉**: CoT不是"提示技巧"，而是一种"深度扩展机制"。每生成一个中间token，就相当于给模型增加了一层计算深度。T步CoT = 将深度L扩展为L+T。

**关键步骤**:

1. 神来之笔1：用position encoding编码电路拓扑结构，每个CoT步骤模拟一个门操作，将门的输出写回下一个输入位置，实现"深度重置"
2. 神来之笔2：证明带T(n)步CoT的常数深度、常数精度Transformer可以表达任意T(n)大小的电路，突破AC^0边界达到SIZE[T(n)]

# DELTA

**vs SOTA**:

- 前人（Merrill & Sabharwal 2023）：常数深度+log精度Transformer ⊆ TC^0
- 本文：常数深度+常数精度+poly(n)步CoT = P/poly（所有多项式可解问题）
- 首次给出CoT的表达能力严格上界（SIZE^(TC^0)[T(n)+1]）和下界（CoT[T(n), d(n), s(n), e(n)]）

**新拼图**:

- 定义新复杂度类CoT[T(n), d(n), s(n), e(n)]：T(n)步CoT、d(n)维embedding、s(n)精度、e(n)指数位
- 证明AC^0 ⊂ CoT[0, log n, 1] ⊂ SIZE[poly(n)] = P/poly
- 实验验证：CoT让低深度模型解决排列组合、迭代平方、CVP等本质串行问题

# CRITIQUE

**隐形假设**:

- 模型必须学会"将中间结果写回输入位置"这一关键pattern（实践中需要训练数据支持）
- 假设CoT长度T(n)可以根据输入长度动态增长（实际LLM受最大序列长度限制）
- 理论构造是non-uniform的（每个输入长度n对应不同的Transformer参数）
- 依赖TC^0 ⊊ NC^1的复杂度假设（虽然广泛相信但未被证明）

**未解之谜**:

- 如何让模型在训练中自动学会"深度重置"机制？论文只给出存在性证明，未提供学习算法
- uniform版本的表达能力如何？（所有输入长度共享同一组参数）
- 多头注意力、LayerNorm等实际架构变体的影响？
- CoT是否能超越P/poly？是否存在更强的表达能力边界？

# LOGIC FLOW

```
Problem: Constant-Depth Transformer stuck in TC^0
                    |
                    v
Insight: CoT = Depth Extension Mechanism
         Each token = One serial computation step
                    |
                    v
Construction:
  Position Encoding stores: (gate_id, gate_type, input1, input2)
  Attention copies input gate values to current position
  MLP computes gate operation (AND/OR/NOT)
  Output writes back to next input position
                    |
                    v
Theory: Transformer + T(n)-step CoT ~ Circuit of size T(n)
        CoT[poly(n), log n, 1] = P/poly
                    |
                    v
Experiments: 4 inherently serial problems
  - Modular Addition (parallelizable, baseline)
  - Permutation Composition S_5 (serial, NC^1-complete)
  - Iterated Squaring (serial, crypto-hard)
  - Circuit Value Problem (serial, P-complete)
                    |
                    v
Result: Base/Hint fail, CoT solves serial problems
        Depth 1 + CoT outperforms Depth 12 without CoT
```

# NAPKIN SKETCH

```
+------------------+     +------------------+     +------------------+
|   Input Tokens   |     |   CoT Step 1     |     |   CoT Step 2     |
|   x1  x2  x3     | --> | Gate1: x3=NOT x1 | --> | Gate2: x4=x2^x3  |
+------------------+     +------------------+     +------------------+
        |                        |                        |
        v                        v                        v
  +-----------+            +-----------+            +-----------+
  | Position  |            | Position  |            | Position  |
  | Encoding  |            | = Gate 1  |            | = Gate 2  |
  |  (input)  |            | Type=NOT  |            | Type=AND  |
  +-----------+            | Input=1   |            | Input=2,3 |
                           +-----------+            +-----------+
                                 |                        |
                           +-----------+            +-----------+
                           | Attention |            | Attention |
                           | copies x1 |            | copies x2,x3 |
                           +-----------+            +-----------+
                                 |                        |
                           +-----------+            +-----------+
                           |    MLP    |            |    MLP    |
                           | NOT(x1)   |            | x2 AND x3 |
                           +-----------+            +-----------+
                                 |                        |
                                 v                        v
                              Output x3               Output x4

       CoT "resets depth" by writing intermediate results
       back to input for next computation step
```

# KEY THEORETICAL RESULTS

**Theorem 3.1** (Tighter Upper Bound):
- T[poly(n), 1, 1] ⊆ CoT[log n, poly(n), 1, 1] ⊆ AC^0
- 即使用CoT，常数精度+poly embedding仍只能表达AC^0（因为无法计数）

**Theorem 3.2** (Log Precision Still TC^0):
- T[poly(n), log(n), 0] ⊆ CoT[log n, poly(n), log(n), 0] ⊆ TC^0
- 核心技术：将iterative rounding视为ordered automaton，利用Krohn-Rhodes分解

**Theorem 3.3** (Main Result - CoT Breaks Barrier):
- SIZE[T(n)] ⊆ CoT[T'(n), log n, 1] where T'(n) = poly(T(n))
- Corollary: P/poly = CoT[poly(n), log n, 1]
- CoT让Transformer从TC^0跳跃到P/poly表达能力

**Theorem 3.5** (Separation from TC^0):
- S_5 wording problem ∈ CoT[n, log n, 1] but ∉ T[poly(n), log n]
- 假设TC^0 ⊊ NC^1（标准硬度假设）

**Theorem 3.7 & 3.8** (Poly Embedding):
- SIZE^(TC^0)[1+T(n)] = CoT[T(n), poly(n), log n]
- SIZE^(AC^0)[1+T(n)] = CoT[T(n), poly(n), 1]
- Poly embedding不比log embedding更强（在standard hardness下）

# EXPERIMENTAL VALIDATION

**Setup**: 4 problems × 3 settings (base, cot, hint)
- Modular Addition C_p: 可并行，深度1足够（验证baseline）
- Permutation Composition S_5: NC^1-complete，需要串行计算
- Iterated Squaring: 密码学硬度，需要O(n)次串行乘法
- Circuit Value Problem: P-complete，定义了P的复杂度

**Key Finding**:
- Base: 只有modular addition高准确率，其他三个任务准确率低（depth 1: 50%, depth 3: 48%）
- CoT: 所有任务100%准确率（包括depth 1）
- Hint（提供中间步骤作为标签）: 比base好，但不如cot
- 结论：CoT的"形式"（生成中间token）比"内容"（中间结果是否正确）更重要

**Surprising Insight**:
- Depth 1 + CoT > Depth 12 without CoT（在串行问题上）
- 证明了"深度扩展"机制的有效性

# CONSTRUCTION SKETCH (Theorem 3.3)

**输入编码**（n个tokens + T(n)个CoT slots）:
- Token embedding: θ_TE(x_i)
- Position encoding: (gate_id, gate_type, input1_id, input2_id)
  - 使用signed binary encoding: sbin_k(i)

**Attention机制**（拷贝输入门的值）:
- Query: sbin_k(a(i)) （当前门的第一个输入门id）
- Key: B_s · sbin_k(j) （位置j的门id）
- Attention score: 1[gate_id = j]（精确匹配）
- 利用rounded sum保持注意力分数和为1

**MLP层**（计算门操作）:
- F(x_{a(i)}, x_{b(i)}, c(i)) = relu(1-x_{a(i)}-c(i)) + relu(x_{a(i)}+x_{b(i)}+c(i)-2)
- 当c(i)=0: 输出1-a(i) = NOT(a(i))
- 当c(i)=1: 输出a(i) AND b(i)
- 利用2层ReLU网络模拟逻辑门（Lemma E.5）

**关键trick**:
- 用position encoding存储电路结构（避免参数依赖电路）
- Attention作为"数据路由"（根据gate_id拷贝值）
- MLP作为"计算单元"（执行gate operation）
- 输出层"写回"到下一个位置，实现串行链

# TECHNICAL INNOVATIONS

1. **Finite-Precision Modeling**: 引入e-bit exponent, 2s-bit significand的浮点数表示（Definition 3.1-3.3），避免无限精度假设

2. **Ordered Automaton + Krohn-Rhodes**: 将iterative rounding建模为ordered automaton，利用其counter-free性质（transformation semigroup is group-free），通过K-R分解证明可被AC^0电路模拟（Theorem D.1-D.4）

3. **Non-uniform Complexity Class CoT**: 首次形式化定义带CoT的复杂度类，建立与经典电路复杂度的精确对应关系

4. **Position Encoding as Circuit Serialization**: 用两种序列化方式编码电路拓扑（natural order vs CoT simulation order），使Transformer能够"执行"电路

# IMPLICATIONS

**For Theory**:
- CoT是一种通用的"深度放大器"（depth amplifier）
- Transformer + CoT = Universal Circuit Simulator
- 揭示了"为什么LLM需要长上下文"：不是为了记住更多信息，而是为了执行更多串行计算步骤

**For Practice**:
- 解释了为什么zero-shot CoT有效：格式本身就赋予了模型更强的计算能力
- 训练时应关注"写回中间结果"的pattern，而非中间结果的正确性
- 低深度+长CoT可能比高深度+短输出更高效

**Open Questions**:
- 如何设计训练目标自动学习"深度重置"机制？
- Uniform模型（所有长度共享参数）的表达能力界？
- 是否存在CoT无法解决但图灵机可解的问题？
