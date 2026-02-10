---
title: xray-ace-incremental-context
date: 2026-02-09 13:23
tags: [read, xray, paper]
identifier: 20260209T132349
source: arXiv:2510.04618v1
authors: Qizheng Zhang, Changran Hu, Shubhangi Upasani, et al.
venue: arXiv preprint
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Context = Delta_1 + Delta_2 + ... + Delta_n           |
|   (NOT: Context = Rewrite(Everything))                  |
|                                                          |
+----------------------------------------------------------+
```

上下文应该是一系列增量更新的累积，而不是不断压缩重写的总结。

# PROBLEM

**痛点定义**: 现有上下文自适应方法会将长上下文压缩成简短总结，丢失关键细节导致性能崩溃。

**前人困境**:
- Brevity Bias（简洁性偏见）：优化器倾向生成短小通用的提示，丢失领域特定知识
- Context Collapse（上下文坍缩）：LLM 重写上下文时会把 18,282 tokens 压成 122 tokens，准确率从 66.7% 暴跌至 57.1%

# INSIGHT

**核心直觉**: 上下文不是需要压缩的负担，而是应该精心培育的"演化剧本"（evolving playbook），每次只添加新发现的 delta，保留所有细节。

**关键步骤**:
1. **增量 Delta 更新**：将上下文表示为结构化的 bullets（每个包含 identifier + content），每次只更新相关的 bullets，而非重写全部
2. **三角色分工**：Generator（生成轨迹）-> Reflector（提取洞见）-> Curator（合并增量），避免单模型过载
3. **Grow-and-Refine 机制**：允许上下文先自由增长，然后用语义去重清理冗余，防止无脑膨胀

# DELTA

**vs SOTA**:
- AppWorld agent 任务上比 GEPA 提升 10.6%，比 Dynamic Cheatsheet 提升 7.6%
- 适应延迟降低 86.9%，rollout 次数减少 75.1%
- 在无标注反馈情况下，仍能超越基线 14.8%

**新拼图**:
- 证明"上下文应该详细且进化"这一反直觉原则
- 展示增量更新可以同时实现高质量和低成本
- 在 AppWorld 排行榜上，用开源模型（DeepSeek-V3.1）追平闭源生产级系统（IBM CUGA with GPT-4.1）

# CRITIQUE

**隐形假设**:
- 依赖 Reflector 能从执行轨迹中提取有意义的洞见，如果 Reflector 失败，上下文会变得噪声甚至有害
- 需要高质量反馈信号（代码执行结果、环境反馈等），没有可靠反馈时效果会退化
- 假设任务需要"积累领域知识"，对于简单查询或固定规则游戏，额外上下文可能是冗余

**未解之谜**:
- 长上下文的推理成本虽然有 KV cache 等优化，但仍比短提示贵，论文未详细讨论成本-收益权衡
- 何时触发 Grow-and-Refine 的去重？论文说"定期或懒惰"，但缺乏明确策略
- 如果领域专家发现上下文中有错误洞见，如何支持 selective unlearning？

# LOGIC FLOW

```
Problem: Context Collapse
    |
    v
Why? LLM 被要求"重写全部" -> 倾向压缩成短总结 -> 丢失细节
    |
    v
Insight: 不要重写，只做增量追加
    |
    v
Method: ACE = Generator + Reflector + Curator
    |
    +---> Generator: 执行任务，生成轨迹
    |
    +---> Reflector: 分析轨迹，提取 delta insights
    |
    +---> Curator: 将 delta 合并到现有 Playbook（增量更新）
    |
    v
Context Structure: List of Bullets (id + content)
    |
    +---> Localization: 只更新相关 bullets
    +---> Fine-grained: 每个 bullet 是独立的知识单元
    +---> Incremental: 保留历史 + 添加新发现
    |
    v
Result: 详细、可解释、可扩展的上下文
    |
    v
Performance: +10.6% agents, +8.6% finance, -86.9% latency
```

# NAPKIN SKETCH

```
Traditional Approach (Monolithic Rewrite):
+-------------------+         +-------------------+
| Long Context      |  LLM    | Short Summary     |
| 18,282 tokens     | ------> | 122 tokens        |
| Acc: 66.7%        | Rewrite | Acc: 57.1% CRASH! |
+-------------------+         +-------------------+

ACE Approach (Incremental Delta):
+-------------------+         +-------------------+
| Playbook v1       |  Delta  | Playbook v2       |
| - Strategy A      |   +     | - Strategy A      |
| - Code Snippet B  | ------> | - Code Snippet B  |
|                   |  merge  | - New Insight C   |
+-------------------+         | - Fix for Error D |
                              +-------------------+
                              (Preserves All Details)

Three-Agent Architecture:

    Query --> [Generator] --> Trajectory
                                  |
                                  v
                              [Reflector] --> Delta Insights
                                  |
                                  v
    Playbook <-- [Curator] <-- Update Rules
```
