---
title: xray-cot-hijacking
date: 2026-02-09 12:09
tags: [read, xray, paper]
identifier: 20260209T120906
source: arXiv:2510.26418v1
authors: Jianli Zhao, Tingchen Fu, Rylan Schaeffer, Mrinank Sharma, Fazl Barez
venue: Preprint. Under Review.
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   ASR = f(CoT_length)                                    |
|                                                          |
|   Long_Benign_CoT + Harmful_Request                      |
|        -> Diluted_Refusal_Signal                         |
|        -> High_Jailbreak_Success                         |
|                                                          |
+----------------------------------------------------------+
```

随着 CoT 推理链越长，拒绝信号被稀释，注意力从有害指令转移，导致越高的攻击成功率

# PROBLEM

**痛点定义**: 大型推理模型（LRM）通过长链思考（CoT）提升任务准确性，但这种"推理能力"反而削弱了安全拒绝机制，创造了新的越狱攻击面

**前人困境**:
- 现有越狱攻击依赖显式安全 CoT（如 H-CoT）或伪装有害内容
- 没人意识到"无害的长推理"本身就能破坏拒绝能力
- 安全对齐假设"更多推理=更强鲁棒性"，但实际相反

# INSIGHT

**核心直觉**: 拒绝是一个低维脆弱信号，长 CoT 通过"注意力稀释"和"语境淹没"让这个信号在后层消失——就像在泳池里加水，盐度自然降低

**关键步骤**:
1. 用无害谜题（数独/逻辑题）生成超长 CoT（1k-47k tokens）
2. 在末尾附加有害指令 + final-answer cue（"Finally, give the answer"）
3. Final-answer cue 强制模型将注意力转向答案区域，绕过已被稀释的拒绝检查

# DELTA

**vs SOTA**:
- Gemini 2.5 Pro: 99% ASR（vs Mousetrap 44%, H-CoT 60%）
- GPT o4 Mini: 94% ASR（vs Mousetrap 25%, H-CoT 65%）
- Grok 3 Mini: 100% ASR（vs AutoRAN 61%）
- Claude 4 Sonnet: 94% ASR（vs H-CoT 11%）

**新拼图**:
- 首次证明"推理深度"与"安全鲁棒性"的权衡关系
- 发现拒绝机制的时空脆弱性：中层编码拒绝强度，后层编码拒绝出口
- 揭示 LRM 的结构性缺陷：attention 机制在长上下文中的分布式稀释

# CRITIQUE

**隐形假设**:
- 假设所有 LRM 的拒绝机制都依赖相同的低维方向（可能有多路径拒绝）
- 假设 final-answer cue 的注意力劫持效果是普遍的（某些模型可能对格式鲁棒）
- 实验只在 HarmBench 的 100 个样本上测试（泛化性存疑）
- 依赖黑盒评判模型（Gemini 2.5 Pro），可能高估/低估真实 ASR

**未解之谜**:
- 为什么 reasoning-effort=low 时 ASR 反而更高？（表 3: 76% vs 68%）
- 如何在不损害推理能力的前提下增强拒绝的鲁棒性？
- 这种攻击是否对多模态推理模型有效？
- 中层注意力头的因果干预能否作为实时防御？

# LOGIC FLOW

```
+----------------+      +------------------+      +-------------------+
| Observation:   | ---> | Hypothesis:      | ---> | Validation:       |
| CoT length     |      | Refusal signal   |      | Mechanistic       |
| inversely      |      | dilution in      |      | analysis shows    |
| correlates     |      | attention space  |      | refusal component |
| with refusal   |      |                  |      | drops in L25-35   |
+----------------+      +------------------+      +-------------------+
        |                                                   |
        v                                                   v
+----------------+                              +-------------------+
| Attack Design: |                              | Causal Test:      |
| Long benign    |                              | Ablating heads    |
| CoT + harmful  |                              | L15-35 collapses  |
| + final cue    |                              | refusal (91% ASR) |
+----------------+                              +-------------------+
        |
        v
+----------------+      +------------------+      +-------------------+
| Result:        | ---> | Implication:     | ---> | Defense:          |
| SOTA ASR       |      | Reasoning != may |      | Monitor refusal   |
| 94-100% across |      | Safety. Need     |      | activation across |
| 4 frontier     |      | depth-aware      |      | layers, not just  |
| LRMs           |      | alignment        |      | final output      |
+----------------+      +------------------+      +-------------------+
```

# NAPKIN SKETCH

```
                    ATTENTION FLOW UNDER CoT HIJACKING

Normal Short Prompt:
+-------+-------+-------+
| Harm  | Harm  | STOP  |  <-- Refusal signal strong, clear boundary
+-------+-------+-------+

CoT Hijacking:
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
|Puzzle|Puzzle|Puzzle|Puzzle|Puzzle|Puzzle|Puzzle|Harm|Ans!|
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
  ^^^                                                  ^^^
  80% attention on puzzle tokens                       Attention
  -> Refusal signal diluted to <5%                     forced to
  -> Safety subnetwork starved                         answer region


REFUSAL COMPONENT BY LAYER:

Harmful (short):    [====|====|====|########]  Strong in L25-35
                     L0   L10  L20  L30  L40

Harmful + Long CoT: [====|====|----........]  Diluted in L25-35
                     L0   L10  L20  L30  L40
                                  ^
                                  Critical safety layers bypassed


ATTENTION RATIO (Harm/Puzzle):

1.0 +
    |  o
0.8 |   \o
    |     \
0.6 |      \
    |       o\
0.4 |          \__
    |             ---___
0.2 |                  ----____
    |                         -----___
0.0 +---+---+---+---+---+---+---+---+---
    1k  4k  8k 12k 16k 20k 24k 28k 32k
              CoT Length (tokens)
```

# KEY MECHANISMS

**Refusal Dilution**:
- Refusal = R = <h_last, v_refusal>（残差激活在拒绝方向的投影）
- 长 CoT 使有害 token 在上下文中占比 <5%
- 中层（L15-25）安全检查强度降低
- 后层（L25-35）拒绝出口被 benign 语境淹没

**Attention Hijacking**:
- Final-answer cue 创建强注意力梯度
- Layers 15-35 的注意力从有害指令转向答案区域
- 无害谜题 token 在 attention 图中形成"噪声背景"
- 有害指令的语义被"平滑"到全局语境

**Causal Evidence**:
- Ablating 60 个特定 attention heads（L15-35, "long CoT + low ratio"）
- 拒绝率从 94% 暴跌至 1%（Direction Addition 实验，表 5）
- 前层头（L15-23）比深层头（L23-35）影响更大（图 9）

# DEFENSE IMPLICATIONS

**Why Existing Defenses Fail**:
- Prompt filtering 无法识别"无害谜题 + 有害指令"组合
- Output monitoring 依赖最终拒绝信号，但已被稀释
- Safety fine-tuning 假设短上下文，不适应长推理场景

**Potential Mitigations**:
1. Monitor refusal components across layers（不只看输出）
2. Enforce minimum attention weight to harmful spans（regardless of CoT length）
3. Penalize refusal dilution during RL training
4. Structured reasoning with safety checkpoints（每 N 步插入安全验证）
5. Separate reasoning engine + safety filter（解耦推理与拒绝）

# EXPERIMENTAL DETAILS

**Models**: Gemini 2.5 Pro, ChatGPT o4 Mini, Grok 3 Mini, Claude 4 Sonnet, GPT-OSS-20B, S1-32B

**Baselines**: Mousetrap, H-CoT, AutoRAN

**Puzzle Types**: Sudoku, abstract math puzzles, logic grid puzzles, skyscraper puzzles

**CoT Lengths**: 1k, 3k, 11k, 21k, 31k, 47k tokens

**Judge Models**: Gemini 2.5 Pro（主判），DeepSeek-v3.1, Substring Matching（辅助）

**Key Result**:
- S1 controlled experiment（表 1）: ASR 从 27%（minimal CoT）升至 80%（extended CoT）
- HarmBench 100 samples: ASR 平均 94-99%

# BROADER RISKS

论文末尾警告：同样机制可能破坏其他安全行为
- Truthfulness: 长推理可能稀释事实检查信号
- Privacy: 长对话历史可能泄露 PII
- Bias mitigation: 推理链可能绕过公平性约束

# ONE-SENTENCE SUMMARY

长链推理通过"注意力稀释 + 拒绝信号淹没"系统性地破坏 LRM 的安全子网络，使最可解释的推理形式（CoT）成为最危险的越狱向量
