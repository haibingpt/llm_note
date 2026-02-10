---
title: xray-llm-generated-questions-characteristics
date: 2026-02-09 13:13
tags: [read, xray, paper]
identifier: 20260209T131342
source: arXiv:2501.03491v2
authors: Yuchen Zhang, Xiaoyuan Liu, Yiyou Sun, Atheer Alharbi, Hend Alzahrani, Tianmeng Shi, Basel Alomair, Dawn Song
venue: Preprint (arXiv)
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   LLM Questions = High Descriptive Bias                 |
|                   + Even Context Focus                   |
|                   - Positional Bias                      |
|                                                          |
|   vs                                                     |
|                                                          |
|   Human Questions = Factoid Focus                       |
|                     + Beginning Bias                     |
|                     + Shorter Answers                    |
|                                                          |
+----------------------------------------------------------+
```

LLM生成的问题系统性地偏向需要描述性长答案的问题类型,并在整个上下文中均匀分布注意力,这与人类提问时专注于事实性问题和上下文开头的模式形成鲜明对比。

# PROBLEM

**痛点定义**: 随着LLM在问题生成(QG)任务中的广泛应用,我们不知道LLM生成的问题与人类提问有何系统性差异,缺乏对其特征的深入理解,这阻碍了QG系统的优化和评估。

**前人困境**:
- 现有QG研究主要关注方法开发,缺少对LLM生成问题特征的系统性分析
- 评估方法要么依赖统计指标(无法捕捉语义丰富性),要么需要大量人工标注
- 没有人深入研究过LLM在QG中是否存在与QA任务类似的系统性偏差

# INSIGHT

**核心直觉**: LLM在生成问题时的行为模式和在回答问题时一样,会表现出可量化的系统性偏好——它们不会简单地模仿人类提问方式,而是展现出自己独特的"提问风格"。

**关键步骤**:
1. **六维度评估框架**: 不是简单对比问题质量,而是解构问题的六个特征维度(类型、长度、上下文覆盖、可回答性、非常见性、所需答案长度),用统计+LLM自动评估的混合方法量化差异
2. **答案长度压缩测试**: 通过迭代压缩答案至最短可接受版本,反向测量问题实际需要多少信息量——揭示LLM偏好长答案问题的本质

# DELTA

**vs SOTA**:
- 首次系统性揭示LLM生成问题的三大特征差异:(1) 描述性问题占比高(T8-T10类型占37-44% vs 人类1.5-3%); (2) 上下文覆盖更均匀(无开头偏见); (3) 答案需求长度显著更长(压缩后仍需13-17词 vs 人类7词)
- 提出自动化评估工作流,将人工标注从数千样本降低到300样本验证级别(Pearson 0.76)

**新拼图**:
- 证明了LLM的QG偏好与QA偏好存在对称性:QA中的位置偏差(positional bias)在QG中消失,表现为均匀的上下文关注
- 为RAG系统和幻觉检测提供新思路:LLM生成的非常见问题比例更高,可作为自动化测试基准

# CRITIQUE

**隐形假设**:
- 假设Wikipedia语料能代表通用知识场景,但专业领域(医疗、金融)可能有不同的问题特征分布
- 假设"描述性"等同于"高质量",但在某些应用场景(快速事实核查)中事实性问题可能更优
- 评估仅用300样本进行人工对齐验证,可能存在采样偏差
- 使用默认参数设置,未探索temperature/top-p等生成参数对问题特征的影响

**未解之谜**:
- 为什么LLM偏好描述性问题?是训练数据中描述性文本占比高,还是生成机制本身的特性?
- 如何让LLM在保持描述性优势的同时,也能生成高质量的事实性问题?
- 不同推理模式(如CoT)是否会改变问题生成偏好?
- 10类问题分类是否完备?模型生成的"Others"类占比低(<1%),可能存在分类体系盲区

# LOGIC FLOW

```
PROBLEM: LLMs widely used in QG, but question characteristics unknown
    |
    v
APPROACH: Compare LLM-generated vs Human-generated questions
    |
    +---> Data: WikiText context --> 4 LLMs generate questions
    |                            --> Compare with TriviaQA & HotpotQA
    |
    +---> Metrics (6 dimensions):
    |       [1] Question Type (10 categories via inductive coding)
    |       [2] Question Length (word count + variance)
    |       [3] Context Coverage (sentence-level + word-level)
    |       [4] Answerability (LLM judges with/without context)
    |       [5] Uncommonness (LLM judges without context)
    |       [6] Required Answer Length (iterative shortening)
    |
    v
KEY FINDINGS:
    [A] Question Type Distribution:
        - LLMs: T8-T10 (Descriptive/Event/Sequential) = 37-44%
        - Humans: T8-T10 = 1.5-3%
        - LLMs: T1-T5 (Factoid) consistent with humans

    [B] Context Coverage:
        - Humans: 50% questions focus on first 10% of context (positional bias)
        - LLMs: Even distribution across entire context (30% per bucket)

    [C] Answer Length Requirements:
        - LLM questions: 13-17 words (after compression)
        - Human questions: ~7 words
        - T6-T10 drive longer answers (especially T8-T10)

    [D] Uncommonness:
        - LLMs: 25% questions unanswerable without context
        - Humans (HotpotQA): 10% unanswerable without context

    v
IMPLICATIONS:
    - LLM QG suitable for: RAG evaluation, hallucination testing
    - Not suitable for: Positional bias detection in QA
    - Need: Prompt engineering to balance question type distribution
```

# NAPKIN SKETCH

```
HUMAN QUESTIONS:                 LLM QUESTIONS:

Context:                         Context:
+---+---+---+---+---+            +---+---+---+---+---+
| * | * |   |   |   |  <-- 50%   | * | * | * | * | * |  <-- Even
| * | * |   |   |   |  focus     | * | * | * | * | * |  spread
| * |   |   |   |   |  on        | * | * | * | * | * |  across
|   |   |   |   |   |  start     | * | * | * | * | * |  context
+---+---+---+---+---+            +---+---+---+---+---+
  Beginning Bias                   No Positional Bias

Question Types:                  Question Types:
+---------+-------+              +---------+-------+
| Factoid | Desc  |              | Factoid | Desc  |
| (T1-T5) |(T8-10)|              | (T1-T5) |(T8-10)|
|         |       |              |         |       |
|  95%    |  3%   |              |  55%    |  40%  |
|         | tiny  |              |         | LARGE |
+---------+-------+              +---------+-------+

Answer Length:                   Answer Length:
Short:  ========== (7 words)     Long: ==================== (15 words)
```

# KEY INSIGHTS DEEP DIVE

## 1. The Descriptive Question Dominance

LLM生成的问题中,描述性/表征性问题(T8)占比达到27-44%,远超人类的1.5-3%。这不是bug,而是feature:

- **Why it matters**: 描述性问题需要综合推理和长答案,这正是测试LLM综合能力的好方法
- **Training data hypothesis**: LLM训练语料中包含大量说明性、描述性文本,导致模型内化了"好答案=详细阐述"的模式
- **Application value**: 对RAG系统评估非常有价值,因为描述性问题能测试系统的信息整合能力

## 2. The No-Positional-Bias Phenomenon

这是最违反直觉的发现:LLM在QG中不表现出QA中常见的位置偏差。

```
QA Positional Bias:              QG Context Coverage:
Answer tends to be at            Questions evenly distributed
beginning or end                 across entire context

[Beginning] ======> [Answer]     [Q1]--[Q2]--[Q3]--[Q4]--[Q5]
                                 Uniform attention
[End] ============> [Answer]     30% each bucket
```

**Interpretation**:
- QA中的位置偏差可能是"快捷推理"的结果(首尾信息显著)
- QG任务迫使模型遍历整个上下文寻找可提问点,抵消了位置捷径

## 3. The Answer-Length-as-Proxy Method

通过迭代压缩答案来测量问题复杂度,这是个巧妙的反向工程:

```
Original Answer: "The gold dollar produced at the Dahlonega
mint in 1860 are significant because only 1,566 were
minted, making them one of the great rarities from
Dahlonega, with roughly a hundred known to exist today."
(~35 words)

Iterative Compression:
v1 (20 words): "Only 1,566 gold dollars minted at Dahlonega
in 1860, making them rare with ~100 known today."

v2 (13 words): "1,566 minted at Dahlonega in 1860, rare,
~100 exist."

v3 (7 words): "Rare Dahlonega 1860 gold dollars, ~100 exist."

Minimum preserving rating: 13 words (for LLM questions)
vs 7 words (for human questions)
```

这说明LLM问题本质上需要更多信息bits才能充分回答。

## 4. Methodology Innovation: Hybrid Evaluation

作者没有选择纯统计或纯人工,而是:
1. **Automatic metrics** for scalability (question type classification via LLM)
2. **Human alignment** on 300 samples (Pearson 0.76) to validate
3. **Iterative refinement** of category definitions

This creates a scalable yet reliable evaluation pipeline for future QG research.

# PRACTICAL IMPLICATIONS

**For RAG Systems**:
- Use LLM-generated questions as test set for multi-hop reasoning evaluation
- LLM questions naturally avoid positional shortcuts, better stress-test

**For Hallucination Detection**:
- LLM's uncommon questions (25% unanswerable without context) can serve as probes
- If model answers these without context, likely hallucinating

**For Prompt Engineering**:
- Need explicit constraints to generate factoid questions (T1-T5)
- Default LLM behavior favors descriptive questions

**For Dataset Creation**:
- Self-Instruct-style QG inherits these biases
- Be aware when using LLM-generated QA pairs for training
