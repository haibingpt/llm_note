---
title: xray-too-good-to-be-bad
date: 2026-02-09 20:04
tags: [read, xray, paper]
identifier: 20260209T200444
source: https://github.com/Tencent/DigitalHuman/tree/main/RolePlay_Villain
authors: Zihao Yi et al.
venue: arXiv 2025
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Role-Playing Fidelity ∝ 1 / Character Morality        |
|                                                          |
|   Safety Alignment ⊥ Villain Authenticity               |
|                                                          |
+----------------------------------------------------------+
```

角色扮演能力随道德水平降低而单调递减——安全对齐与反派真实性存在根本冲突。

# PROBLEM

**痛点定义**: LLM在扮演道德复杂角色时表现优秀，但在扮演反派/恶棍角色时系统性崩溃。

**前人困境**:
- 缺乏系统化的道德谱系评估框架
- 没有平衡的测试集（反派角色严重不足）
- 通用对话能力（Arena排名）无法预测反派扮演能力
- 安全对齐训练系统性压制了"负面特质"的生成能力

# INSIGHT

**核心直觉**: 安全对齐创造了一个根本性冲突——它教会模型"做好人"，但这恰恰摧毁了扮演"坏人"的能力。越安全的模型，越难演反派。

**关键步骤**:
1. **构建道德四级谱系**:
   - Level 1: Moral Paragons（道德楷模）
   - Level 2: Flawed-but-Good（有缺陷的好人）
   - Level 3: Egoists（自私者）
   - Level 4: Villains（反派）

2. **控制变量的平衡测试集**: 800个角色，每级200个，控制场景完整度、情感基调、人物特质分布——确保比较的唯一变量是"道德水平"

3. **特质级分析**: 不止看整体分数，深挖77种人格特质的表现——发现"Deceitful"、"Manipulative"、"Cruel"等负面特质获得最高惩罚

# DELTA

**vs SOTA**:
- 首个系统化的道德对齐谱系基准（Moral RolePlay Benchmark）
- 首次证明"安全对齐税"在创意生成中的存在
- 揭示Arena排名与反派扮演能力的脱钩现象（glm-4.6: Arena第10名 → VRP第1名）

**新拼图**:
- 证实了"对齐的代价"：高度安全对齐的模型（Claude-sonnet-4.5、Claude-opus-4.1）在反派扮演上掉分最严重
- 发现Level 2→Level 3的断崖式下跌（-0.42分）是通用现象
- 建立了"负面特质→性能惩罚"的量化映射

# CRITIQUE

**隐形假设**:
- 假设"特质标注"能准确捕捉角色道德复杂性（但现实中道德是连续谱而非离散标签）
- 假设zero-shot prompting能反映模型真实能力（但可能低估了few-shot或fine-tuned的潜力）
- 假设人类评分者对"authenticity"的判断是一致的（但不同文化对反派的期待不同）
- 评分公式中权重（-0.5×D, -0.1×Dm, +0.15×T）是启发式的，缺乏理论支撑

**未解之谜**:
- 为什么Level 2→3是最陡的断崖？是否存在"自私性阈值"？
- 能否通过对抗性fine-tuning恢复反派扮演能力而不牺牲安全性？
- 高度对齐模型是否能通过"角色扮演框架"（"你在演戏"）绕过限制？
- 负面特质的压制是在pretraining、RLHF还是constitutional AI阶段发生的？

# LOGIC FLOW

```
[Problem Space]
    |
    v
LLMs good at heroes --> bad at villains (why?)
    |
    v
[Hypothesis]
    |
    +---> Safety alignment suppresses negative traits
    |
    +---> Need systematic moral spectrum evaluation
    |
    v
[Method: Moral RolePlay Benchmark]
    |
    +---> Data Curation (23,191 scenes, 54,591 characters)
    |     |
    |     +---> Filter by completeness (>3/5)
    |     +---> Annotate emotional tone (Pos/Neu/Neg)
    |     +---> Assign moral levels (1-4)
    |     +---> Label 77 personality traits
    |
    +---> Balanced Test Set (800 characters)
    |     |
    |     +---> 200 per level
    |     +---> Control for scenario diversity
    |     +---> Control for trait distribution
    |
    +---> Prompting Strategy
    |     |
    |     +---> Zero-shot
    |     +---> "You are an expert actor"
    |     +---> Character profile + Scene context
    |
    +---> Evaluation Protocol
          |
          +---> Fidelity scoring (S = 5 - 0.5D - 0.1Dm + 0.15T)
          +---> Human raters identify inconsistencies
    |
    v
[Findings]
    |
    +---> Monotonic decline: 3.21 (L1) -> 2.61 (L4)
    |
    +---> Steepest drop: Level 2->3 (-0.42)
    |
    +---> Negative traits get highest penalty (3.41 avg)
    |     |
    |     +---> "Manipulative", "Deceitful", "Cruel"
    |
    +---> Arena rank ≠ Villain role-play skill
    |     |
    |     +---> glm-4.6: Arena #10, VRP #1
    |     +---> gemini-2.5-pro: Arena #1, VRP #4
    |
    +---> Highly aligned models drop most
          |
          +---> claude-sonnet-4.5: Arena #2, VRP #9
          +---> claude-opus-4.1: Arena #1, VRP #13
    |
    v
[Root Cause Analysis]
    |
    +---> Trait-level breakdown
    |     |
    |     +---> Positive traits: 3.16 penalty
    |     +---> Neutral traits: 3.23 penalty
    |     +---> Negative traits: 3.41 penalty
    |
    +---> Villain-specific traits get hammered
    |     |
    |     +---> "Hypocritical" (3.55)
    |     +---> "Deceitful" (3.54)
    |     +---> "Selfish" (3.52)
    |
    +---> Models substitute nuance with aggression
          |
          +---> "Manipulative" -> "explodes with rage"
          +---> "Cunning" -> "open insults"
    |
    v
[Implication]
    |
    +---> Safety alignment tax is real
    |
    +---> Creative fidelity ⊥ Prosocial objectives
    |
    +---> Need context-aware alignment methods
```

# NAPKIN SKETCH

```
                  Role-Playing Fidelity

     3.5 |  O  ◯ ◯                              Moral Paragons
         |     ◯     ◯                          (Heroes, Saints)
     3.0 |  □ □   □ □   □
         |                                       Flawed-but-Good
         |     ∇ ∇   ∇     ∇                    (Complex Humans)
     2.5 |  △ △   △   △ △   △
         |                           △          Egoists
         |                                       (Self-serving)
     2.0 |                               ▽
         |                                       Villains
         +----------------------------------------> (Antagonists)
           L1    L2    L3    L4

         THE ALIGNMENT CLIFF
              |
              v
         Level 2 -> Level 3
           -0.42 drop
         (The "Selfish Threshold")


         Safety Alignment Impact:

         +---------------------+
         |  Before Alignment   |
         |                     |
         |  Can play anyone    |
         +---------------------+
                  |
                  | RLHF + Constitutional AI
                  v
         +---------------------+
         |  After Alignment    |
         |                     |
         |  Good at heroes     |
         |  Bad at villains    |
         +---------------------+

         Negative Trait Suppression:

         Manipulative  ▓▓▓▓▓▓▓▓▓ 3.42 penalty
         Deceitful     ▓▓▓▓▓▓▓▓▓ 3.54 penalty
         Selfish       ▓▓▓▓▓▓▓▓▓ 3.52 penalty
         Cruel         ▓▓▓▓▓▓▓▓  3.46 penalty

         vs.

         Resilient     ▓▓       2.93 penalty
         Brave         ▓▓▓      3.12 penalty
         Kind          ▓▓▓      3.17 penalty


         The Paradox:

           Arena Rank    ≠    Villain RolePlay Rank
              |                      |
              v                      v
         gemini-2.5-pro: #1      #4 (VRP)
         claude-opus-4.1: #1    #13 (VRP)
         glm-4.6: #10            #1 (VRP)

         (General smarts ≠ Creative antagonism)
```
