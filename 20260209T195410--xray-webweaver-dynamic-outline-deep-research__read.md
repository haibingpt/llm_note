---
title: xray-webweaver-dynamic-outline-deep-research
date: 2026-02-09 19:54
tags: [read, xray, paper]
identifier: 20260209T195410
source: arXiv:2509.13312v3
authors: Zijian Li, Xin Guan, Bo Zhang, Shen Huang, Houquan Zhou, Shaopeng Lai, Ming Yan, Yong Jiang, Pengjun Xie, Fei Huang, Jun Zhang, Jingren Zhou
venue: Tongyi Lab, Alibaba Group
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Research = Planner(outline ↔ search) + Writer(§ → §)  |
|                                                          |
|   Quality = Dynamic_Outline × Hierarchical_Writing      |
|                                                          |
+----------------------------------------------------------+
```

人类做研究不是一条直线：先搜集证据，边搜边调整大纲；写作时逐段取用证据，不是把所有材料塞进上下文。

# PROBLEM

**痛点定义**: 让AI写长篇深度研究报告（100+网页，10k+ token），现有方法要么"先搜后写"导致文章空洞，要么"先写大纲"导致搜索盲目，都会产生幻觉和引用错误。

**前人困境**:
- 静态流程：搜索和大纲规划完全解耦，大纲基于LLM内部过时知识而非实际证据
- 暴力生成：把所有搜到的100k+ token原文塞进上下文，导致"迷失在中间"、引用错乱
- 单通生成：一次性写完整篇报告，无法做章节级的精细推理

# INSIGHT

**核心直觉**: 模仿人类研究员的认知流程——不要分离"探索"和"写作"，让它们像人类那样循环互动：搜到新证据→更新大纲→根据大纲继续搜索→逐章节写作。

**关键步骤**:
1. **动态研究循环（Planner）**: 每搜一轮，就用新证据优化大纲（添加章节、插入引用ID），大纲反过来指导下一轮搜什么。大纲不是静态的目录，而是"带引用的可执行计划"。
2. **分层检索写作（Writer）**: 写每个章节时，只从Memory Bank中retrieve这一节大纲里标注的引用ID对应的证据，写完就把证据从上下文踢出去。这样上下文保持小而精，避免注意力崩溃。

# DELTA

**vs SOTA**:
- DeepResearch Bench上超越所有开源+闭源系统（包括GPT-4、Claude），RACE质量50.58，引用准确率93.37%
- 比"search-then-generate"提升6+分（质量），比"outline-first"减少50%+幻觉

**新拼图**:
- 证明"动态大纲优化"比静态规划关键：2-3轮迭代后，大纲深度/广度/支撑度显著提升
- 证明"分层检索"比全文塞入有效：每步只用相关证据，输出token随写作步数线性增长而非爆炸
- 开源了WebWeaver-3k数据集：3k条完整研究轨迹，可微调小模型（Qwen3-30B）达到接近GPT-4的性能

# CRITIQUE

**隐形假设**:
- 假设搜索引擎返回的snippet足够准确，能支撑LLM做URL筛选（如果snippet质量差，前期筛选就会失效）
- 假设大纲能迭代收敛到"足够好"（论文显示平均2.16轮，但没说如果问题太开放会不会发散）
- 假设引用ID的citation机制能正确映射（依赖LLM严格遵守格式，实际可能出错）

**未解之谜**:
- 如何处理搜索引擎本身的偏见（如SEO操纵、信息茧房）？论文未讨论信息源多样性
- 多轮迭代的停止条件是什么？仅说"planner输出terminate"，缺少自动判断大纲质量的机制
- Memory Bank的存储结构是什么？只说"summary+evidence"，没说如何处理证据之间的冲突或冗余
- 成本如何？动态迭代+多步写作的token消耗和延迟对比baseline未给出

# LOGIC FLOW

```
User Query
    |
    v
+-------------------+
| Planner (Round 1) |  <--- Think: What to search?
+-------------------+
    |
    | search() --> URLs --> Parse --> Extract Evidence
    |                                       |
    v                                       v
+-------------------+              +----------------+
|   Memory Bank     | <----------- | ID_1: Evidence |
| (ID -> Evidence)  |              | ID_2: Evidence |
+-------------------+              +----------------+
    |
    | write_outline() --> Optimize outline with citations
    |
    v
+-------------------+
| Outline v1        |  Sections: [§1: <ID_1, ID_2>, §2: <ID_3>...]
+-------------------+
    |
    | (Iterate if needed)
    v
+-------------------+
| Planner (Round 2) |  <--- Think: What's missing?
+-------------------+
    | search() --> More evidence --> Memory Bank updated
    |
    v
+-------------------+
| Outline v2        |  (Refined: added §2.1, more citations)
+-------------------+
    |
    | terminate() when comprehensive
    |
    v
+---------------------------------+
|          Writer                 |
+---------------------------------+
    |
    | For each section §i:
    |   1. retrieve(§i.citations) --> Pull evidence from Memory Bank
    |   2. think() --> Analyze evidence, synthesize insights
    |   3. write(§i) --> Compose paragraph with citations
    |   4. prune() --> Remove used evidence from context
    |
    v
+-------------------+
|   Final Report    |  (Hierarchical, well-cited, comprehensive)
+-------------------+
```

# NAPKIN SKETCH

```
Traditional Approach:
  Search --> [||||||||||||] Evidence Pile --> Generate Report
                 (monolithic, lost in middle)

WebWeaver Approach:

  Query
    |
    v
  +-------+      +--------+       +----------+
  |Planner|  <-> | Search |  <->  |  Outline |  (Dynamic Co-evolution)
  +-------+      +--------+       +----------+
                                       |
                 Evidence              | (Citation-grounded)
                     |                 |
                     v                 v
              +----------------+   +--------+
              | Memory Bank    |   | Writer |
              | [ID -> Content]|   +--------+
              +----------------+       |
                     ^                 | (Hierarchical retrieval)
                     |                 |
                     +---- retrieve ---+
                              |
                              v
                        Section 1 [ID_1, ID_2]
                        Section 2 [ID_3, ID_5]
                        Section 3 [ID_7]
                              |
                              v
                     Final Report (No hallucination!)

  Key Innovation:
  - Outline is NOT static structure, but DYNAMIC PLAN with citations
  - Memory Bank is NOT dump, but INDEXED RETRIEVAL with pruning
```

---

## 深层解读

### 为什么"动态大纲"是关键创新？

传统方法的问题：
1. **Outline-first**（如STORM、WriteHere）：先生成大纲，再搜索填充。但大纲来自LLM内部知识（可能过时/错误），搜索被限制在错误的框架内。
2. **Search-first**（如WebShaper）：先搜索，收集所有信息后再生成。但没有大纲指导，搜索漫无目的，收集了大量无关内容。

WebWeaver的突破：
- 大纲和搜索**互相塑造**：搜到新证据→发现新角度→扩展大纲→指导下一轮搜索
- 大纲不是"目录"，而是**可执行计划**：每个章节标注了引用ID（如`Section 1.1 <citation>id_2, id_5</citation>`），告诉Writer去Memory Bank取哪些证据
- 实验证明：2-3轮迭代后，大纲的深度（Depth）从81.9→92.3，支撑度（Support）从51.2→73.6

### 为什么"分层检索"解决长文问题？

问题：传统long-context方法把100k+ token全塞进上下文，导致：
- **Lost in the middle**: 模型注意力集中在开头/结尾，中间证据被忽略
- **Hallucination**: 信息过载，模型"记不住"哪个证据在哪，乱引用

WebWeaver的解法：
- Writer每次只处理**当前章节相关的证据**（通过大纲的citation ID检索）
- 写完一段后，**主动pruning**：把用过的证据从上下文移除，腾出空间给下一段
- 结果：上下文长度保持稳定（不随报告长度爆炸），输出质量线性增长而非下降

论文Figure 12的关键发现：
- 分层写作的上下文token随步数缓慢增长（每步+2k）
- 暴力写作的上下文一次性达到100k，输出质量在第6步后崩溃
- 分层写作最终生成26k token报告（25步），暴力写作只能生成不到20k就开始重复

### 训练数据WebWeaver-3k的价值

问题：30B模型能否学会这套复杂流程？

方法：
1. 用GPT-4/Claude跑WebWeaver，生成3.3k条完整轨迹（平均15步搜索+2轮大纲优化+100页证据）
2. 严格过滤：只保留完整执行了整个workflow的case（剔除中途失败的）
3. SFT微调Qwen3-30B（1000 iter，学习率7e-6）

结果：
- 引用准确率从25%飙升到85.90%（学会了精确的retrieve机制）
- DeepConsult得分从4.57→6.09（学会了动态规划）
- DeepResearchGym从77.27→90.89（学会了深度推理）

**启示**：复杂的多步推理能力可以通过high-quality trajectory蒸馏到小模型，关键是数据要包含完整的"思考→工具→反思"循环。

### 隐藏的工程细节

论文没说清楚但很关键的点：
1. **URL筛选的两阶段设计**：先用LLM从搜索结果中选URL（基于title+snippet），再解析后用LLM提取evidence。这避免了解析所有URL的浪费，但依赖snippet质量。
2. **Memory Bank的"短摘要+完整证据"双层结构**：搜索时看到摘要（节省token），写作时取完整内容（保证细节）。但没说如何处理证据之间的逻辑关系。
3. **Outline的citation机制**：用特殊标签`<citation>id_1, id_2</citation>`绑定章节和证据。这要求LLM严格遵守格式，否则检索失效。论文没讨论格式错误的容错。

### 与人类研究流程的对比

论文声称"emulates human research process"，实际相似度如何？

相似之处：
- 人类确实会"边读边调整论文结构"（对应动态大纲）
- 人类写每段时会"翻到相关笔记"（对应hierarchical retrieval）

差异之处：
- 人类有**元认知**：会主动判断"这个大纲够不够好"，WebWeaver只能靠外部规则（如最多3轮）
- 人类会**批判性阅读**：发现证据之间矛盾时会深入分析，WebWeaver的Memory Bank是平铺的，没有证据质量评估
- 人类有**创造性综合**：能跨领域类比、提出新假设，WebWeaver只是"整理已有证据"

### 实验设计的亮点

1. **Ablation on outline optimization**（Fig 6-7）：对比1/2/3轮大纲迭代，证明多轮确实有收益，但边际递减（2→3轮提升很小）
2. **Hierarchical vs Brute-force writing**（Fig 10-11）：证明分层写作在所有指标上碾压暴力写作，尤其是Readability和Support
3. **LLM-as-a-judge for outline quality**（Fig 8-9）：用GPT-4-mini评估大纲的Depth/Breadth/Support，量化了"大纲质量"这个抽象概念
4. **Agentic finetuning效果**（Fig 14）：证明trajectory数据能教会小模型复杂流程，引用准确率从25%→85%是质的飞跃

### 局限性（论文没充分讨论的）

1. **成本/延迟**: 动态迭代+多步写作的token消耗是多少？实际部署中能否接受？论文只给了统计数据（15步搜索，25步写作），没给出与baseline的cost对比。
2. **错误传播**: 如果早期搜索方向错误，后续迭代能自我纠正吗？还是会强化错误？论文没有failure case分析。
3. **信息源偏见**: 依赖搜索引擎，如果某个观点在网络上占主导，系统会复制这种偏见。没有讨论多样性保障机制。
4. **泛化能力**: 三个benchmark都是"知识密集型问答"，对于需要逻辑推理/数学证明的深度研究是否适用？

### 对社区的启示

1. **范式转变**: 从"单通生成"到"多轮规划-执行循环"，这是agentic system的核心思想
2. **数据为王**: 高质量的trajectory数据（3.3k条）让30B模型逼近GPT-4，证明了SFT for reasoning的潜力
3. **结构化Memory**: 不要把所有信息扔进长上下文，用"索引+检索"机制管理知识
4. **Human-in-the-loop**: WebWeaver的设计本质是"可解释的"——大纲和引用都可审查，比黑盒生成更适合实际应用

---

## 一句话总结

把AI研究助手从"搜索引擎+摘抄机"升级为"会思考的写作者"：让大纲和证据在动态循环中共同进化，用分层检索避免信息过载，最终生成有深度、有引用、可验证的长篇报告。
