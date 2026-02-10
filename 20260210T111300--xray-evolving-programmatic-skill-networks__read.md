---
title: xray-evolving-programmatic-skill-networks
date: 2026-02-10 11:13
tags: [read, xray, paper]
identifier: 20260210T111300
source: https://arxiv.org/abs/2601.03509
authors: Haochen Shi, Xingdi Yuan, Bang Liu
venue: arXiv preprint
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Skill(t+1) = Skill(t) + LLM.REFLECT(error) - buggy    |
|                                                          |
|   + maturity_gating(new_features)                       |
|                                                          |
+----------------------------------------------------------+
```

技能即可执行代码,持续演化而非重写。用"结构化故障定位"修 bug,用"成熟度门控"保护老技能不被破坏。

# PROBLEM

**痛点定义**: 开放世界中(如 Minecraft),智能体需要持续学习新技能,但每次更新要么破坏旧技能(灾难性遗忘),要么代码臃肿不可维护。

**前人困境**:
- Voyager 等方法:每次失败就生成全新代码,没有"debug"能力,技能库变垃圾场
- 单体强化学习:需要海量样本,泛化性差
- 无法平衡"稳定老技能"与"探索新能力"

# INSIGHT

**核心直觉**: 把技能当成"可调试的程序",而非"一次性生成的黑盒"。用 LLM 做"结构化调试员"——定位 bug,局部修复,验证回滚。

**关键步骤**:
1. **REFLECT 机制**: 失败后不重写,而是让 LLM 分析错误日志,精准定位故障函数(如"导航卡死"→修复路径规划部分)
2. **成熟度感知优化**: 给每个技能打"成熟度分数",高分技能锁定核心逻辑只改边缘,低分技能允许大改

# DELTA

**vs SOTA**:
- 对比 Voyager:技能复用率提升 40%,代码行数减少 30%
- 对比 REGAL:在 MineDojo 和 Crafter 环境中任务成功率提升 15-25%

**新拼图**:
- 首次将"程序调试"范式引入具身智能体的技能演化
- 提出"成熟度门控"防止技能退化的机制

# CRITIQUE

**隐形假设**:
- LLM 的调试能力足够强(实际可能误诊 bug)
- 技能可以用"函数式编程"表达(对需要复杂状态机的任务可能失效)
- 环境反馈足够清晰(噪声环境下 REFLECT 可能无效)

**未解之谜**:
- 如何处理"涌现 bug"(多个技能交互产生的故障)?
- 成熟度阈值如何自适应不同环境?
- 技能网络规模增长后(如 1000+ 技能)检索效率如何保证?

# LOGIC FLOW

```
Task --> Skill Selection --> Execute
          ^                    |
          |                  Fail?
          |                    |
          +---- REFLECT <------+
                   |
              Localize Bug
                   |
          +--------+---------+
          |                  |
      Mature Skill      New Skill
          |                  |
    Minimal Edit      Aggressive Refactor
          |                  |
          +--------+---------+
                   |
            Rollback Test
                   |
              Update Library
```

# NAPKIN SKETCH

```
   Skill Library (Code Repo)
   +------------------------+
   | move()     [Mature: 9] |  <-- Protected
   | mine()     [Mature: 7] |
   | craft()    [Mature: 3] |  <-- Editable
   | build()    [Mature: 1] |  <-- Full Refactor OK
   +------------------------+
             |
       LLM Debugger
      /      |      \
  Error   Localize  Patch
   Log      Bug     Code
     \      |      /
      \     |     /
     Rollback Validator
           |
      Update if Pass
```

---

**核心洞察**: 把智能体技能当"Git 仓库"——有版本控制、有测试、有代码审查。老代码别乱动,新代码敢于试错。
