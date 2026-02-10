---
title: xray-evolving-programmatic-skill-networks
date: 2026-02-10 11:13
tags: [read, xray, paper]
identifier: 20260210T111315
source: https://arxiv.org/abs/2601.03509
authors: Haochen Shi, Xingdi Yuan, Bang Liu
venue: arXiv 2026
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Skill(t+1) = Skill(t) + REFLECT(failure)              |
|                         - REFACTOR(redundancy)           |
|                         * MATURITY_GATE(reliability)     |
|                                                          |
+----------------------------------------------------------+
```

技能网络 = 可执行程序图 + 反向传播式的错误归因 + 成熟度感知的更新门控 + 结构重构

# PROBLEM

**痛点定义**: 开放世界中的 AI 智能体需要持续积累技能，但现有方法要么把技能存成静态文本（无法执行），要么疯狂遗忘旧技能，还不会自动"合并重复代码"。

**前人困境**:
- 基于 LLM 的智能体（如 Voyager）：技能是扁平的代码库，不断膨胀，没有层次复用，学新忘旧
- 分层强化学习：技能是黑盒神经网络，无法解释和调试
- 无法像人类程序员一样"重构代码"——发现重复逻辑、抽象共性、删除冗余

# INSIGHT

**核心直觉**: 把技能当作"可微的程序"——不是梯度下降的微分，而是**符号化的反向传播**。失败时，沿着调用链向上追溯责任，定位错误的子技能；成功时，检测重复模式并重构成抽象技能。

**关键步骤**:
1. **REFLECT = 符号微分**: 失败时，通过执行 trace 反向传播"责任信号"，定位哪个子技能、哪个参数、哪个前置条件出错
2. **成熟度门控 = 学习率调度**: 类似神经网络的 layer freezing，成熟技能（高 V(s) 值）降低更新频率，不成熟技能保持可塑性
3. **REFACTOR = 神经架构搜索**: 定期扫描技能网络，自动合并重复子图、抽取公共子技能、删除冗余分支，用 rollback 验证重构安全性

# DELTA

**vs SOTA**:
- vs Voyager: 在 Minecraft 科技树任务中，PSN 用 51 次迭代解锁钻石工具，Voyager 需 102 次；PSN 的技能留存率 67-100%，Voyager 灾难性遗忘降至 0%
- vs ReAct/Reflexion: 后者每次都重新推理，PSN 积累可复用的程序化技能
- vs AutoGPT: 后者无结构化技能库，PSN 维护显式的依赖图

**新拼图**: 首次将**神经网络训练的三大机制**（反向传播、学习率调度、架构搜索）映射到**符号程序学习**，证明这些原理跨表征通用。

# CRITIQUE

**隐形假设**:
- 假设环境反馈足够明确：REFLECT 依赖清晰的失败信号（如"木板不足"），但真实世界很多失败是模糊的（"为什么用户不喜欢这个 UI？"）
- 假设技能可分解为树状调用：但某些技能是并行协作的（多智能体、异步任务），树状结构可能不够
- 假设 LLM 生成的代码质量足够高：论文用 GPT-5-mini，但更弱的模型可能生成不可执行代码

**未解之谜**:
- 重构的"投影保证"缺失：论文承认重构没有形式化证明保持行为等价，只靠 rollback 验证（试错式）
- 批量学习的缺失：当前是 batch-size=1 的在线学习，限制了并行探索
- 如何扩展到更大规模：目前技能库 ~50-70 个技能，扩展到 1000+ 技能时搜索和重构效率如何？

# LOGIC FLOW

```
Task Stream --> Hybrid Planner --> Execute Skill --> Success?
                     ^                   |               |
                     |                   v               v
                     |              [Trace T]        [Feedback f]
                     |                   |               |
                     |                   v               v
                     +--- REFACTOR <-- OPTIMIZE <-- REFLECT
                              |             |             |
                              v             v             v
                        [Merge Skills] [Patch Code] [Localize Fault]
                              |             |             |
                              +-------------+-------------+
                                            |
                                            v
                                   Update Skill Network N
                                            |
                                            v
                                   Maturity-Aware Gating
                                   (Mature: freeze; Immature: update)
```

# NAPKIN SKETCH

```
       +-------------+
       | Craft Table |  <-- Goal
       +------+------+
              |
       +------v------+
       | Craft Planks|  <-- Skill s1 (mature: V=0.9)
       +------+------+
              |
       +------v------+
       | Mine Oak Log|  <-- Skill s2 (immature: V=0.3)
       +------+------+
              |
         [FAIL: no axe]
              |
              v
       +------+------+
       | REFLECT(s2) | --> "Missing precondition: need axe"
       +------+------+          |
              |                 v
              |          +------+-------+
              +--------> | Craft Axe    | <-- New skill s3
                         | (subgoal)    |
                         +--------------+

After 10+ successes:
       +------+-------+
       | ensureAxe()  | <-- REFACTOR: abstract common pattern
       +------+-------+
              ^
              |
       +------+------+     +------+------+
       | Mine Oak    |     | Mine Birch  |
       +-------------+     +-------------+
       (both depend on ensureAxe)
```

---

## 补充洞察

### 与神经网络训练的类比

论文最精彩的洞察：PSN 的学习动力学与神经网络**结构同构**

| 神经网络            | PSN (程序化技能网络)           |
|---------------------|--------------------------------|
| 反向传播 (Backprop) | REFLECT (责任链式追溯)         |
| 学习率调度          | 成熟度门控 (V(s) 越高更新越少) |
| 架构搜索 (NAS)      | REFACTOR (合并/抽象/剪枝)      |
| 梯度消失            | 技能链过长导致信号衰减         |
| Layer Freezing      | 成熟技能稳定化                 |

这暗示：**连续优化的原理可以迁移到离散符号系统**，只要重新定义"微分"和"更新"的含义。

### 实验亮点

1. **Minecraft 钻石工具解锁**：PSN 完成率 100% (3/3)，Voyager 0% (0/3)
2. **Crafter 生存环境**：PSN 持续增长，Voyager 在铁工具阶段停滞
3. **技能留存率**：PSN 保持 67-100%，Voyager 随任务深入降至 0%（灾难性遗忘）
4. **Ablation**：去掉 Optimizer 后无法解锁钻石；去掉成熟度门控后震荡严重

### 局限性（论文自述）

- **计算资源受限**：batch-size=1，限制了并行探索
- **缺乏形式化保证**：重构的正确性靠试错，不是定理证明
- **反思机制依赖 LLM**：如果 LLM 幻觉严重，REFLECT 可能误诊

### 应用潜力

这个框架不只是游戏 AI，可以迁移到：
- **RPA (机器人流程自动化)**：自动积累和优化工作流脚本
- **DevOps**：持续学习和重构部署脚本
- **代码助手**：学习用户的编程模式并自动生成辅助函数

关键是：任何需要**增量学习可复用程序**的场景。
