---
title: xray-attention-turing-complete
date: 2026-02-09 12:13
tags: [read, xray, paper]
identifier: 20260209T121357
source: https://jmlr.org/papers/v22/20-302.html
authors: Jorge Pérez, Pablo Barceló, Javier Marinkovic
venue: Journal of Machine Learning Research 2021
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Transformer + Hard-Attention + Positional-Encoding     |
|              + Arbitrary-Precision                       |
|                     =                                    |
|              Turing Complete                             |
|                                                          |
+----------------------------------------------------------+
```

Transformer可以通过硬注意力机制模拟图灵机的读写头，通过位置编码访问任意位置，通过任意精度内部表示存储计算状态。

# PROBLEM

**痛点定义**: Transformer作为处理序列的主流架构，其理论计算能力边界未被严格证明，不清楚它能否实现任意算法。

**前人困境**:
- RNN已被证明图灵完备（Siegelmann & Sontag 1995），但基于递归结构
- Transformer基于注意力机制，不直接按序列顺序处理，现有理论无法套用
- 实践中Transformer表现强大，但理论基础薄弱
- 已知Transformer是顺序不变的（order-invariant），无法识别正则语言如 {a,b}*中a和b数量相等的串

# INSIGHT

**核心直觉**: Transformer就像一个"可编程的并行内存访问机器"——编码器将输入写入"内存"，解码器用注意力机制当"可寻址指针"，通过位置编码实现随机访问。

**关键步骤**:
1. **硬注意力作为指针**: 用hardmax代替softmax，注意力机制变成精确的位置寻址机制（类似图灵机读写头）
2. **位置编码作为地址**: 位置编码 pos(i) = [1, i, 1/i, 1/i²] 提供可计算的绝对位置信息，支持任意精度的位置运算
3. **解码器模拟计算步骤**: 每层解码器通过自注意力存储历史状态，通过交叉注意力"读取"编码器特定位置的符号
4. **任意精度避免作弊**: 内部表示使用有理数任意精度，位置编码精度与输入长度对数相关，确保真实可实现

# DELTA

**vs SOTA**:
- RNN图灵完备基于递归+有界资源，Transformer基于注意力+位置编码
- Dehghani等人（2018）要求固定精度，本文证明任意精度下的图灵完备
- 首次形式化证明Transformer可识别所有递归可枚举语言（RE languages）

**新拼图**:
- 确立Transformer的理论计算边界：与图灵机等价
- 揭示位置编码的关键作用：不只是解决顺序不变性，更是实现随机访问的"地址总线"
- 证明硬注意力+残差连接+位置编码是图灵完备的最小充分集合
- 建立精度与可计算性的关系：T(n)-bounded Transformer可识别TIME(T(n))复杂度类

# CRITIQUE

**隐形假设**:
- **任意精度假设**: 实际硬件只支持固定精度浮点数，论文假设可用有理数表示任意精度
- **硬注意力vs软注意力**: 实践中使用softmax（软注意力），证明用hardmax（硬注意力），两者gap未分析
- **残差连接必须**: 证明依赖残差连接（+x, +a），但未探讨是否可省略
- **多层解码器**: 构造用了3层解码器，是否可优化到更少层次未知

**未解之谜**:
- 固定精度Transformer的计算能力如何？（论文用proportion-invariance反证不图灵完备，但具体能力边界不明）
- 软注意力（标准Transformer）的计算能力？（实践中从不用硬注意力）
- 位置编码的最优形式？（论文用特定公式，是否存在更简单的？）
- 单层/双层Transformer是否足够？（构造用了1层编码器+3层解码器）
- 理论与实践的鸿沟：图灵完备不保证可学习性，为何Transformer在实践中泛化良好？

# LOGIC FLOW

```
Problem: Transformer计算能力边界？
   |
   v
Observation 1: Transformer是order-invariant
   |  --> 无法识别正则语言 {w | #a(w) = #b(w)}
   |  --> 需要位置信息突破
   v
Observation 2: 硬注意力可精确定位
   |  --> Att(q,K,V) = v_j* where j* = argmin |<q,k_j>|
   |  --> 类似图灵机读写头
   v
Key Idea: Encoder存储 + Decoder计算
   |
   +---> Encoder: 用位置编码标记每个符号
   |        (K^e, V^e) = (位置, 符号+位置)
   |
   +---> Decoder Layer 1: 累积计算历史
   |        通过自注意力维护 q^(i), s^(i), m^(i)
   |        (状态, 符号, 移动方向)
   |
   +---> Decoder Layer 2: 计算下一个访问位置
   |        c^(i) = Σm^(j), 用注意力定位 c^(i)
   |
   +---> Decoder Layer 3: 读取符号
   |        attend到位置 ℓ(i+1)，复制 s^(r+1)
   |
   v
Result: Trans_M模拟图灵机M
   |
   v
Theorem: Transformer + 位置编码 = 图灵完备
```

# NAPKIN SKETCH

```
Transformer as Turing Machine Simulator

Encoder: 将输入串编码到"内存"
+-----+-----+-----+-----+-----+
| s1  | s2  | s3  | ... | sn  |  <-- 输入符号
+-----+-----+-----+-----+-----+
| p1  | p2  | p3  | ... | pn  |  <-- 位置编码
+-----+-----+-----+-----+-----+
   |     |     |     |     |
   v     v     v     v     v
(K^e, V^e) = 编码后的key-value对


Decoder: 逐步模拟图灵机计算
         +------------------+
         | y_i = [q^(i),    |  <-- 当前状态
         |        s^(i),    |  <-- 当前符号
         |        m^(i-1)]  |  <-- 上次移动
         +------------------+
                 |
         [Self-Attention]  <-- 累积历史
                 |
         [Cross-Attention] <-- 读取位置 c^(i)
                 |          通过硬注意力精确定位
                 v
         c^(i) = Σ m^(j)   <-- 计算磁头位置
                 j=0..i-1
                 |
                 v
         Attend to position c^(i) in Encoder
                 |
                 v
            复制 s^(r+1)   <-- 下一步读取的符号


关键机制：
+----------+----------+------------------------+
| 组件     | 作用     | 类比图灵机             |
+----------+----------+------------------------+
| 位置编码 | 地址     | 磁带单元格索引         |
| 硬注意力 | 读写头   | 移动并读取当前单元格   |
| 解码器层 | 控制单元 | 状态转移函数 δ         |
| 残差连接 | 状态传递 | 保持历史信息           |
| 任意精度 | 可计算性 | 避免精度限制作弊       |
+----------+----------+------------------------+
```
