---
title: xray-reasoningbank-memory-evolution
date: 2026-02-09 19:57
tags: [read, xray, paper]
identifier: 20260209T195744
source: https://arxiv.org/abs/2509.25140v1
authors: Siru Ouyang, Jun Yan, I-Hung Hsu, Yanfei Chen, et al. (Google Cloud AI Research)
venue: arXiv 2025
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Memory-Driven Experience Scaling:                      |
|                                                          |
|   Better Memory --> Better Scaling --> Better Memory     |
|        ^                                      |          |
|        |                                      v          |
|   +----+----+           +----------+    +--------+       |
|   | Success | -------> |  MaTTS   |--> | Richer |       |
|   | Failure |          | (k * N)  |    | Memory |       |
|   +---------+          +----------+    +--------+       |
|                                                          |
+----------------------------------------------------------+
```

记忆质量引导探索方向，多样化探索反哺记忆库，形成正向循环

# PROBLEM

**痛点定义**: Agent每次遇到新任务都从零开始，无法从过往成功/失败经验中提炼可复用的推理策略

**前人困境**:
- 现有记忆系统只存储原始轨迹或成功工作流，无法从失败中学习
- 过度强调成功案例，忽略了失败轨迹中的预防性经验
- 缺乏将低层次执行细节抽象为高层次可迁移推理模式的机制

# INSIGHT

**核心直觉**: 经验 = 好方法 + 错题本。成功告诉你"怎么做对"，失败告诉你"别踩坑"

**关键步骤**:
1. **记忆蒸馏**: 用LLM将成功/失败轨迹提炼成结构化记忆项(title + description + content)，过滤执行细节保留推理精华
2. **记忆驱动的测试时扩展(MaTTS)**: 用记忆引导多轮探索，并通过自对比(parallel)或自精炼(sequential)将探索结果再次蒸馏回记忆库

# DELTA

**vs SOTA**:
- WebArena: 比无记忆基线高+8.3% SR，比Synapse/AWM高+4~7% SR
- 效率提升: 成功案例减少2.1步，失败案例减少1.4步(26.9%相对减少)
- MaTTS: 在k=5时，parallel scaling达55.1% SR vs 52.4% vanilla TTS

**新拼图**:
- 首次将失败轨迹显式纳入记忆构建，实现"错题本"式学习
- 提出"memory-driven experience scaling"新范式: 记忆与测试时扩展的协同进化
- 证明了记忆质量比数量更重要(3-4条高质量记忆 > 大量低质量记忆)

# CRITIQUE

**隐形假设**:
- LLM-as-a-Judge能准确标注成功/失败，但对模糊任务可能引入噪声
- 假设简单的embedding检索 + top-k召回就能找到相关记忆(实际可能需要更复杂的组合式检索)
- 假设记忆项是独立的原子单元，未考虑记忆之间的层次组合(如多个记忆项合成更高级策略)

**未解之谜**:
- 如何处理记忆冲突? 两条记忆给出矛盾建议时如何选择?
- 记忆何时过时? 需要forget机制吗?
- Sequential scaling在k>3后收益递减，如何突破这个天花板?
- 论文未深入探索episodic vs semantic memory的架构对比

# LOGIC FLOW

```
Raw Trajectory (Success/Failure)
        |
        v
+-------+--------+
| Memory Extract | <-- LLM distills: title, desc, content
+-------+--------+
        |
        v
+-------+--------+
| ReasoningBank  | <-- JSON storage: {title, desc, content}
+-------+--------+
        |
        +-------------> (i) Memory Retrieval (top-k via embedding)
        |                         |
        |                         v
        |               +------------------+
        |               | Agent Decision   |
        |               +------------------+
        |                         |
        |                         v
        |               +------------------+
        |               | New Trajectory   |
        |               +------------------+
        |                         |
        +<------------------------+
        |
        v
+-------+--------+
| Memory Update  | <-- Consolidation: add new items
+----------------+

With MaTTS:
+-----------------------+       +-----------------------+
| Parallel Scaling      |       | Sequential Scaling    |
| (k trajectories)      |       | (self-refinement k    |
|                       |       |  iterations)          |
| Self-Contrast:        |       | Self-Critique:        |
| Compare success vs    |       | Iteratively check &   |
| failure to extract    |       | correct within one    |
| reliable patterns     |       | trajectory            |
+-----------------------+       +-----------------------+
         |                                 |
         +----------------+----------------+
                          |
                          v
              Better Memory Items
```

# NAPKIN SKETCH

```
        Memory Evolution Timeline
        ==========================

Task 1: [Execute]---->[Fail]---> Extract "Don't do X"
                                      |
                                      v
                            +------------------+
                            | ReasoningBank    |
                            | - Title: ...     |
                            | - Desc: ...      |
                            | - Content: ...   |
                            +------------------+
                                      |
Task 2: [Retrieve]<------------------+
        "Ah! Last time failed because X, avoid it"
                |
                v
        [Execute with hint]---->[Success]---> Extract "Do Y works"
                                                      |
                                                      v
                                          +------------------+
                                          | ReasoningBank    |
                                          | (updated)        |
                                          +------------------+

Task 3: [Retrieve 2 items: "avoid X" + "do Y"]
                |
                v
        [Better decision] --> Fewer steps, Higher SR


With MaTTS:
-----------
         /---> Traj 1 (success)
        |
Query --+---> Traj 2 (fail)      Self-Contrast --> Extract common patterns
        |
         \---> Traj k (success)

Result: Distilled memory has higher signal-to-noise ratio
```
