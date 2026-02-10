---
title: xray-alphaone-slow-fast-reasoning
date: 2026-02-09 13:16
tags: [read, xray, paper]
identifier: 20260209T131606
source: https://arxiv.org/abs/2505.24863v1
authors: Junyu Zhang, Runpei Dong, Han Wang, et al.
venue: arXiv 2025
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   p_wait := Bernoulli(S(t)), t = 0,1,...,T_m            |
|                                                          |
|   where S(t) = -1/(T_m) * t + 1  (Linear Anneal)        |
|                                                          |
|   [Pre-α moment]  -->  [α moment]  -->  [Post-α moment] |
|      "wait"              "wait"            "</think>"    |
|    (slow think)      (slow->fast)        (fast think)    |
|                                                          |
+----------------------------------------------------------+
```

通过参数 α 控制的「先慢后快」推理调度：在思考初期高频插入"wait"放慢思考，随时间推移概率递减，最后用"</think>"强制切换到快速答案生成。

# PROBLEM

**痛点定义**: LRM像人类一样需要"慢思考"解决复杂推理问题，但无法自动判断何时该放慢、何时该加速，导致要么思考不足（欠思考）要么思考过度（过思考）。

**前人困境**:
- 并行扩展法（Best-of-N）：只会"广撒网"但不会"深挖掘"
- 单调递增法（s1）：只知道持续加"wait" token，不知道何时该停
- CoD方法：固定5词限制太粗暴，缺乏灵活性
- 所有方法都只调节「思考时长」或「思考深度」其中之一，忽略了两者需要联合优化

# INSIGHT

**核心直觉**: 人类推理是"先慢后快"的（System 1→2过渡）——一开始需要谨慎思考建立框架，中间逐步加速，最后快速收敛到答案。LRM也应该模仿这个过程，而不是单调地增加或减少思考。

**关键步骤**:
1. **α-moment机制**：定义思考预算的"临界点" = αN（α∈[0,1]，N是平均思考token长度），在这个点之前高频插入"wait"强制慢思考，之后自然加速
2. **伯努利随机调度**：不是固定插"wait"，而是按照退火函数 S(t)=−1/T_m·t+1 随机采样是否插入，实现从密集慢思考→稀疏快思考的平滑过渡
3. **确定性终止**：当p_wait=0时强制替换为"</think>"，明确标记思考结束，避免"慢思考惯性"导致无限拖延

# DELTA

**vs SOTA**:
- 比基线模型（vanilla LRM）提升 +6.15% Pass@1准确率（跨6个数学/代码/科学基准）
- 比s1方法（单调递增慢思考）高 +4.62%
- 比CoD（固定5词限制）高 +3.12%
- token使用减少14%（从7280→5916 on AIME24），证明效率更高

**新拼图**:
- 首次提出「推理进度调控」的统一视角：必须同时优化思考时长（thinking budget）和思考调度（scheduling）
- 引入"推理速度"（reasoning velocity）概念：dp/dt衡量单位时间的推理进展，慢思考速度更小但质量更高
- 发现"先慢后快"策略优于"先快后慢"或"单调变化"策略，符合人类认知模式

# CRITIQUE

**隐形假设**:
- **依赖特定transition token**：α1严重依赖"wait"这个特殊token在预训练中学到的语义，如果未来LRM采用不同的慢思考机制（如内部状态调节），该方法可能失效
- **需要预知平均思考长度**：α-moment的计算需要提前在10个样本上测算 N_think，如果测试集分布偏移，α的设置可能次优
- **线性退火函数的最优性未证明**：论文只实验了4种调度函数（constant/linear/exponential/linear anneal），但未探索更复杂的自适应调度策略

**未解之谜**:
- **如何自动发现最优α值**？当前需要人工搜索α∈[0,3]，缺乏理论指导
- **不同问题难度是否需要不同α**？论文用统一的α=1.4应对所有问题，但简单题可能不需要慢思考
- **post-α moment的"快思考惯性"问题**：Table 2消融实验显示去掉post-α调制后性能大幅下降，说明LRM很难自己"刹车"，但为什么会这样？
- **与人类System-1/2切换的神经机制是否真的类似**？论文引用Kahneman但只是隐喻，缺乏认知科学实证

# LOGIC FLOW

```
                    [Problem]
                        |
        LRMs can't auto-modulate reasoning speed
        (overthinknig OR underthinking)
                        |
                        v
                   [Insight]
        Human reasoning: slow-first -> fast-later
        LRMs should mimic this transition
                        |
                        v
                  [α1 Framework]
        +--------------------------------+
        | Pre-α Moment (t < α·N_think)  |
        |   - Insert "wait" ~ S(t)       |
        |   - S(t) = linear anneal       |
        |   - Force slow thinking        |
        +--------------------------------+
                        |
                        v
        +--------------------------------+
        | α Moment (t = α·N_think)       |
        |   - Transition point           |
        |   - p_wait starts decreasing   |
        +--------------------------------+
                        |
                        v
        +--------------------------------+
        | Post-α Moment (t > α·N_think)  |
        |   - Replace "wait" -> "</think>"|
        |   - Deterministic termination  |
        |   - Force fast generation      |
        +--------------------------------+
                        |
                        v
                   [Results]
        +6.15% accuracy, -14% tokens, better REP
```

# NAPKIN SKETCH

```
Reasoning Progress P (0 -> 1)
    ^
  1 |                                    * Final Answer
    |                                 ***
    |                              ***
    |                          ****
    |                      ****  <-- Fast thinking
    |                  ****       (post-α: "</think>")
    |              ****
    |          ****
α·N |      ****  <-- α moment (transition)
    |   ***
    | **
    |**  <-- Slow thinking
    |*       (pre-α: high-freq "wait")
  0 +-------------------------------------------> Time t
    0                   α·N_think              T_total

    [Dense "wait"]  ......  [Sparse "wait"]  [No "wait"]
         |||||||            |   |   |           (fast)
       (slow CoT)         (transition)

    S(t) = -1/T_m * t + 1    p_wait ~ Bernoulli(S(t))

    High prob      -->      Low prob      -->    Force stop
```

