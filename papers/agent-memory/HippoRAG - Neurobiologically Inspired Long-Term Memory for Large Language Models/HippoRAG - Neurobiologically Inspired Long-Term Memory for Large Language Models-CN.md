# HippoRAG：受神经生物学启发的大型语言模型长期记忆

## 译文说明

本译文依据本地 PDF 全文逐节翻译，保留原文的章节结构、公式符号、表格数值、图表编号、脚注编号、引用编号、模型名与数据集名。图中对理解方法有实质作用的文字和关系另行译出。参考文献保留原始书目信息，以免误译题名、作者姓名或出版信息。附录 I 的提示词为忠实呈现论文内容而译；若需复现实验，应以原文提示词为准。

Bernal Jiménez Gutiérrez，Yiheng Shu，Yu Gu，Michihiro Yasunaga，Yu Su

Bernal Jiménez Gutiérrez、Yiheng Shu、Yu Gu、Yu Su：The Ohio State University  
Michihiro Yasunaga：Stanford University

arXiv:2405.14831v3 [cs.CL]，2025 年 1 月 14 日

第 38 届神经信息处理系统大会（NeurIPS 2024）。

## 摘要

为了在充满威胁且不断变化的自然环境中生存繁衍，哺乳动物的大脑逐步演化出这样的能力：存储关于世界的大量知识，并在避免灾难性遗忘的同时持续整合新信息。尽管大型语言模型（LLM）已取得令人瞩目的成就，但即使结合检索增强生成（RAG），它们仍难以在预训练后高效而有效地整合大量新经验。本文提出 HippoRAG：一种受人类长期记忆的海马记忆索引理论启发的新型检索框架，旨在对新经验实现更深入、更高效的知识整合。HippoRAG 协同编排 LLM、知识图谱与个性化 PageRank 算法，以模拟新皮层和海马体在人类记忆中承担的不同角色。我们在多跳问答（QA）任务上将 HippoRAG 与现有 RAG 方法进行比较，结果表明，本方法显著优于当前最先进的方法，提升幅度最高可达 20%。HippoRAG 的单步检索能够达到与 IRCoT 等迭代检索方法相当或更好的性能，同时成本仅为其 $1/10$ 至 $1/20$，速度则快 6 至 13 倍；将 HippoRAG 集成到 IRCoT 中还能带来进一步的显著提升。最后，我们表明，本方法能够处理现有方法力所不及的新型场景。[^1]

## 1 引言

数百万年的演化使哺乳动物的大脑形成了一项关键能力：存储大量世界知识，并在不丢失旧经验的情况下持续整合新经验。这套卓越的长期记忆系统最终使人类能够保有庞大且不断更新的知识储备，而这些知识构成了推理和决策的基础 [19]。

尽管大型语言模型（LLM）近年来进展迅速，这种持续更新的长期记忆在当前 AI 系统中仍明显缺席。部分由于检索增强生成（RAG）易于使用，也部分由于模型编辑 [46] 等其他技术存在局限，RAG 已成为 LLM 长期记忆事实上的解决方案，使用户可以向静态模型提供新知识 [36, 42, 66, 87]。

然而，由于每个新段落都是彼此隔离地编码，当前 RAG 方法仍无法帮助 LLM 完成需要跨越段落边界整合新知识的任务。科学文献综述、法律案件摘要和医学诊断等许多重要现实任务，都要求跨段落或跨文档整合知识。标准多跳问答（QA）虽然复杂度稍低，但同样需要整合检索语料库中不同段落之间的信息。为解决此类任务，当前 RAG 系统不得不迭代执行多轮检索与 LLM 生成，以连接彼此分散的段落 [64, 78]。即便多步 RAG 的每一步都执行得毫无差错，它仍往往不足以完成许多知识整合场景；图 1 展示了我们称为“路径查找型多跳问题”的一类情形。

相比之下，人脑能够相对轻松地解决这类极具挑战性的知识整合任务。海马记忆索引理论 [75] 是一种得到广泛认可的人类长期记忆理论，它为这种非凡能力提供了一种合理解释。Teyler 和 Discenna [75] 提出，我们强大的、基于上下文且持续更新的记忆，依赖新皮层与 C 形海马体之间的交互：新皮层负责处理并存储实际的记忆表征；海马体则保存海马索引，即一组彼此连接的索引，它们指向新皮层中的记忆单元，并存储这些单元之间的关联 [19, 76]。

本文提出 HippoRAG：一种通过模拟上述人类记忆模型、充当 LLM 长期记忆的 RAG 框架。我们首先使用 LLM 将语料库转化为无模式知识图谱（KG），以模拟新皮层处理感知输入的能力；该图谱充当人工海马索引。面对新查询时，HippoRAG 会识别查询中的关键概念，以这些查询概念为种子，在 KG 上运行个性化 PageRank（PPR）算法 [30]，从而跨段落整合信息并完成检索。PPR 使 HippoRAG 能够探索 KG 路径并识别相关子图，实质上在一次检索步骤中完成多跳推理。

这种单步多跳检索能力，在两个常用多跳 QA 基准 MuSiQue [77] 和 2WikiMultiHopQA [33] 上，分别比当前 RAG 方法 [10, 35, 53, 70, 71] 高出约 3 个点和 20 个点。此外，HippoRAG 的在线检索过程比 IRCoT [78] 等当前迭代检索方法便宜 10 至 30 倍、快 6 至 13 倍，同时仍能取得相当的性能。进一步地，我们的方法可与 IRCoT 结合，在同一批数据集上分别带来最高 4% 和 20% 的互补提升；即便在难度较低的多跳 QA 数据集 HotpotQA 上，也能获得改进。最后，我们通过案例研究展示当前方法在前述路径查找型多跳 QA 场景中的局限，以及本方法的潜力。

> **图 1 内文字：** 离线索引；在线检索；当前 RAG；人类记忆；HippoRAG；“哪位 Stanford 教授从事 Alzheimer’s 神经科学研究？”；答案：Thomas 教授。

**图 1：知识整合与 RAG。** 对当前 RAG 系统而言，需要知识整合的任务尤其困难。在图示示例中，我们希望从描述潜在数千名 Stanford 教授和 Alzheimer’s 研究者的段落池中，找出一位从事 Alzheimer’s 研究的 Stanford 教授。由于当前方法彼此隔离地编码各个段落，除非某个段落同时提及这两个特征，否则它们很难识别 Thomas 教授。相比之下，大多数熟悉这位教授的人都能迅速想起他，这得益于人脑的联想记忆能力；一般认为，这种能力由上图蓝色 C 形海马体中所示的索引结构驱动。受这一机制启发，HippoRAG 使 LLM 能够构建并利用类似的关联图，以处理知识整合任务。

[^1]: 代码与数据见 https://github.com/OSU-NLP-Group/HippoRAG 。

## 2 HippoRAG

本节首先简要概述海马记忆索引理论，随后说明 HippoRAG 的索引与检索设计如何受该理论启发，最后详细介绍本方法。

### 2.1 海马记忆索引理论

海马记忆索引理论 [75] 是一种得到广泛认可的理论，它从功能角度描述了人类长期记忆所涉及的组件和神经回路。在该理论中，Teyler 和 Discenna [75] 提出，人类长期记忆由三个协同工作的组件组成，用以实现两个主要目标：一是**模式分离**，确保不同感知经验的表征具有唯一性；二是**模式补全**，使完整记忆能够从部分刺激中被检索出来 [19, 76]。

该理论认为，模式分离主要在记忆编码过程中完成。该过程始于新皮层接收感知刺激，并将其加工成更易操纵、可能也更高层次的特征；随后，这些特征经过海马旁区（parahippocampal regions，PHR），被送往海马体建立索引。当信号到达海马体后，显著信号会被纳入海马索引，并彼此建立关联。

记忆编码完成后，只要海马体从 PHR 通路接收到部分感知信号，模式补全就会驱动记忆检索过程。随后，海马体利用其上下文依赖型记忆系统——一般认为该系统由 CA3 亚区中一个稠密连接的神经元网络实现 [76]——在海马索引中识别完整且相关的记忆，并经由 PHR 将其回传至新皮层进行模拟。因此，这套复杂过程只需改变海马索引，而无须更新新皮层表征，便能整合新信息。

### 2.2 概览

我们提出的 HippoRAG 与上述过程高度对应。如图 2 所示，本方法的每个组件都对应人类长期记忆的三个组件之一。附录 A 给出了 HippoRAG 全流程的详细示例。

**离线索引。** 离线索引阶段对应记忆编码。该阶段首先利用一个经过强指令调优的 LLM——即我们的人工新皮层——抽取知识图谱（KG）三元组。该 KG 不预设模式，这一过程称为开放信息抽取（OpenIE）[3, 5, 60, 98]。它把检索语料库段落中的显著信号抽取为离散名词短语，而非稠密向量表征，因此能够实现更细粒度的模式分离。于是，我们自然地将人工海马索引定义为这张开放 KG，并逐段处理整个检索语料库来构建它。最后，为了像海马旁区那样连接前述两个组件，我们使用针对检索任务微调的现成稠密编码器（下称“检索编码器”）。这些检索编码器会在 KG 中相似但不完全相同的名词短语之间添加额外边，以帮助后续进行模式补全。

**在线检索。** 随后，系统复用这三个组件，模拟人脑的记忆检索过程来执行在线检索。正如海马体接收经新皮层和 PHR 处理的输入，我们的 LLM 新皮层会从查询中抽取一组显著命名实体，称为**查询命名实体**。这些命名实体再依据检索编码器计算的相似度，与 KG 中的节点进行链接；我们将选中的节点称为**查询节点**。查询节点一经确定，便成为合成海马体执行模式补全所依据的部分线索。在海马体中，海马索引各元素之间的神经通路会激活相关邻域，并将其向上游回忆。为了模拟这一高效的图搜索过程，我们采用个性化 PageRank（PPR）算法 [30]。PPR 是 PageRank 的一个变体，它只通过一组由用户指定的源节点在图中分配概率。这一约束使 PPR 的输出仅向查询节点偏置，正如海马体依据特定的部分线索提取关联信号一样。[^2] 最后，如同海马信号被送往上游，我们将 PPR 输出的节点概率聚合到先前建立索引的各个段落上，据此为段落排序并完成检索。

> **图 2 内文字：** 三列依次为“新皮层 / LLM”“海马旁区 / 检索编码器”“海马体 / KG + 个性化 PageRank”；两行依次为“离线索引”和“在线检索”。离线阶段：段落经 OpenIE 形成三元组，例如 `(Thomas, researches, Alzheimer’s)` 与 `(Stanford, employs, Thomas)`，检索编码器识别同义关系，三元组被整合到 KG。在线阶段：查询经 NER 抽取 `Stanford` 与 `Alzheimer’s`，检索编码器将其链接到 KG；节点特异性调整初始权重，随后 PPR 在图上扩散并定位 Thomas 教授。

**图 2：HippoRAG 的详细方法。** 我们对人类长期记忆的三个组件进行建模，以模拟其模式分离和模式补全功能。在离线索引阶段（中部），我们使用 LLM 将段落处理成开放 KG 三元组，并将其加入人工海马索引；与此同时，合成的海马旁区（PHR）负责识别同义关系。在上面的示例中，系统抽取涉及 Thomas 教授的三元组并将其整合到 KG 中。在在线检索阶段（底部），LLM 新皮层从查询中抽取命名实体，海马旁检索编码器再将这些实体链接至海马索引。然后，我们利用个性化 PageRank 算法实现基于上下文的检索，并找出 Thomas 教授。[^4]

[^2]: 有趣的是，一些认知科学研究也发现，人类的词语回忆与 PageRank 算法的输出之间存在相关性 [25]。
[^4]: 为简化论述，本文省略了海马记忆索引理论的许多细节。建议感兴趣的读者进一步查阅第 2.1 节所引文献。

### 2.3 详细方法

**离线索引。** 索引过程使用一个经过指令调优的 LLM $L$ 和一个检索编码器 $M$，处理段落集合 $P$。如图 2 所示，我们首先通过 OpenIE，使用 $L$ 从 $P$ 中的每个段落抽取一组名词短语节点 $N$ 和关系边 $E$。该过程以附录 I 所示提示词对 LLM 进行单样本提示。具体而言，我们先从每个段落中抽取一组命名实体，再将这些命名实体加入 OpenIE 提示词，以抽取最终三元组；这些三元组除命名实体外，还包含一般概念（名词短语）。我们发现，这种两步提示配置在通用性与偏向命名实体之间取得了适当平衡。最后，当 $N$ 中两个实体表征的余弦相似度高于阈值 $\tau$ 时，我们使用 $M$ 添加前述额外同义关系集合 $E'$。如上所述，这会为海马索引引入更多边，从而实现更有效的模式补全。该索引过程定义了一个 $|N|\times|P|$ 的矩阵 $\mathbf{P}$，其中记录 KG 中每个名词短语在各原始段落中出现的次数。

**在线检索。** 检索期间，我们用一个单样本提示来提示 $L$，从查询 $q$ 中抽取一组命名实体，即先前定义的查询命名实体：

$$
C_q=\{c_1,\ldots,c_n\}.
$$

在图 2 的示例中，它们是 `Stanford` 与 `Alzheimer’s`。随后，使用同一个检索编码器 $M$ 对查询中的这些命名实体 $C_q$ 进行编码。接着，从 $N$ 中选取与查询命名实体 $C_q$ 余弦相似度最高的节点，作为先前定义的查询节点。更形式化地，查询节点定义为 $R_q=\{r_1,\ldots,r_n\}$，其中：

$$
r_i=e_k,\qquad
k=\arg\max_j\operatorname{cosine\_similarity}\!\left(M(c_i),M(e_j)\right).
$$

图 2 使用 Stanford 标志和 Alzheimer’s 紫丝带符号表示这些节点。

找到查询节点 $R_q$ 后，我们在海马索引上运行 PPR 算法。该索引是一张拥有 $|N|$ 个节点、$|E|+|E'|$ 条边（三元组边与同义关系边）的 KG。算法使用定义在 $N$ 上的个性化概率分布 $\vec{n}$：每个查询节点获得相同概率，其他节点的概率均为零。这样，概率质量便会主要流向查询节点的联合邻域，例如 Thomas 教授节点，并最终促成检索。运行 PPR 后，我们得到定义在 $N$ 上的更新概率分布 $\vec{n}'$。最后，为获得段落分数，我们将 $\vec{n}'$ 与前述矩阵 $\mathbf{P}$ 相乘，得到 $\vec{p}$，即每个段落的排序分数，并据此进行检索。

**节点特异性。** 我们引入节点特异性，以一种在神经生物学上合理的方式进一步改善检索。众所周知，逆文档频率（IDF）等反映词语重要性的全局信号能够改善信息检索。然而，要让人脑利用 IDF 进行检索，就必须在记忆检索结束前，将已经编码的“段落”总数与所有节点的激活情况进行聚合。这对普通计算机很简单，但对大脑而言，每次检索都要激活一个聚合神经元与海马索引所有节点之间的连接，很可能带来难以承受的计算开销。

鉴于这些约束，我们提出节点特异性，作为一种仅需局部信号、因而在神经生物学上更合理的 IDF 替代信号。节点 $i$ 的节点特异性定义为：

$$
s_i=|P_i|^{-1},
$$

其中，$P_i$ 是 $P$ 中抽取出节点 $i$ 的段落集合；这一信息在每个节点处本就可得。在检索中，我们会在运行 PPR 前，将每个查询节点在 $\vec{n}$ 中的概率乘以 $s_i$；这样既可以调节节点自身的概率，也能调节其邻域的概率。图 2 以符号的相对大小呈现节点特异性：Stanford 标志比 Alzheimer’s 符号变得更大，因为前者出现于更少的文档中。

## 3 实验设置

### 3.1 数据集

我们主要在两个具有挑战性的多跳 QA 基准上评估本方法的检索能力：MuSiQue（answerable 子集）[77] 与 2WikiMultiHopQA [33]。为求完整，我们还纳入 HotpotQA [89]；不过，已有研究发现，由于该数据集包含大量伪相关信号，它对多跳推理的测试力度弱得多 [77]，附录 B 也展示了这一点。为控制实验成本，我们沿用既有工作 [63, 78] 的做法，从每个验证集中抽取 1,000 个问题。为了构建更真实的检索环境，我们遵循 IRCoT [78]，收集所选问题的全部候选段落，包括支持段落与干扰段落，并为每个数据集构建一个检索语料库。数据集详情见表 1。

**表 1：三个 1,000 问题开发集的检索语料库与所抽取 KG 的统计信息。**

| 统计项 | MuSiQue | 2Wiki | HotpotQA |
|---|---:|---:|---:|
| 段落数（$P$） | 11,656 | 6,119 | 9,221 |
| 唯一节点数（$N$） | 91,729 | 42,694 | 82,157 |
| 唯一边数（$E$） | 21,714 | 7,867 | 17,523 |
| 唯一三元组数 | 107,448 | 50,671 | 98,709 |
| Contriever 同义边数（$E'$） | 145,990 | 146,020 | 159,112 |
| ColBERTv2 同义边数（$E'$） | 191,636 | 82,526 | 171,856 |

### 3.2 基线

我们与多种强大且广泛使用的检索方法进行比较：BM25 [69]、Contriever [35]、GTR [53] 和 ColBERTv2 [70]。此外，我们还与两个近期的 LLM 增强基线比较：其一是将段落改写为命题的 Propositionizer [10]，其二是构建摘要节点以简化长文档检索的 RAPTOR [71]。除上述单步检索方法外，我们还纳入多步检索方法 IRCoT [78] 作为基线。

### 3.3 指标

在上述数据集上，我们使用 recall@2 和 recall@5（下文简写为 R@2 与 R@5）评估检索性能，并使用精确匹配（EM）和 F1 分数评估 QA 性能。

**表 2：单步检索性能。** HippoRAG 在 MuSiQue 与 2WikiMultiHopQA 上优于全部基线，并在难度较低的 HotpotQA 数据集上取得相当的性能。

| 方法 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 | HotpotQA R@2 | HotpotQA R@5 | 平均 R@2 | 平均 R@5 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| BM25 [69] | 32.3 | 41.2 | 51.8 | 61.9 | 55.4 | 72.2 | 46.5 | 58.4 |
| Contriever [35] | 34.8 | 46.6 | 46.6 | 57.5 | 57.2 | 75.5 | 46.2 | 59.9 |
| GTR [53] | 37.4 | 49.1 | 60.2 | 67.9 | 59.4 | 73.3 | 52.3 | 63.4 |
| ColBERTv2 [70] | 37.9 | 49.2 | 59.2 | 68.2 | **64.7** | **79.3** | 53.9 | 65.6 |
| RAPTOR [71] | 35.7 | 45.3 | 46.3 | 53.8 | 58.1 | 71.2 | 46.7 | 56.8 |
| RAPTOR（ColBERTv2） | 36.9 | 46.5 | 57.3 | 64.7 | 63.1 | 75.6 | 52.4 | 62.3 |
| Proposition [10] | 37.6 | 49.3 | 56.4 | 63.1 | 58.7 | 71.1 | 50.9 | 61.2 |
| Proposition（ColBERTv2） | 37.8 | 50.1 | 55.9 | 64.9 | 63.9 | 78.1 | 52.5 | 64.4 |
| HippoRAG（Contriever） | **41.0** | **52.1** | **71.5** | **89.5** | 59.0 | 76.2 | 57.2 | 72.6 |
| HippoRAG（ColBERTv2） | 40.9 | 51.9 | 70.7 | 89.1 | 60.5 | 77.7 | **57.4** | **72.9** |

**表 3：多步检索性能。** 将 HippoRAG 与 IRCoT 等标准多步检索方法结合，可在三个数据集上获得强劲的互补提升。

| 方法 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 | HotpotQA R@2 | HotpotQA R@5 | 平均 R@2 | 平均 R@5 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| IRCoT + BM25（默认） | 34.2 | 44.7 | 61.2 | 75.6 | 65.6 | 79.0 | 53.7 | 66.4 |
| IRCoT + Contriever | 39.1 | 52.2 | 51.6 | 63.8 | 65.9 | 81.6 | 52.2 | 65.9 |
| IRCoT + ColBERTv2 | 41.7 | 53.7 | 64.1 | 74.4 | **67.9** | 82.0 | 57.9 | 70.0 |
| IRCoT + HippoRAG（Contriever） | 43.9 | 56.6 | 75.3 | 93.4 | 65.8 | 82.3 | 61.7 | 77.4 |
| IRCoT + HippoRAG（ColBERTv2） | **45.3** | **57.6** | **75.8** | **93.9** | 67.0 | **83.0** | **62.7** | **78.2** |

### 3.4 实现细节

默认情况下，我们使用温度为 0 的 GPT-3.5-turbo-1106 [55] 作为 LLM $L$，并使用 Contriever [35] 或 ColBERTv2 [70] 作为检索器 $M$。我们使用 MuSiQue 训练数据中的 100 个样本调节 HippoRAG 的两个超参数：同义关系阈值 $\tau$ 设为 0.8；PPR 阻尼系数设为 0.5，它决定 PPR 从查询节点重新启动随机游走、而不是继续探索图的概率。总体而言，我们发现 HippoRAG 对超参数相当稳健。更多实现细节见附录 H。

## 4 结果

下面给出检索与 QA 实验结果。由于本方法对 QA 性能的影响是间接的，我们在表现最佳的检索骨干 ColBERTv2 [70] 上报告 QA 结果；与此同时，我们针对多种强大的单步与多步检索技术报告检索结果。

**单步检索结果。** 如表 2 所示，在主要数据集 MuSiQue 和 2WikiMultiHopQA 上，HippoRAG 优于所有其他方法，包括 Propositionizer 和 RAPTOR 等近期 LLM 增强基线；在 HotpotQA 上则取得有竞争力的性能。尤其值得注意的是，在 2WikiMultiHopQA 上，R@2 与 R@5 分别提高了 11% 和 20%；在 MuSiQue 上则提高约 3%。造成这一差异的部分原因，是 2WikiMultiHopQA 采用以实体为中心的设计，与 HippoRAG 尤为契合。我们在 HotpotQA 上的性能较低，主要因为该数据集对知识整合的要求较低（见附录 B），也因为存在概念-上下文权衡；附录 F.2 介绍的集成技术可缓解后一个问题。

**多步检索结果。** 对于多步或迭代检索，表 3 的实验表明 IRCoT [78] 与 HippoRAG 可以互补。将 HippoRAG 用作 IRCoT 的检索器后，MuSiQue 的 R@5 继续提高约 4%，2WikiMultiHopQA 提高 18%，HotpotQA 也再提高 1%。

**表 4：QA 性能。** HippoRAG 在单步检索（第 1-3 行）和多步检索（第 4-5 行）上的 QA 改进，与其检索改进相对应。

| 检索器 | MuSiQue EM | MuSiQue F1 | 2Wiki EM | 2Wiki F1 | HotpotQA EM | HotpotQA F1 | 平均 EM | 平均 F1 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 无 | 12.5 | 24.1 | 31.0 | 39.6 | 30.4 | 42.8 | 24.6 | 35.5 |
| ColBERTv2 | 15.5 | 26.4 | 33.4 | 43.3 | 43.4 | 57.7 | 30.8 | 42.5 |
| HippoRAG（ColBERTv2） | 19.2 | 29.8 | 46.6 | 59.5 | 41.8 | 55.0 | 35.9 | 48.1 |
| IRCoT（ColBERTv2） | 19.1 | 30.5 | 35.4 | 45.1 | 45.5 | 58.4 | 33.3 | 44.7 |
| IRCoT + HippoRAG（ColBERTv2） | **21.9** | **33.3** | **47.7** | **62.7** | **45.7** | **59.2** | **38.4** | **51.7** |

**问答结果。** 表 4 报告 HippoRAG、最强检索基线 ColBERTv2 与 IRCoT，以及使用 HippoRAG 作为检索器的 IRCoT 的 QA 结果。符合预期的是，在使用同一 QA 阅读器时，单步和多步设置下检索性能的改善，分别使 MuSiQue、2WikiMultiHopQA 与 HotpotQA 的总体 F1 最多提高 3%、17% 和 1%。值得注意的是，单步 HippoRAG 达到或超过 IRCoT，同时在线检索便宜 10 至 30 倍、快 6 至 13 倍（附录 G）。

## 5 讨论

### 5.1 HippoRAG 为何有效？

**OpenIE 替代方案。** 为判断维持性能提升是否必须使用 GPT-3.5 这样的闭源模型，我们分别用端到端 OpenIE 模型 REBEL [34]，以及强大的开放权重 LLM Llama-3.1 的 8B 和 70B 指令调优版本 [1] 取代它。如表 5 第 2 行所示，使用 REBEL 构建 KG 会造成大幅性能下降，这凸显了 LLM 灵活性的重要性。具体而言，GPT-3.5 生成的三元组数量是 REBEL 的两倍，说明 REBEL 倾向于不生成含一般概念的三元组，从而遗漏许多有用关联。

就开放权重 LLM 而言，表 5 第 3-4 行表明，除性能显著下降的 2Wiki 外，Llama-3.1-8B 在其余数据集上的表现均可与 GPT-3.5 竞争。更强的 70B 版本则在三个数据集中的两个上超过 GPT-3.5，并在 2Wiki 上仍具竞争力。Llama-3.1-70B 的强劲表现，乃至 8B 模型也能取得相当表现，都令人鼓舞，因为它们为大规模语料库索引提供了成本更低的替代方案。这些 OpenIE 替代模型对应的图统计信息见附录 C。

为更深入理解 OpenIE 与检索性能之间的关系，我们从 MuSiQue 训练集的 20 个样本中抽取了 239 个金标准三元组，随后使用 CaRB [6] 框架开展小规模 OpenIE 内在评估。结果显示，两种 Llama-3.1-Instruct 模型在该内在评估上略逊于 GPT-3.5，但所有 LLM 都远远优于 REBEL。实验详情见附录 D。

**表 5：剖析 HippoRAG。** 为理解 HippoRAG 表现出色的原因，我们用合理替代方案分别替换 OpenIE 模块与 PPR，并消融节点特异性和同义关系边。

| 设置 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 | HotpotQA R@2 | HotpotQA R@5 | 平均 R@2 | 平均 R@5 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| HippoRAG | 40.9 | 51.9 | 70.7 | 89.1 | 60.5 | 77.7 | 57.4 | 72.9 |
| OpenIE：REBEL [34] | 31.7 | 39.6 | 63.1 | 76.5 | 43.9 | 59.2 | 46.2 | 58.4 |
| OpenIE：Llama-3.1-8B-Instruct [1] | 40.8 | 51.9 | 62.5 | 77.5 | 59.9 | 75.1 | 54.4 | 67.8 |
| OpenIE：Llama-3.1-70B-Instruct [1] | **41.8** | **53.7** | 68.8 | 85.3 | **60.8** | **78.6** | 57.1 | 72.5 |
| PPR：仅 $R_q$ 节点 | 37.1 | 41.0 | 59.1 | 61.4 | 55.9 | 66.2 | 50.7 | 56.2 |
| PPR：$R_q$ 节点及其邻居 | 25.4 | 38.5 | 53.4 | 74.7 | 47.8 | 64.5 | 42.2 | 59.2 |
| 无节点特异性 | 37.6 | 50.2 | 70.1 | 88.8 | 56.3 | 73.7 | 54.7 | 70.9 |
| 无同义关系边 | 40.2 | 50.2 | 69.2 | 85.6 | 59.1 | 75.7 | 56.2 | 70.5 |

**PPR 替代方案。** 如表 5 第 5-6 行所示，为考察结果在多大程度上源于 PPR 的能力，我们用两种方案替换 PPR 输出：第一种是查询节点概率 $\vec{n}$ 乘以节点特异性值（第 5 行）；第二种还会向每个查询节点的直接邻居分配少量概率（第 6 行）。首先我们发现，与两种简单基线相比，PPR 在三个数据集上都能更有效地将关联信息纳入检索。有趣的是，不使用 PPR 而直接加入 $R_q$ 节点的邻域，性能反而比只使用查询节点本身更差。

**消融实验。** 如表 5 第 7-8 行所示，节点特异性在 MuSiQue 和 HotpotQA 上带来可观提升，但在 2WikiMultiHopQA 上几乎没有影响。这可能是因为 2WikiMultiHopQA 依赖命名实体，而这些实体在词项权重方面差异很小。相反，同义关系边对 2WikiMultiHopQA 的影响最大，这表明当绝大多数相关概念都是命名实体时，即使带有噪声的实体规范化也很有用；若进一步改进同义关系检测，或许也能在其他数据集上取得更强性能。

### 5.2 HippoRAG 的优势：单步多跳检索

相比传统 RAG 方法，HippoRAG 在多跳 QA 上的一项主要优势，是能够在单个步骤中完成多跳检索。我们通过测量成功检索全部支持段落的问题占比来证明这一点；只有成功进行多跳推理，才能完成这一任务。表 6 表明，在取 top-5 段落时，HippoRAG 与 ColBERTv2 的差距在 MuSiQue 上由 3% 进一步扩大至 6%，在 2WikiMultiHopQA 上则由 20% 扩大至 38%。这说明，大幅改进主要源自获得了全部支持文档，而非只在更多问题上完成部分检索。

**表 6：全召回指标。** 我们测量成功检索出全部支持段落的问题占比（全召回，记作 AR@2 或 AR@5），发现 HippoRAG 的性能提升更为显著。

| 方法 | MuSiQue AR@2 | MuSiQue AR@5 | 2Wiki AR@2 | 2Wiki AR@5 | HotpotQA AR@2 | HotpotQA AR@5 | 平均 AR@2 | 平均 AR@5 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| ColBERTv2 [70] | 6.8 | 16.1 | 25.1 | 37.1 | 33.3 | **59.0** | 21.7 | 37.4 |
| HippoRAG | **10.2** | **22.4** | **45.4** | **75.7** | **33.8** | 57.9 | **29.8** | **52.0** |

表 7 的第一个示例进一步说明了 HippoRAG 独特的单步多跳检索能力。在该例中，尽管 Alhandra 未出现在 Vila de Xira 的段落中，HippoRAG 仍可直接利用 Vila de Xira 与 Alhandra 的出生地关系，判断其重要性；标准 RAG 方法无法直接做到这一点。此外，尽管 IRCoT 也能解决该多跳检索问题，但就在线检索而言——这可以说是面向终端用户提供服务时最关键的因素——它的成本是本方法的 10 至 30 倍，速度则慢 6 至 13 倍，详见附录 G。

**表 7：多跳问题类型。** 不同方法在路径跟随型与路径查找型多跳问题上的示例结果。

| 类型 | 问题 | HippoRAG | ColBERTv2 | IRCoT |
|---|---|---|---|---|
| 路径跟随型 | Alhandra 出生在哪个区？ | 1. Alhandra；2. Vila de Xira；3. Portugal | 1. Alhandra；2. Dimuthu Abayakoon；3. Ja‘ar | 1. Alhandra；2. Vila de Xira；3. Póvoa de Santa Iria |
| 路径查找型 | 哪位 Stanford 教授从事 Alzheimer’s 神经科学研究？ | 1. Thomas Südhof；2. Karl Deisseroth；3. Robert Sapolsky | 1. Brian Knutson；2. Eric Knudsen；3. Lisa Giocomo | 1. Brian Knutson；2. Eric Knudsen；3. Lisa Giocomo |

### 5.3 HippoRAG 的潜力：路径查找型多跳检索

表 7 的第二个示例也出现在图 1 中，它展示了一类对掌握相关知识的人类而言轻而易举、但未经额外训练的当前检索器无法处理的问题。我们将其称为**路径查找型多跳问题**：当存在许多可探索路径时，这类问题要求从一组实体之间找出一条路径，而不是像标准多跳问题那样沿一条特定路径前进。[^5]

具体而言，对于第一个问题，简单的迭代过程可以沿着 Alhandra 唯一出生地所确定的单一路径，检索出恰当段落；IRCoT 的完美表现印证了这一点。然而，面对第二个问题，迭代过程会因为可探索路径过多而陷入困难：可以从 Stanford University 的教授出发，也可以从从事 Alzheimer’s 神经科学研究的教授出发。只有将关于 Thomas Südhof 的分散信息关联起来，了解这位教授的人才能轻松回答。表 7 显示，ColBERTv2 与 IRCoT 都无法访问这些关联，因而未能抽取必要段落。HippoRAG 则利用海马索引中的关联网络与图搜索算法，判断 Thomas 教授与查询相关，并恰当地检索其段落。附录 E 的案例研究给出了更多路径查找型多跳问题。

[^5]: 当 `Stanford` 和 `Alzheimer’s` 等搜索实体恰好没有共同出现在同一段落中时，路径查找型问题便需要知识整合；对于新信息，这一条件经常成立。

## 6 相关工作

### 6.1 LLM 长期记忆

**参数化长期记忆。** 即使是持怀疑态度的研究者，也普遍承认现代 LLM 的参数编码了数量惊人的世界知识 [2, 12, 23, 28, 31, 39, 62, 79]，而 LLM 能以灵活、稳健的方式利用这些知识 [81, 83, 93]。然而，更新这一庞大知识库的能力——任何长期记忆系统不可或缺的一环——至今仍出乎意料地有限。尽管已经出现许多更新 LLM 的技术，例如标准微调、模型编辑 [15, 49, 50, 51, 52, 95]，乃至受人类记忆启发的外部参数化记忆模块 [58, 82, 32]，但尚无任何方法成为 LLM 持续学习的稳健解决方案 [26, 46, 97]。

**作为长期记忆的 RAG。** 另一方面，将 RAG 方法用作长期记忆系统，为随时间更新知识提供了一条简单途径 [36, 42, 66, 73]。更复杂的 RAG 方法会执行多轮检索和 LLM 生成，甚至能够跨越新增或更新后的知识元素整合信息 [38, 64, 72, 78, 88, 90, 92]；这同样是长期记忆系统的关键方面。然而，如前所述，这种在线信息整合无法解决路径查找型多跳 QA 示例所代表的更复杂知识整合任务。

RAPTOR [71]、MemWalker [9] 和 GraphRAG [18] 等其他方法与 HippoRAG 类似，会在离线索引阶段整合信息，因此可能有能力处理这些更复杂的任务。然而，这些方法通过摘要知识元素来整合信息，这意味着每次加入新数据都必须重新执行摘要过程。HippoRAG 则只需向 KG 添加边，即可持续整合新知识。

**作为长期记忆的长上下文。** 过去一年中，开放模型与闭源 LLM 的上下文长度均大幅增长 [11, 17, 22, 61, 68]。这种扩展趋势似乎意味着，未来 LLM 可以在超大上下文窗口中完成长期记忆存储。然而，考虑到其中涉及的众多工程障碍，以及长上下文 LLM 即便在当前上下文长度内也显现出的局限 [41, 45, 96, 21]，这种未来是否可行仍存在很大不确定性。

### 6.2 多跳 QA 与图

以往许多工作也使用图结构处理多跳 QA。这些研究大致分为两类：第一类是**图增强阅读理解**，即从检索到的文档中抽取图，并用它改善模型的推理过程；第二类是**图增强检索**，即让模型遍历图结构来寻找相关文档。

**图增强阅读理解。** 这一类别中的早期工作主要是监督方法，通过图神经网络（GNN）将超链接图或共现图的信号与语言模型融合 [20, 67, 65]。近期工作则使用 LLM，并将知识图谱三元组直接加入 LLM 提示词 [57, 43, 47]。尽管这些工作与 HippoRAG 一样使用图来处理多跳 QA，但它们的改进发生在生成环节，与完全依靠改善检索的 HippoRAG 充分互补。

**图增强检索。** 在第二类工作中，以往方法会训练一个重排序模块，使其能够遍历由 Wikipedia 超链接构建的图 [16, 100, 54, 14, 4, 44]。相比之下，HippoRAG 使用 LLM 从零构建 KG，并在没有任何监督的情况下执行多跳检索，因此适应性更强。

### 6.3 LLM 与 KG

多年来，结合语言模型与知识图谱的优势一直是一项活跃的研究方向：既包括以不同方式利用 KG 增强 LLM [48, 80, 84]，也包括从 LLM 的参数知识中蒸馏知识 [7, 85] 或使用 LLM 直接解析文本 [8, 29, 94] 来增强 KG。Pan 等人 [56] 在一篇极其全面的综述中给出了这一研究方向的路线图，并强调了协同利用这两项重要技术的研究价值 [37, 74, 27, 91, 99]。与这些工作一样，HippoRAG 展示了二者协同的潜力：将 LLM 的知识图谱构建能力与结构化知识的检索优势结合起来，实现更有效的 RAG。

## 7 结论与局限

我们提出的方法遵循神经生物学原理，虽然简单，却已经展现出克服标准 RAG 系统固有局限、同时保留其相对参数化记忆优势的潜力。HippoRAG 的知识整合能力，既体现在路径跟随型多跳 QA 上的强劲结果，也体现在路径查找型多跳 QA 上展现的潜力；再加上显著的效率提升以及持续更新的特性，使它成为介于标准 RAG 方法和参数化记忆之间的有力中间框架，并为 LLM 长期记忆提供了一个颇具吸引力的解决方案。

尽管如此，未来仍可解决若干局限，使 HippoRAG 更好地实现这一目标。首先，HippoRAG 当前的所有组件都是直接使用现成模型，没有额外训练。因此，通过针对具体组件进行微调，本方法的实际可行性仍有很大提升空间。附录 F 的错误分析清楚说明了这一点：系统的大部分错误源于 NER 和 OpenIE，因而可能从直接微调中受益。其余错误是图搜索错误；同一附录也指出，简单 PPR 仍有多条改进路径，例如允许关系直接引导图遍历。此外，如附录 F.4 所示，还需要进一步改善 OpenIE 在长文档与短文档之间的一致性。最后，也是最重要的一点，HippoRAG 的可扩展性仍需进一步验证。虽然我们表明 Llama-3.1 能够取得与闭源模型相近的性能，从而大幅降低成本，但当合成海马索引的规模远超当前基准时，其效率与有效性尚未得到实证证明。

## 致谢

作者感谢 OSU NLP 小组的同事及 Percy Liang 提出的宝贵意见。本研究部分获得 NSF OAC 2112606、NIH R01LM014199、ARL W911NF2220144 与 Cisco 的支持。本文所述观点与结论仅代表作者，不应被解释为美国政府明示或暗示的官方政策。无论本文包含何种版权声明，美国政府均获授权为政府用途复制和分发本文重印本。

## 参考文献

以下书目信息按原文完整保留。

[1] AI@Meta. Llama 3 model card. 2024. URL https://github.com/meta-llama/llama3/blob/main/MODEL_CARD.md.

[2] B. AlKhamissi, M. Li, A. Celikyilmaz, M. T. Diab, and M. Ghazvininejad. A review on language models as knowledge bases. ArXiv, abs/2204.06031, 2022. URL https://arxiv.org/abs/2204.06031.

[3] G. Angeli, M. J. Johnson Premkumar, and C. D. Manning. Leveraging linguistic structure for open domain information extraction. In C. Zong and M. Strube, editors, Proceedings of the 53rd Annual Meeting of the Association for Computational Linguistics and the 7th International Joint Conference on Natural Language Processing (Volume 1: Long Papers), pages 344–354, Beijing, China, July 2015. Association for Computational Linguistics. doi: 10.3115/v1/P15-1034. URL https://aclanthology.org/P15-1034.

[4] A. Asai, K. Hashimoto, H. Hajishirzi, R. Socher, and C. Xiong. Learning to retrieve reasoning paths over wikipedia graph for question answering. In International Conference on Learning Representations, 2020. URL https://openreview.net/forum?id=SJgVHkrYDH.

[5] M. Banko, M. J. Cafarella, S. Soderland, M. Broadhead, and O. Etzioni. Open information extraction from the web. In Proceedings of the 20th International Joint Conference on Artifical Intelligence, IJCAI’07, page 2670–2676, San Francisco, CA, USA, 2007. Morgan Kaufmann Publishers Inc.

[6] S. Bhardwaj, S. Aggarwal, and Mausam. CaRB: A crowdsourced benchmark for open IE. In K. Inui, J. Jiang, V. Ng, and X. Wan, editors, Proceedings of the 2019 Conference on Empirical Methods in Natural Language Processing and the 9th International Joint Conference on Natural Language Processing (EMNLP-IJCNLP), pages 6262–6267, Hong Kong, China, Nov. 2019. Association for Computational Linguistics. doi: 10.18653/v1/D19-1651. URL https://aclanthology.org/D19-1651.

[7] A. Bosselut, H. Rashkin, M. Sap, C. Malaviya, A. Celikyilmaz, and Y. Choi. COMET: Commonsense transformers for automatic knowledge graph construction. In A. Korhonen, D. Traum, and L. Màrquez, editors, Proceedings of the 57th Annual Meeting of the Association for Computational Linguistics, pages 4762–4779, Florence, Italy, July 2019. Association for Computational Linguistics. doi: 10.18653/v1/P19-1470. URL https://aclanthology.org/P19-1470.

[8] B. Chen and A. L. Bertozzi. AutoKG: Efficient Automated Knowledge Graph Generation for Language Models. In 2023 IEEE International Conference on Big Data (BigData), pages 3117–3126, Los Alamitos, CA, USA, dec 2023. IEEE Computer Society. doi: 10.1109/BigData59044.2023.10386454. URL https://doi.ieeecomputersociety.org/10.1109/BigData59044.2023.10386454.

[9] H. Chen, R. Pasunuru, J. Weston, and A. Celikyilmaz. Walking Down the Memory Maze: Beyond Context Limit through Interactive Reading. CoRR, abs/2310.05029, 2023. doi: 10.48550/ARXIV.2310.05029. URL https://doi.org/10.48550/arXiv.2310.05029.

[10] T. Chen, H. Wang, S. Chen, W. Yu, K. Ma, X. Zhao, H. Zhang, and D. Yu. Dense x retrieval: What retrieval granularity should we use? arXiv preprint arXiv:2312.06648, 2023. URL https://arxiv.org/abs/2312.06648.

[11] Y. Chen, S. Qian, H. Tang, X. Lai, Z. Liu, S. Han, and J. Jia. Longlora: Efficient fine-tuning of long-context large language models. arXiv:2309.12307, 2023.

[12] Y. Chen, P. Cao, Y. Chen, K. Liu, and J. Zhao. Journey to the center of the knowledge neurons: Discoveries of language-independent knowledge neurons and degenerate knowledge neurons. Proceedings of the AAAI Conference on Artificial Intelligence, 38(16):17817–17825, Mar. 2024. doi: 10.1609/aaai.v38i16.29735. URL https://ojs.aaai.org/index.php/AAAI/article/view/29735.

[13] G. Csárdi and T. Nepusz. The igraph software package for complex network research. 2006. URL https://igraph.org/.

[14] R. Das, A. Godbole, D. Kavarthapu, Z. Gong, A. Singhal, M. Yu, X. Guo, T. Gao, H. Zamani, M. Zaheer, and A. McCallum. Multi-step entity-centric information retrieval for multi-hop question answering. In A. Fisch, A. Talmor, R. Jia, M. Seo, E. Choi, and D. Chen, editors, Proceedings of the 2nd Workshop on Machine Reading for Question Answering, pages 113–118, Hong Kong, China, Nov. 2019. Association for Computational Linguistics. doi: 10.18653/v1/D19-5816. URL https://aclanthology.org/D19-5816.

[15] N. De Cao, W. Aziz, and I. Titov. Editing factual knowledge in language models. In M.-F. Moens, X. Huang, L. Specia, and S. W.-t. Yih, editors, Proceedings of the 2021 Conference on Empirical Methods in Natural Language Processing, pages 6491–6506, Online and Punta Cana, Dominican Republic, Nov. 2021. Association for Computational Linguistics. doi: 10.18653/v1/2021.emnlp-main.522. URL https://aclanthology.org/2021.emnlp-main.522.

[16] M. Ding, C. Zhou, Q. Chen, H. Yang, and J. Tang. Cognitive graph for multi-hop reading comprehension at scale. In A. Korhonen, D. Traum, and L. Màrquez, editors, Proceedings of the 57th Annual Meeting of the Association for Computational Linguistics, pages 2694–2703, Florence, Italy, July 2019. Association for Computational Linguistics. doi: 10.18653/v1/P19-1259. URL https://aclanthology.org/P19-1259.

[17] Y. Ding, L. L. Zhang, C. Zhang, Y. Xu, N. Shang, J. Xu, F. Yang, and M. Yang. Longrope: Extending llm context window beyond 2 million tokens. ArXiv, abs/2402.13753, 2024. URL https://api.semanticscholar.org/CorpusID:267770308.

[18] D. Edge, H. Trinh, N. Cheng, J. Bradley, A. Chao, A. Mody, S. Truitt, and J. Larson. From local to global: A graph rag approach to query-focused summarization. 2024. URL https://arxiv.org/abs/2404.16130.

[19] H. Eichenbaum. A cortical–hippocampal system for declarative memory. Nature Reviews Neuroscience, 1:41–50, 2000. URL https://www.nature.com/articles/35036213.

[20] Y. Fang, S. Sun, Z. Gan, R. Pillai, S. Wang, and J. Liu. Hierarchical graph network for multi-hop question answering. In B. Webber, T. Cohn, Y. He, and Y. Liu, editors, Proceedings of the 2020 Conference on Empirical Methods in Natural Language Processing (EMNLP), pages 8823–8838, Online, Nov. 2020. Association for Computational Linguistics. doi: 10.18653/v1/2020.emnlp-main.710. URL https://aclanthology.org/2020.emnlp-main.710.

[21] Y. Fu. Challenges in deploying long-context transformers: A theoretical peak performance analysis, 2024. URL https://arxiv.org/abs/2405.08944.

[22] Y. Fu, R. Panda, X. Niu, X. Yue, H. Hajishirzi, Y. Kim, and H. Peng. Data engineering for scaling language models to 128k context, 2024.

[23] M. Geva, J. Bastings, K. Filippova, and A. Globerson. Dissecting recall of factual associations in auto-regressive language models. In H. Bouamor, J. Pino, and K. Bali, editors, Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing, EMNLP 2023, Singapore, December 6-10, 2023, pages 12216–12235. Association for Computational Linguistics, 2023. doi: 10.18653/V1/2023.EMNLP-MAIN.751. URL https://doi.org/10.18653/v1/2023.emnlp-main.751.

[24] C. Gormley and Z. J. Tong. Elasticsearch: The definitive guide. 2015. URL https://www.elastic.co/guide/en/elasticsearch/guide/master/index.html.

[25] T. L. Griffiths, M. Steyvers, and A. J. Firl. Google and the mind. Psychological Science, 18: 1069 – 1076, 2007. URL https://cocosci.princeton.edu/tom/papers/google.pdf.

[26] J.-C. Gu, H.-X. Xu, J.-Y. Ma, P. Lu, Z.-H. Ling, K.-W. Chang, and N. Peng. Model Editing Can Hurt General Abilities of Large Language Models, 2024.

[27] Y. Gu, X. Deng, and Y. Su. Don’t generate, discriminate: A proposal for grounding language models to real-world environments. In A. Rogers, J. Boyd-Graber, and N. Okazaki, editors, Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 4928–4949, Toronto, Canada, July 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.acl-long.270. URL https://aclanthology.org/2023.acl-long.270.

[28] W. Gurnee and M. Tegmark. Language models represent space and time. In The Twelfth International Conference on Learning Representations, 2024. URL https://openreview.net/forum?id=jE8xbmvFin.

[29] J. Han, N. Collier, W. Buntine, and E. Shareghi. PiVe: Prompting with Iterative Verification Improving Graph-based Generative Capability of LLMs, 2023.

[30] T. H. Haveliwala. Topic-sensitive pagerank. In D. Lassner, D. D. Roure, and A. Iyengar, editors, Proceedings of the Eleventh International World Wide Web Conference, WWW 2002, May 7-11, 2002, Honolulu, Hawaii, USA, pages 517–526. ACM, 2002. doi: 10.1145/511446.511513. URL https://dl.acm.org/doi/10.1145/511446.511513.

[31] Q. He, Y. Wang, and W. Wang. Can language models act as knowledge bases at scale?, 2024.

[32] Z. He, L. Karlinsky, D. Kim, J. McAuley, D. Krotov, and R. Feris. CAMELot: Towards large language models with training-free consolidated associative memory. In First Workshop on Long-Context Foundation Models @ ICML 2024, 2024. URL https://openreview.net/forum?id=VLDTzg1a4Y.

[33] X. Ho, A.-K. Duong Nguyen, S. Sugawara, and A. Aizawa. Constructing a multi-hop QA dataset for comprehensive evaluation of reasoning steps. In D. Scott, N. Bel, and C. Zong, editors, Proceedings of the 28th International Conference on Computational Linguistics, pages 6609–6625, Barcelona, Spain (Online), Dec. 2020. International Committee on Computational Linguistics. doi: 10.18653/v1/2020.coling-main.580. URL https://aclanthology.org/2020.coling-main.580.

[34] P.-L. Huguet Cabot and R. Navigli. REBEL: Relation extraction by end-to-end language generation. In M.-F. Moens, X. Huang, L. Specia, and S. W.-t. Yih, editors, Findings of the Association for Computational Linguistics: EMNLP 2021, pages 2370–2381, Punta Cana, Dominican Republic, Nov. 2021. Association for Computational Linguistics. doi: 10.18653/v1/2021.findings-emnlp.204. URL https://aclanthology.org/2021.findings-emnlp.204.

[35] G. Izacard, M. Caron, L. Hosseini, S. Riedel, P. Bojanowski, A. Joulin, and E. Grave. Unsupervised dense information retrieval with contrastive learning, 2021. URL https://arxiv.org/abs/2112.09118.

[36] G. Izacard, P. Lewis, M. Lomeli, L. Hosseini, F. Petroni, T. Schick, J. A. Yu, A. Joulin, S. Riedel, and E. Grave. Few-shot learning with retrieval augmented language models. ArXiv, abs/2208.03299, 2022. URL https://arxiv.org/abs/2208.03299.

[37] J. Jiang, K. Zhou, Z. Dong, K. Ye, X. Zhao, and J.-R. Wen. StructGPT: A general framework for large language model to reason over structured data. In H. Bouamor, J. Pino, and K. Bali, editors, Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing, pages 9237–9251, Singapore, Dec. 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.emnlp-main.574. URL https://aclanthology.org/2023.emnlp-main.574.

[38] Z. Jiang, F. Xu, L. Gao, Z. Sun, Q. Liu, J. Dwivedi-Yu, Y. Yang, J. Callan, and G. Neubig. Active retrieval augmented generation. In H. Bouamor, J. Pino, and K. Bali, editors, Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing, pages 7969–7992, Singapore, Dec. 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.emnlp-main.495. URL https://aclanthology.org/2023.emnlp-main.495.

[39] S. Kambhampati. Can large language models reason and plan? Annals of the New York Academy of Sciences, 2024. URL https://nyaspubs.onlinelibrary.wiley.com/doi/abs/10.1111/nyas.15125.

[40] W. Kwon, Z. Li, S. Zhuang, Y. Sheng, L. Zheng, C. H. Yu, J. E. Gonzalez, H. Zhang, and I. Stoica. Efficient memory management for large language model serving with pagedattention. In Proceedings of the ACM SIGOPS 29th Symposium on Operating Systems Principles, 2023.

[41] M. Levy, A. Jacoby, and Y. Goldberg. Same task, more tokens: the impact of input length on the reasoning performance of large language models, 2024.

[42] P. Lewis, E. Perez, A. Piktus, F. Petroni, V. Karpukhin, N. Goyal, H. Küttler, M. Lewis, W.-t. Yih, T. Rocktäschel, S. Riedel, and D. Kiela. Retrieval-augmented generation for knowledge-intensive NLP tasks. In Proceedings of the 34th International Conference on Neural Information Processing Systems, NIPS ’20, Red Hook, NY, USA, 2020. Curran Associates Inc. ISBN 9781713829546. URL https://dl.acm.org/doi/abs/10.5555/3495724.3496517.

[43] R. Li and X. Du. Leveraging structured information for explainable multi-hop question answering and reasoning. In H. Bouamor, J. Pino, and K. Bali, editors, Findings of the Association for Computational Linguistics: EMNLP 2023, pages 6779–6789, Singapore, Dec. 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.findings-emnlp.452. URL https://aclanthology.org/2023.findings-emnlp.452.

[44] S. Li, X. Li, L. Shang, X. Jiang, Q. Liu, C. Sun, Z. Ji, and B. Liu. Hopretriever: Retrieve hops over wikipedia to answer complex questions. Proceedings of the AAAI Conference on Artificial Intelligence, 35:13279–13287, 05 2021. doi: 10.1609/aaai.v35i15.17568.

[45] T. Li, G. Zhang, Q. D. Do, X. Yue, and W. Chen. Long-context LLMs Struggle with Long In-context Learning, 2024.

[46] Z. Li, N. Zhang, Y. Yao, M. Wang, X. Chen, and H. Chen. Unveiling the pitfalls of knowledge editing for large language models. In The Twelfth International Conference on Learning Representations, 2024. URL https://openreview.net/forum?id=fNktD3ib16.

[47] Y. Liu, X. Peng, T. Du, J. Yin, W. Liu, and X. Zhang. ERA-CoT: Improving chain-of-thought through entity relationship analysis. In L.-W. Ku, A. Martins, and V. Srikumar, editors, Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 8780–8794, Bangkok, Thailand, Aug. 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.476. URL https://aclanthology.org/2024.acl-long.476.

[48] L. LUO, Y.-F. Li, R. Haf, and S. Pan. Reasoning on graphs: Faithful and interpretable large language model reasoning. In The Twelfth International Conference on Learning Representations, 2024. URL https://openreview.net/forum?id=ZGNWW7xZ6Q.

[49] K. Meng, D. Bau, A. Andonian, and Y. Belinkov. Locating and editing factual associations in gpt. In Neural Information Processing Systems, 2022.

[50] E. Mitchell, C. Lin, A. Bosselut, C. Finn, and C. D. Manning. Fast model editing at scale. ArXiv, abs/2110.11309, 2021.

[51] E. Mitchell, C. Lin, A. Bosselut, C. D. Manning, and C. Finn. Memory-based model editing at scale. ArXiv, abs/2206.06520, 2022.

[52] T. T. Nguyen, T. T. Huynh, P. L. Nguyen, A. W.-C. Liew, H. Yin, and Q. V. H. Nguyen. A survey of machine unlearning. arXiv preprint arXiv:2209.02299, 2022.

[53] J. Ni, C. Qu, J. Lu, Z. Dai, G. Hernandez Abrego, J. Ma, V. Zhao, Y. Luan, K. Hall, M.-W. Chang, and Y. Yang. Large dual encoders are generalizable retrievers. In Y. Goldberg, Z. Kozareva, and Y. Zhang, editors, Proceedings of the 2022 Conference on Empirical Methods in Natural Language Processing, pages 9844–9855, Abu Dhabi, United Arab Emirates, Dec. 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.emnlp-main.669. URL https://aclanthology.org/2022.emnlp-main.669.

[54] Y. Nie, S. Wang, and M. Bansal. Revealing the importance of semantic retrieval for machine reading at scale. In K. Inui, J. Jiang, V. Ng, and X. Wan, editors, Proceedings of the 2019 Conference on Empirical Methods in Natural Language Processing and the 9th International Joint Conference on Natural Language Processing (EMNLP-IJCNLP), pages 2553–2566, Hong Kong, China, Nov. 2019. Association for Computational Linguistics. doi: 10.18653/v1/D19-1258. URL https://aclanthology.org/D19-1258.

[55] OpenAI. GPT-3.5 Turbo, 2024. URL https://platform.openai.com/docs/models/gpt-3-5-turbo.

[56] S. Pan, L. Luo, Y. Wang, C. Chen, J. Wang, and X. Wu. Unifying large language models and knowledge graphs: A roadmap. IEEE Transactions on Knowledge and Data Engineering, pages 1–20, 2024. doi: 10.1109/TKDE.2024.3352100.

[57] J. Park, A. Patel, O. Z. Khan, H. J. Kim, and J.-K. Kim. Graph elicitation for guiding multi-step reasoning in large language models, 2024. URL https://arxiv.org/abs/2311.09762.

[58] S. Park and J. Bak. Memoria: Resolving fateful forgetting problem through human-inspired memory architecture. In ICML, 2024. URL https://openreview.net/forum?id=yTz0u4B8ug.

[59] A. Paszke, S. Gross, F. Massa, A. Lerer, J. Bradbury, G. Chanan, T. Killeen, Z. Lin, N. Gimelshein, L. Antiga, A. Desmaison, A. Köpf, E. Z. Yang, Z. DeVito, M. Raison, A. Tejani, S. Chilamkurthy, B. Steiner, L. Fang, J. Bai, and S. Chintala. Pytorch: An imperative style, high-performance deep learning library. In H. M. Wallach, H. Larochelle, A. Beygelzimer, F. d’Alché-Buc, E. B. Fox, and R. Garnett, editors, Advances in Neural Information Processing Systems 32: Annual Conference on Neural Information Processing Systems 2019, NeurIPS 2019, December 8-14, 2019, Vancouver, BC, Canada, pages 8024–8035, 2019. URL https://dl.acm.org/doi/10.5555/3454287.3455008.

[60] K. Pei, I. Jindal, K. C.-C. Chang, C. Zhai, and Y. Li. When to use what: An in-depth comparative empirical analysis of OpenIE systems for downstream applications. In A. Rogers, J. Boyd-Graber, and N. Okazaki, editors, Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 929–949, Toronto, Canada, July 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.acl-long.53. URL https://aclanthology.org/2023.acl-long.53.

[61] B. Peng, J. Quesnelle, H. Fan, and E. Shippole. Yarn: Efficient context window extension of large language models, 2023.

[62] F. Petroni, T. Rocktäschel, S. Riedel, P. Lewis, A. Bakhtin, Y. Wu, and A. Miller. Language models as knowledge bases? In K. Inui, J. Jiang, V. Ng, and X. Wan, editors, Proceedings of the 2019 Conference on Empirical Methods in Natural Language Processing and the 9th International Joint Conference on Natural Language Processing (EMNLP-IJCNLP), pages 2463–2473, Hong Kong, China, Nov. 2019. Association for Computational Linguistics. doi: 10.18653/v1/D19-1250. URL https://aclanthology.org/D19-1250.

[63] O. Press, M. Zhang, S. Min, L. Schmidt, N. Smith, and M. Lewis. Measuring and narrowing the compositionality gap in language models. In H. Bouamor, J. Pino, and K. Bali, editors, Findings of the Association for Computational Linguistics: EMNLP 2023, pages 5687–5711, Singapore, Dec. 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.findings-emnlp.378. URL https://aclanthology.org/2023.findings-emnlp.378.

[64] O. Press, M. Zhang, S. Min, L. Schmidt, N. A. Smith, and M. Lewis. Measuring and narrowing the compositionality gap in language models, 2023. URL https://openreview.net/forum?id=PUwbwZJz9dO.

[65] L. Qiu, Y. Xiao, Y. Qu, H. Zhou, L. Li, W. Zhang, and Y. Yu. Dynamically fused graph network for multi-hop reasoning. In A. Korhonen, D. Traum, and L. Màrquez, editors, Proceedings of the 57th Annual Meeting of the Association for Computational Linguistics, pages 6140–6150, Florence, Italy, July 2019. Association for Computational Linguistics. doi: 10.18653/v1/P19-1617. URL https://aclanthology.org/P19-1617.

[66] O. Ram, Y. Levine, I. Dalmedigos, D. Muhlgay, A. Shashua, K. Leyton-Brown, and Y. Shoham. In-context retrieval-augmented language models. Transactions of the Association for Computational Linguistics, 11:1316–1331, 2023. doi: 10.1162/tacl_a_00605. URL https://aclanthology.org/2023.tacl-1.75.

[67] G. Ramesh, M. N. Sreedhar, and J. Hu. Single sequence prediction over reasoning graphs for multi-hop QA. In A. Rogers, J. Boyd-Graber, and N. Okazaki, editors, Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 11466–11481, Toronto, Canada, July 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.acl-long.642. URL https://aclanthology.org/2023.acl-long.642.

[68] M. Reid, N. Savinov, D. Teplyashin, D. Lepikhin, T. Lillicrap, J.-b. Alayrac, R. Soricut, A. Lazaridou, O. Firat, J. Schrittwieser, et al. Gemini 1.5: Unlocking multimodal understanding across millions of tokens of context. arXiv preprint arXiv:2403.05530, 2024. URL https://arxiv.org/abs/2403.05530.

[69] S. E. Robertson and S. Walker. Some simple effective approximations to the 2-poisson model for probabilistic weighted retrieval. In W. B. Croft and C. J. van Rijsbergen, editors, Proceedings of the 17th Annual International ACM-SIGIR Conference on Research and Development in Information Retrieval. Dublin, Ireland, 3-6 July 1994 (Special Issue of the SIGIR Forum), pages 232–241. ACM/Springer, 1994. doi: 10.1007/978-1-4471-2099-5\_24. URL https://link.springer.com/chapter/10.1007/978-1-4471-2099-5_24.

[70] K. Santhanam, O. Khattab, J. Saad-Falcon, C. Potts, and M. Zaharia. ColBERTv2: Effective and efficient retrieval via lightweight late interaction. In M. Carpuat, M.-C. de Marneffe, and I. V. Meza Ruiz, editors, Proceedings of the 2022 Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies, pages 3715–3734, Seattle, United States, July 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.naacl-main.272. URL https://aclanthology.org/2022.naacl-main.272.

[71] P. Sarthi, S. Abdullah, A. Tuli, S. Khanna, A. Goldie, and C. D. Manning. RAPTOR: recursive abstractive processing for tree-organized retrieval. CoRR, abs/2401.18059, 2024. doi: 10.48550/ARXIV.2401.18059. URL https://arxiv.org/abs/2401.18059.

[72] Z. Shao, Y. Gong, Y. Shen, M. Huang, N. Duan, and W. Chen. Enhancing retrieval-augmented large language models with iterative retrieval-generation synergy. In H. Bouamor, J. Pino, and K. Bali, editors, Findings of the Association for Computational Linguistics: EMNLP 2023, pages 9248–9274, Singapore, Dec. 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.findings-emnlp.620. URL https://aclanthology.org/2023.findings-emnlp.620.

[73] W. Shi, S. Min, M. Yasunaga, M. Seo, R. James, M. Lewis, L. Zettlemoyer, and W. tau Yih. Replug: Retrieval-augmented black-box language models. ArXiv, abs/2301.12652, 2023. URL https://api.semanticscholar.org/CorpusID:256389797.

[74] J. Sun, C. Xu, L. Tang, S. Wang, C. Lin, Y. Gong, L. Ni, H.-Y. Shum, and J. Guo. Think-on-graph: Deep and responsible reasoning of large language model on knowledge graph. In The Twelfth International Conference on Learning Representations, 2024. URL https://openreview.net/forum?id=nnVO1PvbTv.

[75] T. J. Teyler and P. Discenna. The hippocampal memory indexing theory. Behavioral neuroscience, 100 2:147–54, 1986. URL https://pubmed.ncbi.nlm.nih.gov/3008780/.

[76] T. J. Teyler and J. W. Rudy. The hippocampal indexing theory and episodic memory: Updating the index. Hippocampus, 17, 2007. URL https://pubmed.ncbi.nlm.nih.gov/17696170/.

[77] H. Trivedi, N. Balasubramanian, T. Khot, and A. Sabharwal. MuSiQue: Multihop questions via single-hop question composition. Trans. Assoc. Comput. Linguistics, 10:539–554, 2022. doi: 10.1162/TACL\_A\_00475. URL https://aclanthology.org/2022.tacl-1.31/.

[78] H. Trivedi, N. Balasubramanian, T. Khot, and A. Sabharwal. Interleaving retrieval with chain-of-thought reasoning for knowledge-intensive multi-step questions. In A. Rogers, J. Boyd-Graber, and N. Okazaki, editors, Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 10014–10037, Toronto, Canada, July 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.acl-long.557. URL https://aclanthology.org/2023.acl-long.557.

[79] C. Wang, X. Liu, Y. Yue, X. Tang, T. Zhang, C. Jiayang, Y. Yao, W. Gao, X. Hu, Z. Qi, Y. Wang, L. Yang, J. Wang, X. Xie, Z. Zhang, and Y. Zhang. Survey on factuality in large language models: Knowledge, retrieval and domain-specificity, 2023.

[80] J. Wang, Q. Sun, N. Chen, X. Li, and M. Gao. Boosting language models reasoning with chain-of-knowledge prompting, 2023.

[81] X. Wang, J. Wei, D. Schuurmans, Q. V. Le, E. H. Chi, S. Narang, A. Chowdhery, and D. Zhou. Self-consistency improves chain of thought reasoning in language models. In The Eleventh International Conference on Learning Representations, 2023. URL https://openreview.net/forum?id=1PL1NIMMrw.

[82] Y. Wang, Y. Gao, X. Chen, H. Jiang, S. Li, J. Yang, Q. Yin, Z. Li, X. Li, B. Yin, J. Shang, and J. Mcauley. MEMORYLLM: Towards self-updatable large language models. In R. Salakhutdinov, Z. Kolter, K. Heller, A. Weller, N. Oliver, J. Scarlett, and F. Berkenkamp, editors, Proceedings of the 41st International Conference on Machine Learning, volume 235 of Proceedings of Machine Learning Research, pages 50453–50466. PMLR, 21–27 Jul 2024. URL https://proceedings.mlr.press/v235/wang24s.html.

[83] J. Wei, X. Wang, D. Schuurmans, M. Bosma, brian ichter, F. Xia, E. H. Chi, Q. V. Le, and D. Zhou. Chain of thought prompting elicits reasoning in large language models. In A. H. Oh, A. Agarwal, D. Belgrave, and K. Cho, editors, Advances in Neural Information Processing Systems, 2022. URL https://openreview.net/forum?id=_VjQlMeSB_J.

[84] Y. Wen, Z. Wang, and J. Sun. Mindmap: Knowledge graph prompting sparks graph of thoughts in large language models. arXiv preprint arXiv:2308.09729, 2023.

[85] P. West, C. Bhagavatula, J. Hessel, J. Hwang, L. Jiang, R. Le Bras, X. Lu, S. Welleck, and Y. Choi. Symbolic knowledge distillation: from general language models to commonsense models. In M. Carpuat, M.-C. de Marneffe, and I. V. Meza Ruiz, editors, Proceedings of the 2022 Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies, pages 4602–4625, Seattle, United States, July 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.naacl-main.341. URL https://aclanthology.org/2022.naacl-main.341.

[86] T. Wolf, L. Debut, V. Sanh, J. Chaumond, C. Delangue, A. Moi, P. Cistac, T. Rault, R. Louf, M. Funtowicz, J. Davison, S. Shleifer, P. von Platen, C. Ma, Y. Jernite, J. Plu, C. Xu, T. L. Scao, S. Gugger, M. Drame, Q. Lhoest, and A. M. Rush. Huggingface’s transformers: State-of-the-art natural language processing. ArXiv, abs/1910.03771, 2019. URL https://arxiv.org/abs/1910.03771.

[87] J. Xie, K. Zhang, J. Chen, R. Lou, and Y. Su. Adaptive chameleon or stubborn sloth: Revealing the behavior of large language models in knowledge conflicts. In The Twelfth International Conference on Learning Representations, 2024. URL https://openreview.net/forum?id=auKAUJZMO6.

[88] W. Xiong, X. Li, S. Iyer, J. Du, P. Lewis, W. Y. Wang, Y. Mehdad, S. Yih, S. Riedel, D. Kiela, and B. Oguz. Answering complex open-domain questions with multi-hop dense retrieval. In International Conference on Learning Representations, 2021. URL https://openreview.net/forum?id=EMHoBG0avc1.

[89] Z. Yang, P. Qi, S. Zhang, Y. Bengio, W. W. Cohen, R. Salakhutdinov, and C. D. Manning. HotpotQA: A dataset for diverse, explainable multi-hop question answering. In E. Riloff, D. Chiang, J. Hockenmaier, and J. Tsujii, editors, Proceedings of the 2018 Conference on Empirical Methods in Natural Language Processing, Brussels, Belgium, October 31 -November 4, 2018, pages 2369–2380. Association for Computational Linguistics, 2018. doi: 10.18653/V1/D18-1259. URL https://aclanthology.org/D18-1259/.

[90] S. Yao, J. Zhao, D. Yu, N. Du, I. Shafran, K. Narasimhan, and Y. Cao. ReAct: Synergizing reasoning and acting in language models. In International Conference on Learning Representations (ICLR), 2023.

[91] M. Yasunaga, A. Bosselut, H. Ren, X. Zhang, C. D. Manning, P. Liang, and J. Leskovec. Deep bidirectional language-knowledge graph pretraining. In Neural Information Processing Systems (NeurIPS), 2022. URL https://arxiv.org/abs/2210.09338.

[92] O. Yoran, T. Wolfson, B. Bogin, U. Katz, D. Deutch, and J. Berant. Answering questions by meta-reasoning over multiple chains of thought. In The 2023 Conference on Empirical Methods in Natural Language Processing, 2023. URL https://openreview.net/forum?id=ebSOK1nV2r.

[93] W. Yu, D. Iter, S. Wang, Y. Xu, M. Ju, S. Sanyal, C. Zhu, M. Zeng, and M. Jiang. Generate rather than retrieve: Large language models are strong context generators. In The Eleventh International Conference on Learning Representations, 2023. URL https://openreview.net/forum?id=fB0hRu9GZUS.

[94] K. Zhang, B. Jimenez Gutierrez, and Y. Su. Aligning instruction tasks unlocks large language models as zero-shot relation extractors. In A. Rogers, J. Boyd-Graber, and N. Okazaki, editors, Findings of the Association for Computational Linguistics: ACL 2023, pages 794–812, Toronto, Canada, July 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.findings-acl.50. URL https://aclanthology.org/2023.findings-acl.50.

[95] N. Zhang, Y. Yao, B. Tian, P. Wang, S. Deng, M. Wang, Z. Xi, S. Mao, J. Zhang, Y. Ni, et al. A comprehensive study of knowledge editing for large language models. arXiv preprint arXiv:2401.01286, 2024.

[96] X. Zhang, Y. Chen, S. Hu, Z. Xu, J. Chen, M. K. Hao, X. Han, Z. L. Thai, S. Wang, Z. Liu, and M. Sun. ∞bench: Extending long context evaluation beyond 100k tokens, 2024.

[97] Z. Zhong, Z. Wu, C. D. Manning, C. Potts, and D. Chen. Mquake: Assessing knowledge editing in language models via multi-hop questions. In Conference on Empirical Methods in Natural Language Processing, 2023. URL https://aclanthology.org/2023.emnlp-main.971.pdf.

[98] S. Zhou, B. Yu, A. Sun, C. Long, J. Li, and J. Sun. A survey on neural open information extraction: Current status and future directions. In L. D. Raedt, editor, Proceedings of the Thirty-First International Joint Conference on Artificial Intelligence, IJCAI-22, pages 5694–5701. International Joint Conferences on Artificial Intelligence Organization, 7 2022. doi: 10.24963/ijcai.2022/793. URL https://doi.org/10.24963/ijcai.2022/793. Survey Track.

[99] H. Zhu, H. Peng, Z. Lyu, L. Hou, J. Li, and J. Xiao. Pre-training language model incorporating domain-specific heterogeneous knowledge into a unified representation. Expert Systems with Applications, 215:119369, 2023. ISSN 0957-4174. doi: https://doi.org/10.1016/j.eswa.2022.119369. URL https://www.sciencedirect.com/science/article/pii/S0957417422023879.

[100] Y. Zhu, L. Pang, Y. Lan, H. Shen, and X. Cheng. Adaptive information seeking for open-domain question answering. In M.-F. Moens, X. Huang, L. Specia, and S. W.-t. Yih, editors, Proceedings of the 2021 Conference on Empirical Methods in Natural Language Processing, pages 3615–3626, Online and Punta Cana, Dominican Republic, Nov. 2021. Association for Computational Linguistics. doi: 10.18653/v1/2021.emnlp-main.293. URL https://aclanthology.org/2021.emnlp-main.293.

# 附录

补充材料进一步说明以下内容：

- 附录 A：HippoRAG 管线示例
- 附录 B：数据集比较
- 附录 C：消融实验统计
- 附录 D：OpenIE 内在评估
- 附录 E：路径查找型多跳案例研究
- 附录 F：错误分析
- 附录 G：成本与效率比较
- 附录 H：实现细节与计算需求
- 附录 I：LLM 提示词

## A HippoRAG 管线示例

为更清楚地展示 HippoRAG 管线的运行方式，我们采用表 7 中来自 MuSiQue 数据集的路径跟随型示例。我们用 HippoRAG 的索引与检索过程处理该问题及相关语料库的一个子集。问题、答案、支持段落与干扰段落如图 3 所示。图 4 展示索引阶段，其中既包括 OpenIE 流程，也包括 KG 的相关子图。最后，图 5 展示检索阶段，包括查询 NER、查询节点检索、PPR 算法如何改变节点概率，以及如何计算排名最高的检索结果。

> **图 3 内文字：问题与答案**  
> **问题：** Alhandra 出生在哪个区？  
> **答案：** Lisbon
>
> **支持段落**  
> **1. Alhandra（足球运动员）**  
> Luís Miguel Assunção Joaquim（1979 年 3 月 5 日出生于 Lisbon 的 Vila Franca de Xira），通称 Alhandra，是一名已退役的葡萄牙足球运动员，主要司职左后卫，也可出任中场。  
> **2. Vila Franca de Xira**  
> Vila Franca de Xira 是葡萄牙 Lisbon District 的一个市镇。2011 年人口为 136,886，面积为 $318.19\ \mathrm{km}^2$。它坐落于 Tagus River 两岸，位于葡萄牙首都 Lisbon 东北 32 千米处。Pedra Furada Cave 中的发现表明，该地区的定居历史可追溯至新石器时代。据称，约在 1200 年，葡萄牙首位国王 Afonso Henriques 的法国追随者建立了 Vila Franca de Xira。  
>
> **干扰段落（节选）**  
> **1. Chirakkalkulam**  
> Chirakkalkulam 是南印度 Kerala 州 Kannur District 的 Kannur 镇附近一片小型居住区，位于 Thayatheru 与 Kannur City 之间。Chirakkalkulam 的重要性源于历史上的 Arakkal Kingdom 诞生于此。  
> **2. Frank T. and Polly Lewis House**  
> Frank T. and Polly Lewis House 位于美国 Wisconsin 州 Lodi，2009 年被列入 National Register of Historic Places。该住宅位于 Portage Street Historic District 内。  
> **3. 出生证明**  
> 在美国，出生证明由各州、首都地区、领地和前领地的 Vital Records Office 签发……

**图 3：HippoRAG 管线示例（问题与标注）。** 上部给出一个示例问题及其答案；中部与下部给出该问题的支持段落和干扰段落。解答该问题需要两篇支持段落。干扰段落的节选均与问题中的“区”有关。

> **图 4 内文字：索引——支持段落的段落 NER 与 OpenIE**  
> **1. Alhandra（足球运动员）**  
> NER：`["5 March 1979", "Alhandra", "Lisbon", "Luís Miguel Assunção Joaquim", "Portuguese", "Vila Franca de Xira"]`  
> OpenIE：  
> `[("Alhandra", "是一名", "足球运动员"),`  
> `("Alhandra", "出生于", "Vila Franca de Xira"),`  
> `("Alhandra", "出生于", "Lisbon"),`  
> `("Alhandra", "出生日期为", "5 March 1979"),`  
> `("Alhandra", "是", "Portuguese"),`  
> `("Luís Miguel Assunção Joaquim", "也被称为", "Alhandra")]`  
> **2. Vila Franca de Xira**  
> NER：`["2011", "Afonso Henriques", "Cave of Pedra Furada", "French", "Lisbon", "Lisbon District", "Portugal", "Tagus River", "Vila Franca de Xira"]`  
> OpenIE：  
> `[("Vila Franca de Xira", "是一个位于……的市镇", "Lisbon District"),`  
> `("Vila Franca de Xira", "位于", "Portugal"),`  
> `("Vila Franca de Xira", "坐落于", "Tagus River"),`  
> `("Vila Franca de Xira", "由……建立", "Afonso Henriques 的法国追随者"),`  
> `("Tagus River", "位于……附近", "Lisbon"),`  
> `("Cave of Pedra Furada", "证明……存在定居活动", "新石器时代"),`  
> `("Afonso Henriques", "是葡萄牙首位国王，时间为", "1200"),`  
> `("Vila Franca de Xira", "人口为", "2011 年的 136,886"),`  
> `("Vila Franca de Xira", "面积为", "318.19 km²")]`
>
> **与问题相关的索引子图：** `Alhandra` 通过“出生于”连接 `Lisbon` 与 `Vila Franca de Xira`，通过“出生日期为”连接 `5 March 1979`，通过“是一名”连接 `footballer`，通过“是”连接 `Portuguese`，并通过“也被称为”连接 `Luís Miguel Assunção Joaquim`。`Vila Franca de Xira` 与 `vila`、`municipality of Vila Franca de Xira` 存在等价关系；它通过“位于”连接 `Portugal`，通过“是一个位于……的市镇”连接 `Lisbon District`，通过“坐落于”连接 `Tagus River`，并连接其创建者、人口与面积等节点。

**图 4：HippoRAG 管线示例（索引）。** 系统依次对语料库中的每个段落执行 NER 与 OpenIE，从而为整个语料库形成一张开放知识图谱。图中仅展示 KG 的相关子图。

> **图 5 内文字：检索——查询 NER 与节点检索**  
> **问题：** Alhandra 出生在哪个区？  
> **NER：** `["Alhandra"]`  
> **节点检索：** `{"Alhandra": "Alhandra"}`  
>
> **检索——PPR 导致的节点概率变化：**  
> `Alhandra`：$1.000\Rightarrow0.533$；`Vila Franca de Xira`：$0.000\Rightarrow0.054$；`Lisbon`：$0.000\Rightarrow0.049$；`footballer`：$0.000\Rightarrow0.047$；`Portuguese`：$0.000\Rightarrow0.046$；`5 March 1979`：$0.000\Rightarrow0.045$；`Luís Miguel Assunção Joaquim`：$0.000\Rightarrow0.044$；`Portugal`：$0.000\Rightarrow0.009$；`Tagus River`：$0.000\Rightarrow0.007$；`José Pinto Coelho`：$0.000\Rightarrow0.004$；……  
>
> **检索——排名最高的结果**（原图以突出显示标出 PPR 排名最高的节点）：  
> **1. Alhandra（足球运动员）** Luís Miguel Assunção Joaquim（1979 年 3 月 5 日出生于 Lisbon 的 Vila Franca de Xira），通称 Alhandra，是一名已退役的葡萄牙足球运动员，主要司职左后卫，也可出任中场。  
> **2. Vila Franca de Xira** Vila Franca de Xira 是葡萄牙 Lisbon District 的一个市镇。2011 年人口为 136,886，面积为 $318.19\ \mathrm{km}^2$。它坐落于 Tagus River 两岸，位于葡萄牙首都 Lisbon 东北 32 千米处。Pedra Furada Cave 中的发现表明，该地区的定居历史可追溯至新石器时代。据称，约在 1200 年，葡萄牙首位国王 Afonso Henriques 的法国追随者建立了 Vila Franca de Xira。  
> **3. Portugal** Portuguese 是 Portugal 的官方语言。Portuguese 是一种 Romance language，起源于今天的 Galicia 与 Northern Portugal，并源自 Galician-Portuguese；后者在 Portugal 独立前一直是 Galician 与 Portuguese 人民的共同语言。尤其在 Portugal 北部，Galician 文化与 Portuguese 文化至今仍有许多相似之处。Galicia 是 Community of Portuguese Language Countries 的咨询观察员。根据 *Ethnologue of Languages*，Portuguese 与 Spanish 的词汇相似度为 89%，两种语言中受过教育的使用者能够轻松交流。  
> **4. Huguenots** 最早离开 France 的 Huguenots 前往 Switzerland 与 Netherlands 寻求免遭迫害的自由……人们修建了一座名为 Fort Coligny 的堡垒，以保护他们免受 Portuguese 军队和 Brazilian Native Americans 的袭击。这是一次在 South America 建立法国殖民地的尝试。1560 年，Portuguese 摧毁堡垒并俘获部分 Huguenots。Portuguese 威胁这些囚犯：若不改信 Catholicism，便处死他们……  
> **5. East Timor** Democratic Republic of Timor-Leste；Repúblika Demokrátika Timór Lorosa'e（Tetum）；República Democrática de Timor-Leste（Portuguese）；国旗；国徽；格言：Unidade, Acção, Progresso（Portuguese），Unidade, Asaun, Progresu（Tetum），英文意为“Unity, Action, Progress”；国歌：Pátria（Portuguese），英文意为“Fatherland”；首都及最大城市：Dili；坐标：$8^\circ20'\mathrm{S},125^\circ20'\mathrm{E}$……

**图 5：HippoRAG 管线示例（检索）。** 检索时，系统先从问题中抽取查询命名实体（上部），再使用检索编码器选择查询节点。在本例中，查询命名实体 `Alhandra` 的名称与对应 KG 节点相同。随后（中部），系统根据检索到的查询节点设置 PPR 个性化概率。运行 PPR 后，查询节点概率会按照图 4 的子图进行分配，使 `Vila Franca de Xira` 节点获得部分概率质量。最后（下部），系统在这些节点出现的各段落上对节点概率求和，得到段落级排序。原图在排名最高的段落中突出显示了 PPR 排名最高的节点。

## B 数据集比较

为分析所用三个数据集之间的差异，我们特别关注干扰段落的质量，也就是它们能否有效地与支持段落混淆。我们使用 Contriever [35] 计算问题与候选段落之间的匹配分数，并在图 6 中展示其密度分布。理想情况下，干扰段落的分数分布应接近支持段落分数的均值。然而可以看到，与另外两个数据集相比，HotpotQA 的干扰段落分数分布更接近支持段落分数的下界。

> **图 6 内文字：** 三个面板依次为 MuSiQue、2WikiMultihopQA 和 HotpotQA；横轴为分数（Scores），纵轴为密度（Density）；三条分布分别表示干扰段落（Distractors）、相似度最高的支持文档（Max supporting document）和相似度最低的支持文档（Min supporting document）。

**图 6：候选段落的相似度分数密度。** 分数由 Contriever 对干扰段落与支持段落计算得到。HotpotQA 干扰段落的相似度分数并未显著高于相似度最低的支持段落，说明这些干扰项并不十分有效。

## C 消融实验统计

OpenIE 消融实验使用 GPT-3.5 Turbo、REBEL [34] 和 Llama-3.1（8B 与 70B）[1]。如表 8 所示，与 GPT-3.5 Turbo 及两个 Llama 模型相比，REBEL 生成的节点数和边数约少一半。这说明，与开放权重及闭源 LLM 相比，REBEL 在开放信息抽取上缺乏灵活性。与此同时，两个 Llama-3.1 版本生成的 OpenIE 三元组数量与 GPT-3.5 Turbo 相近。

**表 8：使用不同 OpenIE 方法得到的知识图谱统计。**

| 模型 | 统计项 | MuSiQue | 2Wiki | HotpotQA |
|---|---|---:|---:|---:|
| GPT-3.5 Turbo (1106) [55]（默认） | 唯一节点数（$N$） | 91,729 | 42,694 | 82,157 |
|  | 唯一边数（$E$） | 21,714 | 7,867 | 17,523 |
|  | 唯一三元组数 | 107,448 | 50,671 | 98,709 |
|  | ColBERTv2 同义边数（$E'$） | 191,636 | 82,526 | 171,856 |
| REBEL-large [34] | 唯一节点数（$N$） | 36,653 | 22,146 | 30,426 |
|  | 唯一边数（$E$） | 269 | 211 | 262 |
|  | 唯一三元组数 | 52,102 | 30,428 | 42,038 |
|  | ColBERTv2 同义边数（$E'$） | 48,213 | 33,072 | 39,053 |
| Llama-3.1-8B-Instruct [1] | 唯一节点数（$N$） | 86,864 | 37,875 | 76,311 |
|  | 唯一边数（$E$） | 22,807 | 6,729 | 18,109 |
|  | 唯一三元组数 | 118,430 | 47,420 | 104,981 |
|  | ColBERTv2 同义边数（$E'$） | 155,889 | 72,963 | 139,181 |
| Llama-3.1-70B-Instruct [1] | 唯一节点数（$N$） | 80,634 | 39,845 | 70,304 |
|  | 唯一边数（$E$） | 22,120 | 6,996 | 16,404 |
|  | 唯一三元组数 | 120,514 | 55,940 | 105,281 |
|  | ColBERTv2 同义边数（$E'$） | 140,328 | 69,125 | 119,948 |

## D OpenIE 内在评估

为更好地理解 OpenIE 与检索如何相互作用，我们从 MuSiQue 训练数据集的 20 篇文档中抽取了金标准三元组，共计 239 个。根据表 9 的结果，我们首先注意到，REBEL 等端到端信息抽取系统与 LLM 之间存在巨大差距。此外，OpenIE 质量与检索性能之间存在一定相关性：Llama-3.1-Instruct 8B 版本在检索指标和内在指标上均逊于 70B 版本。更具体地说，较大模型只在内在评估的召回率指标上获得提升，而召回率似乎对改善检索性能尤为重要。最后，该评估与检索性能并非完全相关：GPT-3.5 的内在性能远强于 Llama-3.1-70B-Instruct，但其检索分数只略高一些。

**表 9：在 20 篇标注段落上使用 CaRB [6] 框架进行的 OpenIE 内在评估。**

| 模型 | AUC | 精确率 | 召回率 | F1 |
|---|---:|---:|---:|---:|
| GPT-3.5 Turbo (1106) [55]（默认） | 46.5 | 68.4 | 55.2 | 61.1 |
| Llama-3.1-8B-Instruct [1] | 40.0 | 66.4 | 48.1 | 55.8 |
| Llama-3.1-70B-Instruct [1] | 42.3 | 66.3 | 50.9 | 57.6 |
| REBEL [34] | 1.0 | 8.0 | 1.8 | 2.9 |

## E 路径查找型多跳 QA 案例研究

如前文所述，跨段落的路径查找型多跳问题对 ColBERTv2 和 IRCoT 等单步及多步 RAG 方法而言极具挑战。这些问题要求跨多篇段落整合信息，从众多可能候选项中找出相关实体，例如从所有 Stanford 教授中找出从事 Alzheimer’s 神经科学研究的人。

### E.1 路径查找型多跳问题的构建过程

这些问题及围绕它们整理的语料库通过以下流程构建。前两个问题的构建过程与第三个问题以及正文中的动机示例略有不同。对于前两个问题，我们首先确定一本书或一部电影，然后找到该书的作者或电影的导演。接着，分别寻找：1）书籍或电影的一项特征；2）作者或导演的另一项特征。随后，利用这两个特征从 Wikipedia 中为每个问题抽取干扰项。

对于第三个问题和正文中的动机示例，我们首先随机选择一位教授或一种药物作为各问题的答案。随后，获取该教授任职的大学或该药物治疗的疾病，并获取该教授或药物的另一项特征；本文选择的特征分别是研究主题和作用机制。在这些问题中，一方面根据大学或疾病，另一方面根据研究主题或作用机制，从 Wikipedia 抽取干扰项。虽然整个过程相当繁琐，却使我们得以整理出这些具有挑战性而又符合现实的路径查找型多跳问题。

### E.2 定性分析

表 10 给出来自三个不同领域的更多示例，用于说明 HippoRAG 在解决需要此类跨段落知识整合的检索任务时所具备的潜力。

在表 10 的第一个问题中，我们要寻找一本 2012 年出版、作者是一位曾获特定奖项的 English 作家的书。与 HippoRAG 不同，ColBERTv2 和 IRCoT 都无法识别 Mark Haddon 符合这一条件。ColBERTv2 聚焦于与奖项有关的段落；IRCoT 则误判 Kate Atkinson 是答案，因为她曾凭借一本 1995 年出版的书赢得同一奖项。

第二个问题要求寻找一部改编自非虚构书籍、且导演以科幻与犯罪片闻名的战争电影。HippoRAG 能够在前四个段落中找到答案——Ridley Scott 执导的 *Black Hawk Down*；ColBERTv2 则完全漏掉答案，转而检索其他电影与电影列表。在该例中，尽管 IRCoT 能检索到 Ridley Scott，但主要依赖的是参数知识。其思维链输出讨论了 Ridley Scott 与 Denis Villeneuve 的知名度，以及二人在科幻与犯罪题材方面的经验。由于本文将迭代限制为三步，而且需要探索两位导演，系统未能识别具体战争电影 *Black Hawk Down*。前两个问题虽稍显曲折，却正是人们仅凭少量互不相连的细节，试图回忆曾看过或听说过的某部电影或某本书时常提出的问题。

最后，第三个问题更接近正文中的动机示例，也展示了此类问题在现实领域中的重要性。该问题要求找出一种通过特定机制——与细胞质 p53 相互作用——治疗淋巴细胞白血病的药物。HippoRAG 能利用支持段落中的关联，判断 Chlorambucil 段落最为重要；ColBERTv2 与 IRCoT 则只能抽取与淋巴细胞白血病有关的段落。有趣的是，IRCoT 利用参数知识猜测同样治疗白血病的 Venetoclax 会通过相关机制起作用，尽管整理后的数据集中没有任何段落明确说明这一点。

**表 10：多种路径查找型多跳问题上不同方法的排序结果示例。**

| 问题 | HippoRAG | ColBERTv2 | IRCoT |
|---|---|---|---|
| 哪本书出版于 2012 年，作者是一位获得过 Whitbread Award 的 English 作家？ | 1. *Oranges Are Not the Only Fruit*；2. *William Trevor Legacies*；3. Mark Haddon | 1. World Book Club Prize winners；2. Leon Garfield Awards；3. *Twelve Bar Blues*（小说） | 1. Kate Atkinson；2. Leon Garfield Awards；3. *Twelve Bar Blues*（小说） |
| 哪部改编自非虚构书籍的战争电影，是由一位以科幻和犯罪类型片闻名的人执导的？ | 1. War Film；2. Time de Zarn；3. Outline of Sci-Fi；4. *Black Hawk Down* | 1. Paul Greengrass；2. List of book-based war films；3. Korean War Films；4. *All the King’s Men* Book | 1. Ridley Scott；2. Peter Hyams；3. Paul Greengrass；4. List of book-based war films |
| 哪种药物通过与细胞质 p53 相互作用来治疗慢性淋巴细胞白血病？ | 1. Chlorambucil；2. Lymphocytic leukemia；3. Mosquito bite allergy | 1. Lymphocytic leukemia；2. Obinutuzumab；3. Venetoclax | 1. Venetoclax；2. Lymphocytic leukemia；3. Idelalisib |

## F 错误分析

### F.1 概览

本节详细分析 HippoRAG 在 MuSiQue 数据集上产生的 100 个错误。如表 11 所示，这些错误可分为三大类：NER、OpenIE 和 PPR。

占比最高的错误类型接近全部错误样例的一半，源于基于 NER 的设计局限。第 F.2 节进一步说明，我们的 NER 设计未能从查询中抽取足够信息用于检索。例如，对于问题“某款互联网浏览器的 Windows 8 版本何时开放使用？”，系统只抽取短语 `Windows 8`，未将关于“浏览器”或“开放使用”的任何信号留给后续图搜索。第二常见的 OpenIE 错误将在第 F.3 节详细讨论。

第三类错误是：NER 与 OpenIE 都正常工作，但 PPR 算法仍无法识别相关子图；这通常源于混淆信号。例如，考虑查询“有多少难民移居到了那个 Huguenots 因亲缘感而选择移民的欧洲国家？”尽管系统从问题和支持段落中都正确抽取了 `Huguenots`，而且 PPR 从标记为 `European` 与 `Huguenots` 的节点开始运行，但它仍难以在这些节点周围找到能够确定最相关段落的恰当子图。当语料库中有多篇段落讨论极其相似的主题时，就会发生这种情况，因为 PPR 算法无法直接利用查询上下文。

**表 11：MuSiQue 上的错误分析。**

| 错误类型 | 错误占比（%） |
|---|---:|
| NER 局限 | 48 |
| OpenIE 错误或缺失 | 28 |
| PPR | 24 |

### F.2 概念与上下文之间的权衡

由于本方法在抽取与索引中以实体为中心，它强烈偏向概念，许多上下文信号因而未被利用。这一设计使单步多跳检索成为可能，同时也能防止上下文线索分散对更显著实体的注意。如表 12 的第一个示例所示，ColBERTv2 利用上下文检索出与著名 Spanish 航海家有关的段落，却没有找到拳击手 `Sergio Villanueva`。HippoRAG 则能聚焦于 `Sergio`，检索出一篇相关段落。

遗憾的是，这一设计也是本方法最大的局限之一：在小规模错误分析中，忽略上下文线索约占 48% 的错误。第二个示例中，这一问题更明显，因为其中的概念较为宽泛，上下文因而更加重要。HippoRAG 标注的唯一概念是 `protons`，于是它抽取了与 `Uranium` 和 `nuclear weapons` 有关的段落；ColBERTv2 则利用上下文，抽取了与原子序数发现有关、相关性更高的段落。

**表 12：MuSiQue 上展示概念-上下文权衡的示例。**

| 问题 | HippoRAG | ColBERTv2 |
|---|---|---|
| 谁的父亲是一位航海家，曾探索 Sergio Villanueva 后来出生的大陆地区东海岸？ | Sergio Villanueva；César Gaytan；Faustino Reyes | Francisco de Eliza（航海家）；Exploration of N. America；Vicente Pinzón（航海家） |
| 哪项事业包括了发现“每种元素的原子所含质子数具有唯一性”的那个人？ | Uranium；Chemical element；History of nuclear weapons | Atomic number；Atomic theory；Atomic nucleus |

**表 13：单步检索性能。** HippoRAG 在 MuSiQue 与 2WikiMultiHopQA 上显著优于全部基线，并在难度较低的 HotpotQA 数据集上取得相当的性能。

| 模型 | 检索器 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 | HotpotQA R@2 | HotpotQA R@5 | 平均 R@2 | 平均 R@5 |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 基线 | Contriever | 34.8 | 46.6 | 46.6 | 57.5 | 57.2 | 75.5 | 46.2 | 59.9 |
| 基线 | ColBERTv2 | 37.9 | 49.2 | 59.2 | 68.2 | 64.7 | 79.3 | 53.9 | 65.6 |
| HippoRAG | Contriever | 41.0 | 52.1 | 71.5 | **89.5** | 59.0 | 76.2 | 57.2 | 72.6 |
| HippoRAG | ColBERTv2 | 40.9 | 51.9 | 70.7 | 89.1 | 60.5 | 77.7 | 57.4 | 72.9 |
| HippoRAG + 不确定性集成 | Contriever | 42.3 | 54.5 | 71.3 | 87.2 | 60.6 | 79.1 | 58.1 | 73.6 |
| HippoRAG + 不确定性集成 | ColBERTv2 | **42.5** | **54.8** | **71.9** | 89.0 | **62.5** | **80.0** | **59.0** | **74.6** |

为在概念与上下文之间取得更好的平衡，我们引入一种集成设置：当海马旁区对查询实体与 KG 实体之间的链接表现出不确定性时，将 HippoRAG 分数与稠密检索器分数集成。该过程对应这样的情形：上游海马旁信号未能充分激活任何海马索引，因此必须更强地依赖新皮层。

仅当某个查询-KG 实体分数 $\operatorname{cosine\_similarity}(M(c_i),M(e_j))$ 低于阈值 $\theta$ 时，我们才使用不确定性集成。例如，若 KG 中没有 `Stanford` 节点，而 KG 中最接近的节点是 `Stanford Medical Center`，且二者余弦相似度低于 $\theta$，就会触发该机制。不确定性集成的最终段落分数，是 HippoRAG 分数与模型 $M$ 的标准段落检索分数的平均值；在取平均之前，两种分数都会在全部段落上归一化到 0 至 1。

如表 13 所示，在“不确定性集成”设置下将 HippoRAG 与 $M$ 集成后，MuSiQue 上的性能进一步提升，HotpotQA 上的 R@5 也超过基线。与 IRCoT 结合时，如表 14 所示，ColBERTv2 集成在 HotpotQA 的 R@2 与 R@5 上都优于所有先前基线。尽管这一简单方法展现出潜力，但要解决这一概念-上下文权衡，仍需开展更多研究，因为简单集成在某些情况下会降低性能，尤其是在 2WikiMultiHopQA 数据集上。

**表 14：多步检索性能。** 将 HippoRAG 与 IRCoT 等标准多步检索方法结合，可在三个数据集上获得显著提升。

| 模型 | 检索器 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 | HotpotQA R@2 | HotpotQA R@5 | 平均 R@2 | 平均 R@5 |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| IRCoT | Contriever | 39.1 | 52.2 | 51.6 | 63.8 | 65.9 | 81.6 | 52.2 | 65.9 |
| IRCoT | ColBERTv2 | 41.7 | 53.7 | 64.1 | 74.4 | 67.9 | 82.0 | 57.9 | 70.0 |
| IRCoT + HippoRAG | Contriever | 43.9 | 56.6 | 75.3 | 93.4 | 65.8 | 82.3 | 61.7 | 77.4 |
| IRCoT + HippoRAG | ColBERTv2 | **45.3** | 57.6 | **75.8** | **93.9** | 67.0 | 83.0 | **62.7** | 78.2 |
| IRCoT + HippoRAG + 不确定性集成 | Contriever | 44.4 | **58.5** | 75.3 | 91.5 | 66.9 | 85.0 | 62.2 | **78.3** |
| IRCoT + HippoRAG + 不确定性集成 | ColBERTv2 | 40.2 | 53.4 | 74.5 | 91.2 | **68.2** | **85.3** | 61.0 | 76.6 |

### F.3 OpenIE 的局限

OpenIE 是从非结构化文本中抽取结构化知识的关键步骤，但其缺陷可能造成知识缺口，损害检索与 QA 能力。如表 15 所示，GPT-3.5 Turbo 在 OpenIE 过程中遗漏了关键歌曲标题 `Don’t Let Me Wait Too Long`，而该标题是段落中最重要的元素。一个可能原因是，模型对这种较长实体不够敏感。此外，模型没有准确捕捉战争的开始与结束年份，而这些信息对查询至关重要。这也是模型经常忽略时间属性的一个例子。总体而言，这些失败凸显出改善关键信息抽取的必要性。

**表 15：MuSiQue 上的开放信息抽取错误示例。**

| 问题 | 段落 | 遗漏的三元组 |
|---|---|---|
| 哪家公司负责发行包含 `Don’t Let Me Wait Too Long` 的唱片？ | 在 LP 的 A 面曲序中，`Don’t Let Me Wait Too Long` 位于民谣 `The Light That Has Lighted the World` 与 `Who Can See It` 之间…… | (`Don’t Let Me Wait Too Long`, `sequenced on`, `side one of the LP`) |
| Confederate States of America 的总统何时结束参与 Mexican-American War？ | Jefferson Davis 作为一支志愿团的上校参加了 Mexican-American War（1846-1848）…… | (`Mexican-American War`, `starts`, `1846`)；(`Mexican-American War`, `ends`, `1848`) |

### F.4 OpenIE 文档长度分析

最后，我们开展一项小规模内在实验，以理解 OpenIE 方法在段落长度增加时的稳健性。表 16 的长度相关评估结果显示，与较短段落相比，GPT-3.5-Turbo 在较长段落上的 OpenIE 结果显著恶化。这很可能是因为较长段落中的句子和段落结构更加复杂，从而导致抽取质量下降。要解决这一局限仍需进一步研究，因为继续切分文本只会因句间依赖而引入其他问题。

**表 16：使用 CaRB [6] 框架进行的 OpenIE 内在评估。** 使用默认 GPT-3.5 Turbo (1106) 模型时，10 篇最长标注段落与 10 篇最短标注段落之间的性能差异。

| 段落组 | AUC | 精确率 | 召回率 | F1 |
|---|---:|---:|---:|---:|
| 10 篇最短段落 | 58.9 | 79.2 | 65.7 | 71.8 |
| 10 篇最长段落 | 39.0 | 60.7 | 48.5 | 53.9 |

## G 成本与效率比较

相比迭代检索方法，HippoRAG 的主要优势之一，是其单步多跳检索能力在成本与时间两方面都带来了显著的在线检索效率提升。具体而言，如表 17 所示，IRCoT 的检索成本是 HippoRAG 的 10 至 30 倍，因为 HippoRAG 只需从查询中抽取相关命名实体，而无须处理所有检索到的文档。在使用量极高的系统中，这种相差一个数量级的成本差异可能极为重要。IRCoT 与 HippoRAG 的延迟差异同样显著，不过精确测量更具挑战。如表 17 所示，视需执行的检索轮数而定（本文实验为 2-4 轮），HippoRAG 可比 IRCoT 快 6 至 13 倍。[^6]

**表 17：在 1,000 个查询上使用 GPT-3.5 Turbo 进行在线检索的平均成本与效率。**

| 指标 | ColBERTv2 | IRCoT | HippoRAG |
|---|---:|---:|---:|
| API 成本（美元） | 0 | 1-3 | 0.1 |
| 时间（分钟） | 1 | 20-40 | 3 |

尽管 HippoRAG 的离线索引时间与成本高于 IRCoT——每 10,000 个段落约慢 10 倍、多花 15 美元[^7]——但使用开源 LLM 可以大幅降低这些成本。表 5 的消融研究表明，Llama-3.1-70B-Instruct [1] 的表现与 GPT-3.5 Turbo 相近；与此同时，如表 18 所示，它可以通过 vLLM [40] 在本地使用 4 块 H100 GPU 部署，并在约 4 小时内为 10,000 篇文档建立索引。此外，由于本地部署该模型还能进一步降低成本，因此，大规模使用 HippoRAG 的门槛完全可能落在许多机构的计算预算之内。

最后需要指出，即使 LLM 生成成本下降，前述在线检索效率优势仍会保持不变，因为 IRCoT 与 HippoRAG 所需词元数之比不会改变，而且 LLM 很可能仍是系统的主要计算瓶颈。

**表 18：使用 GPT-3.5 Turbo 和通过 vLLM 本地部署的 Llama-3.1（8B 与 70B），对 10,000 个段落进行离线索引的平均成本与延迟。**

| 模型 | 指标 | ColBERTv2 | IRCoT | HippoRAG |
|---|---|---:|---:|---:|
| GPT-3.5 Turbo-1106（主要结果） | API 成本（美元） | 0 | 0 | 15 |
|  | 时间（分钟） | 7 | 7 | 60 |
| GPT-3.5 Turbo-0125 | API 成本（美元） | 0 | 0 | 8 |
|  | 时间（分钟） | 7 | 7 | 60 |
| Llama-3.1-8B-Instruct | API 成本（美元） | 0 | 0 | 0 |
|  | 时间（分钟） | 7 | 7 | 120 |
| Llama-3.1-70B-Instruct | API 成本（美元） | 0 | 0 | 0 |
|  | 时间（分钟） | 7 | 7 | 250 |

[^6]: 在 IRCoT 与 HippoRAG 的在线检索中，我们都使用单线程查询 OpenAI API。由于 IRCoT 是迭代过程，各轮必须依次执行，因此这种速度比较是合理的。
[^7]: 为加快索引速度，我们使用 10 个线程并行调用 OpenAI API 上的 `gpt-3.5-turbo-1106`。论文写作时，API 价格为每百万输入词元 1 美元、每百万输出词元 2 美元。

## H 实现细节与计算需求

除第 3.4 节所述细节外，我们还为 Contriever [35] 和 ColBERTv2 [70] 使用基于 PyTorch [59] 与 HuggingFace [86] 的实现。PPR 算法采用 `python-igraph` [13] 实现，BM25 则使用 Elasticsearch [24]。多步检索采用与 IRCoT [78] 相同的提示实现，并在每一步检索 top-10 段落。根据各数据集的最大推理链长度，我们将 HotpotQA 与 2WikiMultiHopQA 的最大推理步数设为 2，将 MuSiQue 设为 4。

我们用包括 HippoRAG 在内的各检索方法替换 IRCoT 的基础检索器 BM25，以此将 IRCoT 与不同检索器结合；下文将这种组合记为“IRCoT + HippoRAG”。[^8] 对于 QA 阅读器，我们使用检索到的 top-5 段落作为上下文，并采用单样本 QA 演示与 CoT 提示策略 [78]。

就计算需求而言，遗憾的是，OpenAI 并未披露本研究大部分计算需求。运行 ColBERTv2 与 Contriever 进行索引和检索时，我们使用 4 块显存为 48 GB 的 NVIDIA RTX A6000 GPU。使用 Llama-3.1 模型建立索引时，我们使用 4 块显存为 80 GB 的 NVIDIA H100 GPU。最后，个性化 PageRank 算法运行在 2 颗 AMD EPYC 7513 32 核处理器上。

[^8]: 原始 IRCoT 不为每篇检索到的段落提供分数，因此我们在迭代检索过程中采用束搜索。束搜索期间，每篇候选段落保留其历史最高分。

## I LLM 提示词

索引与查询 NER 所用提示词分别见图 7 和图 8，OpenIE 提示词见图 9。

### 图 7：索引时用于段落 NER 的提示词

````text
指令：

你的任务是从给定段落中抽取命名实体。
请以实体的 JSON 列表作答。

单样本演示：

段落：
```
Radio City
Radio City 是 India 的首家私营 FM 广播电台，创办于 2001 年 7 月 3 日。它播放 Hindi、English
和地区歌曲。Radio City 最近于 2008 年 5 月进军 New Media，推出音乐门户
PlanetRadiocity.com；该网站提供音乐相关新闻、视频、歌曲以及其他音乐相关功能。
```

{"named_entities": ["Radio City", "India", "3 July 2001", "Hindi", "English", "May 2008",
"PlanetRadiocity.com"]}

输入：

段落：
```
待索引段落
```
````

**图 7：索引期间用于段落 NER 的提示词。**

### 图 8：检索时用于查询 NER 的提示词

````text
指令：

你是一个非常有效的实体抽取系统。请抽取对解决下列问题十分重要的全部命名实体，并以 JSON
格式给出这些命名实体。

单样本演示：

问题：Arthur’s Magazine 与 First for Women，哪本杂志创办得更早？

{"named_entities": ["First for Women", "Arthur’s Magazine"]}

输入：

问题：待索引查询
````

**图 8：检索期间用于查询 NER 的提示词。**

### 图 9：索引时用于 OpenIE 的提示词

````text
指令：

你的任务是根据给定段落与命名实体列表构建一张 RDF（资源描述框架）图。
请以三元组的 JSON 列表作答，其中每个三元组表示 RDF 图中的一种关系。
请注意以下要求：
- 每个三元组都应包含对应段落实体列表中的至少一个命名实体，最好包含两个。
- 为保持表意清楚，请将代词明确解析为其指代的具体名称。

将段落转换为一个 JSON 字典，其中包含命名实体列表与三元组列表。

单样本演示：

段落：
```
Radio City
Radio City 是 India 的首家私营 FM 广播电台，创办于 2001 年 7 月 3 日。它播放 Hindi、English
和地区歌曲。Radio City 最近于 2008 年 5 月进军 New Media，推出音乐门户
PlanetRadiocity.com；该网站提供音乐相关新闻、视频、歌曲以及其他音乐相关功能。
```
{"named_entities": ["Radio City", "India", "3 July 2001", "Hindi", "English", "May 2008",
"PlanetRadiocity.com"]}

{"triples":
   [
      ["Radio City", "位于", "India"],
      ["Radio City", "是", "私营 FM 广播电台"],
      ["Radio City", "创办于", "3 July 2001"],
      ["Radio City", "播放……语言的歌曲", "Hindi"],
      ["Radio City", "播放……语言的歌曲", "English"],
      ["Radio City", "进军", "New Media"],
      ["Radio City", "推出", "PlanetRadiocity.com"],
      ["PlanetRadiocity.com", "推出于", "May 2008"],
      ["PlanetRadiocity.com", "是", "音乐门户"],
      ["PlanetRadiocity.com", "提供", "新闻"],
      ["PlanetRadiocity.com", "提供", "视频"],
      ["PlanetRadiocity.com", "提供", "歌曲"]
   ]
}

输入：

将段落转换为一个 JSON 字典，其中包含命名实体列表与三元组列表。
段落：
```
待索引段落
```
{"named_entities": [NER 列表]}
````

**图 9：索引期间用于 OpenIE 的提示词。**

## Sources

- `papers/agent-memory/HippoRAG - Neurobiologically Inspired Long-Term Memory for Large Language Models/HippoRAG - Neurobiologically Inspired Long-Term Memory for Large Language Models.pdf`（全文 31 页；已读取正文、图表、脚注、参考文献与全部附录）
