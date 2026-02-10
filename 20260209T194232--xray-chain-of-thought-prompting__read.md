---
title: xray-chain-of-thought-prompting
date: 2026-02-09 19:42
tags: [read, xray, paper]
identifier: 20260209T194232
source: https://arxiv.org/abs/2201.11903
authors: Jason Wei, Xuezhi Wang, Dale Schuurmans, et al. (Google Research)
venue: NeurIPS 2022
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Question + "Let's think step by step" + 8 Examples    |
|   -------------------------------------------------------->
|   Emergent Reasoning (only at ~100B+ parameters)        |
|                                                          |
+----------------------------------------------------------+
```

通过在 prompt 中加入少量思维链示例，让大模型在回答前生成中间推理步骤，从而将复杂问题分解为多步求解。

# PROBLEM

**痛点定义**: 大语言模型在数学推理、常识推理等需要多步推理的任务上表现差，即使扩大模型规模也收效甚微。

**前人困境**:
- 标准 few-shot prompting 只提供输入-输出对，模型无法学会"分步思考"
- 训练 rationale 数据集成本高昂（需要大量人工标注中间步骤）
- 微调方法需要为每个任务单独训练模型，缺乏通用性
- 小模型即使给了 CoT 示例也学不会，会生成不连贯或重复的文本

# INSIGHT

**核心直觉**: 人类解决复杂问题时会自然分解成中间步骤。通过在 prompt 中展示"思维链"示例，足够大的语言模型能够模仿这种分步推理模式。

**关键步骤**:
1. **手工构造 8 个思维链示例**: 从训练集中选问题，人工写出详细的推理步骤（不是训练模型，只是改 prompt）
2. **涌现能力门槛**: 只在 ~100B 参数以上的模型才有效，小模型会失败（生成不连贯或错误的推理）

# DELTA

**vs SOTA**:
- GSM8K 数学题：PaLM 540B 从 18% → 57%（超越微调的 GPT-3 verifier）
- SVAMP：从 13% → 79%
- 常识推理 StrategyQA：从 69% → 76%
- 符号推理（Coin Flip）：从 0% → 99%（OOD 泛化）

**新拼图**:
- 证明了"reasoning as emergent ability of scale"：推理能力不是线性增长，而是在模型达到临界规模时突然涌现
- 无需微调或梯度更新，仅通过 prompting 就能激活多步推理能力
- 开创了"intermediate steps as natural language"的范式（而非形式语言或符号系统）

# CRITIQUE

**隐形假设**:
- 需要模型规模 ≥100B 参数（对小模型无效甚至有害）
- 需要任务本身适合分步推理（对单步问题增益有限）
- 需要人工编写高质量的思维链示例（prompt engineering 成本不可忽视）
- 假设模型在预训练时已见过足够的"reasoning-like"文本模式

**未解之谜**:
- 为什么 scaling 会带来涌现？46% 的错误推理链几乎正确（小错误如计算器错误、符号映射错误），54% 有重大语义理解或连贯性错误
- 推理路径的正确性无法保证（可能得到正确答案但推理错误，或推理正确但答案错误）
- 如何在更小的模型上实现 CoT 能力？
- 如何自动生成高质量的思维链示例（避免人工标注）？
- 如何验证推理的真实性（factuality）和可靠性？

# LOGIC FLOW

```
Problem: Multi-step reasoning fails with standard prompting
    |
    v
Insight: Humans decompose problems into intermediate steps
    |
    v
Method: Add 8 manually-written chain-of-thought exemplars
    |
    |---> Exemplar format: Q -> "Let's think step by step:" ->
    |     detailed reasoning -> A
    |
    v
Emergence: Only works at ~100B+ parameters
    |
    |---> Small models: incoherent/repetitive text
    |---> Large models: coherent multi-hop reasoning
    |
    v
Results: Dramatic gains on arithmetic, commonsense, symbolic tasks
    |
    v
Ablations:
    |---> Equation only: helps simple tasks, not complex
    |---> Variable compute: not the key (it's about language steps)
    |---> After answer: no help (must come before)
    |---> Robustness: works across annotators, exemplars, order
    |
    v
Scaling Analysis: 62B -> 540B fixes semantic understanding
                  and one-step-missing errors
```

# NAPKIN SKETCH

```
Standard Prompting:              Chain-of-Thought Prompting:
+-----------------+              +---------------------------+
| Q: 5 balls,     |              | Q: 5 balls, buy 2 more,   |
|    buy 2 more   |              |    buy 2 more             |
| A: 11           |              | A: Roger started with 5.  |
|      X WRONG    |              |    2 cans of 3 = 6 balls. |
+-----------------+              |    5 + 6 = 11.            |
                                 |    Answer: 11  ✓ CORRECT  |
                                 +---------------------------+
                                            |
                                            v
                          +-----------------------------------+
                          | Key: Intermediate natural language|
                          |      steps unlock reasoning       |
                          +-----------------------------------+

Emergent Scaling Curve:

  Performance
      ^
  60% |                                    +----+
      |                                  /
  40% |                              +--+
      |                            /
  20% |    +---+---+---+          +              CoT
      |                      +--+
   0% +----+---+---+---+---+----------------------> Model Size
      8B   62B 137B 175B 540B                      (Standard)

      Reasoning emerges only at ~100B+ parameters
```
