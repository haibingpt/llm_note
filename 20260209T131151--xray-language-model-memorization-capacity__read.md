---
title: xray-language-model-memorization-capacity
date: 2026-02-09 13:11
tags: [read, xray, paper]
identifier: 20260209T131151
source: arXiv:2505.24882v3
authors: John X. Morris, Chawin Sitawarin, Chuan Guo, Narine Kokhlikyan, G. Edward Suh, Alexander M. Rush, Kamalika Chaudhuri, Saeed Mahloujifar
venue: Meta FAIR, Google DeepMind, Cornell University, NVIDIA
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Capacity(model) = 3.6 bits/parameter                  |
|                                                          |
|   memU(x, θ) = H^K(x | θ) - H^K(x | θ̃)               |
|                                                          |
+----------------------------------------------------------+
```

GPT家族模型的记忆容量约为每参数3.6比特，通过Kolmogorov复杂度（压缩率）可以精确测量模型对单个数据点的非意图记忆（unintended memorization）。

# PROBLEM

**痛点定义**: 无法区分语言模型是"真的记住了训练数据"还是"学会了泛化规律"——模型能输出训练样本，可能因为死记硬背，也可能因为理解了生成规律。

**前人困境**:
- 现有定义依赖训练算法，无法只看最终模型就判断
- 提取式攻击（extraction attack）能成功不代表真记忆——模型可能是通过泛化能力生成的
- 无法量化单个样本的记忆量，只能做整体统计

# INSIGHT

**核心直觉**: 记忆 = 压缩。如果模型把一个数据点压缩得比"理想泛化模型"更短，那多出来的压缩能力就是"记住"的部分。

**关键步骤**:
1. **双组分分解**: 记忆 = 总信息 - 泛化信息
   - memU(x, θ) = mem(x, θ) - memI(x, θ)
   - 非意图记忆 = 用训练模型压缩 - 用更大oracle模型压缩

2. **合成实验隔离变量**: 在随机bitstring上训练模型（无泛化可能），直接测量纯记忆容量
   - 训练数百个不同大小的Transformer（500K到1.5B参数）
   - 在均匀分布随机序列上训练到饱和
   - 发现记忆量plateau在 3.5-3.6 bits/parameter

# DELTA

**vs SOTA**:
- 首次给出模型容量上界的精确测量（之前估计为2 bits/param）
- 提出样本级（sample-level）的记忆定义，独立于训练算法
- 发现"double descent"临界点：当数据量 = 模型容量时，记忆转为泛化

**新拼图**:
- 建立了记忆的信息论定义（基于Kolmogorov复杂度）
- 给出membership inference的scaling law：
  ```
  F1(θ, D) = 1/2(1 + c1·σ(c2·Capacity(θ)/|D| + c3))
  ```
- 证明：提升精度从bfloat16到float32，每参数多存0.5比特

# CRITIQUE

**隐形假设**:
- 假设算术编码（arithmetic coding）能近似真实Kolmogorov复杂度——但编码效率依赖模型架构
- 用更大模型作为"oracle"参考模型——但oracle本身也有capacity限制
- 实验在GPT-2架构上做，scaling law能否适用于不同架构（如Mamba、SSM）未验证
- 数据去重很关键——实验用了完全去重的数据，但真实预训练数据有1-2%重复

**未解之谜**:
- 为什么容量恰好是3.6 bits/param？理论上界在哪？
- 真实文本实验中，哪些数据点被优先记忆？发现TF-IDF高（稀有词多）的样本记得最牢，但机制不明
- 如何设计更好的压缩算法来逼近真实Kolmogorov复杂度？
- 是否能通过架构设计提升"泛化容量"占比，降低"记忆容量"占比？

# LOGIC FLOW

```
Problem: Can't distinguish memorization vs generalization
                          |
                          v
          +-------------------------------+
          | Insight: Use Compression Rate |
          +-------------------------------+
                          |
                          v
          +-------------------------------+
          |  Decompose into 2 components  |
          |  mem(x) = memU(x) + memI(x)   |
          +-------------------------------+
                          |
          +---------------+---------------+
          |                               |
          v                               v
  +----------------+            +------------------+
  | Synthetic Data |            |   Real Text      |
  | (pure memory)  |            | (mem + general)  |
  +----------------+            +------------------+
          |                               |
          v                               v
  +----------------+            +------------------+
  | Measure alpha  |            | Double descent:  |
  | 3.6 bits/param |            | D/C ratio = 1    |
  +----------------+            +------------------+
                          |
                          v
          +-------------------------------+
          |   Scaling Law for Membership  |
          |   Inference (sigmoid form)    |
          +-------------------------------+
```

# NAPKIN SKETCH

```
Unintended Memorization (bits)
  ^
  |     *  *  *  * <- Capacity plateau (3.6 bits/param)
  |    *  *  *
  |   *  *                Model size:
  |  *  *                   170K (yellow)
  | *  *                    2.3M  (blue)
  |* *                      7M    (purple)
  +--------------------------> Dataset size

Double Descent Phenomenon:

Train Loss    |    Test Loss
              |
    \         |        ----\
     \        |             \___
      \___    |                 \___
          \   |                     \___
--------------+---------------------------> Data/Capacity ratio
     ^        1        ^
     |                 |
  Memorize         Generalize
  Phase            Phase

Compression View:

  [Data sample x] ---> [Model θ]  ---> Compressed: H^K(x|θ) bits
                         |
                         v
                    [Oracle θ̃] ---> Baseline:    H^K(x|θ̃) bits

  Unintended Memorization = H^K(x|θ) - H^K(x|θ̃)
                          = "Extra compression from training"
```
