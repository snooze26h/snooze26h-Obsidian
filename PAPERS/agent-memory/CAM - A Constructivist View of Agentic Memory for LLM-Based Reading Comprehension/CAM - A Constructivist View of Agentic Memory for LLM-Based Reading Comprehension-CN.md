# CAM：面向 LLM 阅读理解的智能体记忆——一种建构主义视角

Rui Li¹˒∗˒†˒‡，Zeyu Zhang¹˒†˒‡，Xiaohe Bo¹˒†˒‡，Zihang Tian¹˒†˒‡  
Xu Chen¹˒†˒‡˒§，Quanyu Dai²˒§，Zhenhua Dong²，Ruiming Tang²

¹ 中国人民大学高瓴人工智能学院  
² 华为诺亚方舟实验室  
{lirui121200, xu.chen}@ruc.edu.cn　daiquanyu@huawei.com

> 译注：本文发表于第 39 届神经信息处理系统大会（NeurIPS 2025）。CAM 是 Constructivist Agentic Memory（建构主义智能体记忆）的缩写。方法名、模型名和数据集名保留英文；术语首次出现时酌情附英文。公式、图表、脚注及引文编号均与原文一致。参考文献保留原文书目信息，以便准确检索。

∗ 本工作于 Rui Li 在华为诺亚方舟实验室实习期间完成。  
† 北京市大模型与智能治理重点实验室。  
‡ 教育部下一代智能搜索与推荐工程研究中心。  
§ 通信作者。

## 摘要

当前，大语言模型（Large Language Model，LLM）在理解长篇文档时面临海量信息带来的压力。这一挑战凸显了构建统一记忆模块的必要性：它能够将原始 LLM 提升为自主阅读智能体。尽管一些启发式方法已经出现，但该领域仍缺乏系统性的设计原则。为填补这一空白，我们从 Jean Piaget 的建构主义理论中汲取灵感，由此揭示智能体记忆的三项特征：结构化图式、灵活同化与动态顺应。这一蓝图为面向 LLM 阅读理解构建更稳健、更高效的记忆系统指明了清晰路径。为此，我们开发了 CAM——建构主义智能体记忆（Constructivist Agentic Memory）的一种原型实现，同时体现结构性、灵活性与动态性。CAM 的核心是一种用于结构化记忆发展的增量式重叠聚类算法，既支持连贯的层级摘要，也支持在线批量整合。在推理阶段，CAM 会自适应探索记忆结构，激活与查询相关的信息以生成有上下文依据的回答，这一过程类似于人类的联想过程。与现有方法相比，我们的设计在多种长文本阅读理解任务上同时展现出性能与效率优势，任务包括问答、基于查询的摘要和主张验证。

## 1 引言

基于 Transformer 的大语言模型（LLM）[1–4] 已成为自然语言理解领域具有变革意义的工具。然而，当面对极长文档时，其能力往往会减弱 [5, 6]。这一挑战不仅源于 LLM 有限的上下文容量，更根本的原因在于：关键的信息片段可能散布于长文本中彼此相距很远的位置，模型难以感知并汇聚这些片段 [7, 8]。

解决这些局限的一种常见方法，是为 LLM 配备显式记忆模块 [9]，用于存储和检索信息，其方式类似于人类的认知模式。此类模块维护一份关于所摄取内容的统一“心理”表征，使 LLM 能够充当自主阅读智能体，存储、检索并推理长距离依赖关系。然而，尽管智能体记忆方法近来大量涌现 [10–18]，它们通常只在表层模仿人类记忆，缺乏一套连贯原则作为设计基础。因此，我们提出一个关键问题：面向基于 LLM 的阅读智能体，有效记忆模块应具备哪些特征？

**图 1：建构主义视角下的记忆发展过程。** 图中标签依次为：顺应（Accommodation）、新图式（New Schemata）、调整后的图式（Adapted Schemata）与同化（Assimilation）。新信息通过同化纳入已有图式；当信息无法直接纳入时，则通过顺应形成新图式或调整既有图式。

为回答这一基本问题，我们越过表层设计，转而探究人类认知的内在基础。具体而言，我们借鉴 Jean Piaget 的建构主义理论 [19–22]；该理论是认知科学的基石，深刻塑造了人们对人类记忆如何发展的认识。根据这一理论，记忆是一套不断演化的心理系统，它会主动把接收到的信息组织成连贯的认知结构（即图式）。新信息到来时，两项基本操作（如图 1 所示）会驱动图式的发展过程：（1）**同化**，即把新信息纳入当前的记忆图式 [23]；（2）**顺应**，即当新信息无法轻易纳入时，改变当前图式 [22]。这两项操作共同维持记忆图式的认知平衡化 [19–21]，亦即一种均衡的生长状态，使记忆能够更准确地形成对已接收信息的心理表征。值得注意的是，同化具有灵活性，允许每个信息单元整合到图式中的多个相关位置；顺应则具有动态性，使图式能够通过局部调整持续演化，而无需在每次输入到来时整体重建。

受这一建构主义视角启发，我们提出三项关键的记忆特征，用于指导基于 LLM 的阅读智能体设计：**结构化图式、灵活同化和动态顺应**。然而，当前的智能体记忆系统从根本上偏离了这一设计原则。如表 1 所示，早期工作（如 MemGPT [11]、MemoryBank [12] 和 ReadAgent [13]）把记忆视为表格式存储库，分别存放文本块或压缩后的要旨。这种非结构化表示无法捕捉长文本背后的信息关联。近期一些方法（如 RAPTOR [14]、GraphRAG [15]、HippoRAG [16] 和 MemTree [18]）认识到记忆结构性的重要性，却没有任何一种方法在记忆结构发展过程中同时体现灵活性和动态性。因此，我们进一步寻求一种遵循建构主义设计原则、面向基于 LLM 的阅读智能体的新型记忆实现。

**表 1：建构主义视角下的方法比较。** 更多细节见附录 B。

| 方法 | 类型 | 结构性 | 灵活性 | 动态性 |
|---|---|:---:|:---:|:---:|
| MemGPT [11] | 在线 | ✗ | ✗ | ✗ |
| ReadAgent [13] | 在线 | ✗ | ✗ | ✗ |
| RAPTOR [14] | 离线 | ✓ | ✓ | ✗ |
| GraphRAG [15] | 离线 | ✓ | ✗ | ✗ |
| MemTree [18] | 在线 | ✓ | ✗ | ✗ |
| CAM（本文） | 在线（批量） | ✓ | ✓ | ✓ |

为此，我们提出 CAM——建构主义智能体记忆的一种原型，旨在增强 LLM 的长文本阅读理解能力。为实现记忆的结构性，CAM 会主动把输入文档组织成统一的层级架构。在基础层，CAM 由原始文本块构建一张连贯的语义网络，同时捕捉文本相似性和叙事连贯性。更高层的记忆节点则是由紧密相关的低层节点汇聚形成的抽象摘要。在构建这一层级结构的过程中，CAM 配备了一种局部优先的增量式重叠聚类算法，既支持灵活同化（即一个低层记忆节点可以为多个高层抽象作出贡献），也支持动态顺应（即记忆结构可以高效适应新输入）。尤其值得注意的是，我们的设计天然允许 CAM 分批整合新到达的文本块；相较仍局限于离线或逐条在线设置的现有方法，它具有显著的效率优势（速度提升超过 4 倍，见第 5.3 节）。在推理时，CAM 采用“剪枝-生长”（Prune-and-Grow）联想策略，激活与查询相关的记忆节点，从而生成有上下文依据的回答，并在多种长文本阅读任务上取得很高的准确性。

本文贡献概括如下：

- **蓝图：** 我们借鉴 Piaget 的建构主义理论，为面向 LLM 阅读理解的智能体记忆提出了一项明确设计原则，强调记忆模块的三项关键特征：结构化图式、灵活同化与动态顺应。
- **原型：** 我们进一步开发了建构主义智能体记忆原型 CAM，以增强 LLM 的阅读理解能力；该原型使用增量式重叠聚类算法发展记忆，并使用剪枝-生长策略检索记忆。
- **评估：** 我们在问答、基于查询的摘要和主张验证基准上评估了这一设计，覆盖单文档和多文档场景。结果表明，在性能与效率两方面，CAM 均优于现有方法。

第 39 届神经信息处理系统大会（NeurIPS 2025）。

## 2 背景

**长上下文 LLM。** 由于上下文容量有限，LLM 难以处理长文本。最直接的解决办法，是利用扩展后的上下文窗口对 LLM 进行微调 [24–29]。然而，即使输入文本没有超过规定的上下文长度，LLM 的性能仍倾向于随文本增长而下降 [5, 6, 8]。针对这一问题，大量工作转而采用记忆增强型 LLM 阅读智能体。

**表格式记忆。** 早期基于 LLM 的阅读智能体把长文本切分为短块并存入记忆 [10–13]。收到用户查询后，系统召回相关文本块以辅助 LLM 推理，这与标准检索增强生成技术 [30–32] 一致。MemoryBank [12] 和 Ret-LLM [33] 使用稠密检索模型及读写函数管理完整历史记录。MemGPT [11] 采用类似缓存的架构，为近期信息赋予更高优先级。SCM [34] 通过记忆流和控制器模块增强 LLM 维持长期记忆的能力。ReadAgent [13] 把每个文本块压缩为要旨记忆，并借助交互式查找过程，按具体任务需要访问相关信息。此类非结构化记忆设计虽然易于实现，但当关键信息分散在多个条目中时，其能力不足 [8, 17]。

**结构化记忆。** 近期进展认识到非结构化信息存储的局限，开始转向结构化记忆系统的设计。MemWalker [35] 自底向上地把长上下文处理成一棵摘要节点树，并沿树导航以寻找与推理相关的内容。类似地，RAPTOR [14] 也把文本组织成递归树，在每一层对摘要聚类，以形成更高层理解。GraphRAG [15] 从上下文中抽取实体与关系以构建知识图谱，并为相关实体群体生成摘要。收到问题后，每个摘要节点提供部分信息，系统再将其组合成最终答案。GraphReader [36] 也把输入文本转换为图结构，并使用一组预定义函数辅助推理过程中的规划与反思。尽管这些工作能够有效组织大规模文本数据，增强检索和生成能力，但它们局限于静态语料库，整合新信息时需要完整重建。为解决这一问题，MemTree [18] 使用在线、自顶向下的聚类算法维护动态树状记忆。然而，它只能依次整合新文本块，并存在结构失衡的潜在风险 [37]。此外，如表 1 所示，这些工作均未在记忆发展过程中同时体现灵活性和动态性。更多讨论见附录 B。

## 3 蓝图：结构性、灵活性与动态性

作为第一步，我们力求为基于 LLM 的阅读智能体提出一份明确的记忆设计蓝图。不同于现有文献中的启发式方法，我们把设计建立在一套基础性的认知发展框架之上——Jean Piaget 的建构主义理论（概述见附录 A）。受该理论启发，我们把三项关键的记忆特征设为设计目标：结构化图式、灵活同化和动态顺应。这些特征并非相互排斥，而是协同发挥作用，帮助系统整合、保留并检索长文本中的信息。

### 3.1 结构化图式

依照 Piaget 的建构主义理论，记忆系统应当主动、自底向上地把所有接收到的信息重组为层级图式。形式化地说，给定一组输入信息单元（如原始文本块）$V=\{v_1,v_2,\ldots,v_n\}$，记忆首先感知这些基本单元之间的潜在关联 [38]，构建基础语义网络 $G_0=(V,E)$，其中 $E$ 是捕捉单元节点之间语义连贯性的边集合。随后，系统把紧密相关的单元汇聚为更高层超节点，形成一张表示抽象理解的粗粒度网络。对这些抽象反复进行进一步汇聚，便可递归构建如下记忆图式层级：

$$
\mathcal{M}=\left(\{G_l\}_{l=0}^{L},\{\psi_l\}_{l=1}^{L}\right),
\tag{1}
$$

其中，$G_l=(V_l,E_l)$ 表示第 $l$ 层的图，$V_l$ 和 $E_l$ 分别为节点集合和边集合；$\psi_l:G_{l-1}\rightarrow G_l$ 是向上映射，反映 $G_{l-1}$ 中的低层元素与 $G_l$ 中高层抽象之间的隶属关系。这种层级结构性至关重要，因为它能够自然地无缝整合抽象概念和细粒度细节，为深入理解及准确回忆复杂信息建立统一框架。$\mathcal{M}$ 的构建由两项主要认知过程驱动：同化与顺应。二者共同构成信息如何纳入记忆结构、以及结构如何调整以维持连贯性的基础。

### 3.2 灵活同化

同化是指在不大幅改变记忆图式结构的情况下，把信息纳入其中。形式化地说，给定一批新信息单元 $V_{\mathrm{new}}=\{v_{n+1},\ldots,v_{n+m}\}$，记忆系统首先把这些单元组织进基础语义网络 $G_0$，将其更新为 $G'_0=(V\cup V_{\mathrm{new}},E\cup E_{\mathrm{new}})$；其中，$E_{\mathrm{new}}$ 是新单元与当前节点之间、以及新单元彼此之间的关联边集合。这一步确保基础网络始终完整表示所有输入信息构成的语义图景。

在 $G'_0$ 之上，同化的下一个关键步骤，是为每个新增节点 $v_{\mathrm{new}}\in V_{\mathrm{new}}$ 建立层级隶属关系。具体而言，记忆系统通过确定向上映射 $\psi_1(v_{\mathrm{new}})$，把 $v_{\mathrm{new}}$ 与 $G_1$ 中的高层抽象关联起来。该映射既可以把每个节点分配给已有抽象（即 $\psi_1(v_{\mathrm{new}})\subseteq V_1$），以丰富其语义范围；也可以在 $G_1$ 中创建全新的抽象节点，从而在更高记忆层触发进一步同化。

同化的关键特征在于其灵活性：每个低层信息单元可以同时丰富多个高层抽象，亦即 $\psi$ 应为多对多映射。通过允许这种重叠隶属关系，记忆图式能够捕捉复杂输入信息的多面性——单个单元可能同时关联多个主题、话题或概念。

### 3.3 动态顺应

同化过程中，随着新节点和新边的整合，记忆图式逐步扩展。然而，这种扩展可能使记忆层级变得次优，或与更新后的上下文不再一致。例如，某个抽象节点可能负载过重，从而削弱其表示下属单元的有效性。对此，顺应会改变层级结构中受到影响的部分，重新分配下属单元并重新校准抽象节点，以恢复结构连贯性和认知平衡化。这一动态过程使记忆图式保持均衡，并能高效适应新信息，而无需进行全局重建。

这一记忆特征对于处理现实场景中的长篇文档尤其重要，因为文本可能以增量方式到达（如流式或批量设置）。例如，当一本书的新章节到来时，记忆系统应当只细化图式中的相关部分来整合它——例如更新代表全书总体叙事的高层抽象——而非从头重建整个层级。这项能力保证了可扩展性与效率，使记忆系统能够以连续、在线的方式适应新输入。

**图 2：CAM 的记忆发展（第 4.1 节）与记忆检索（第 4.2 节）过程。** 记忆发展包括：（a）基础网络扩展；（b）自我中心式解耦，通过节点复制体现灵活性；（c）在线聚类更新，通过局部结构调整体现动态性。记忆检索采用（d）自适应剪枝-生长策略，由查询驱动节点激活与扩展。

## 4 原型：建构主义智能体记忆

基于上述蓝图，我们进一步开发了建构主义智能体记忆（CAM）原型，用于基于 LLM 的长篇文档阅读理解。如图 2 所示，该框架在阅读记忆的构建过程中体现结构性、灵活性与动态性（第 4.1 节），并采用自适应推理策略处理用户查询（第 4.2 节）。

### 4.1 记忆发展

在阅读阶段，CAM 应当把输入文本灵活地同化进一个连贯层级，并动态地加以顺应。我们的技术实现基于两点观察。第一，灵活同化与重叠聚类高度契合：紧密相关的记忆节点形成簇，而每个节点可以属于多个簇。第二，动态顺应对应于增量聚类：簇结构会响应新输入而演化。这两项认知过程相互交织，而非彼此独立，因此最好采用一种紧凑、统一的算法设计，同时整合二者的功能。由此，我们为 CAM 配备了一种用于记忆发展的增量式重叠聚类算法，在保持简洁的同时保证效率。具体而言，该过程包含三个主要步骤：

1. **基础网络扩展：** 将新文本块 $V_{\mathrm{new}}$ 整合进基础语义网络 $G_0=(V,E)$（初始为空）；依据文本相关性和叙事连贯性建立新边 $E_{\mathrm{new}}$。
2. **自我中心式解耦：** 对受影响节点 $A$（新节点及其邻居节点），CAM 分析其局部结构，以更新 $G_0$ 的副本网络 $\widetilde G_0$（初始为空）；该网络通过节点复制，显式解耦重叠簇。
3. **在线聚类更新：** 在无重叠的副本网络 $\widetilde G_0$ 上，应用增量式标签传播算法修改簇分配。对于发生变化的簇，CAM 使用 LLM 更新抽象节点；受影响的超节点则会在下一层进一步触发上述过程（步骤 2 和步骤 3）。

下文详细阐述这三个步骤。为清楚起见，我们重点介绍从基础层 $G_0$ 到上一层 $G_1$ 的发展过程；这一过程可以自然扩展到更高层。

#### 4.1.1 基础网络扩展

给定 $m$ 个新增且连续的文本块 $V_{\mathrm{new}}=\{v_{n+1},v_{n+2},\ldots,v_{n+m}\}$，CAM 首先把这些信息单元纳入基础语义网络 $G_0=(V,E)$，为后续的层级发展奠定基础。为捕捉已接收文本块之间的文本相关性和叙事连贯性，我们定义了一个综合评分函数，同时融合语义相似度和位置邻近度。

形式化地说，对于每一对文本块 $v_i\in V_{\mathrm{new}}$ 和 $v_j\in V\cup V_{\mathrm{new}}$，语义相似度由预训练模型 $f_{\mathrm{emb}}(\cdot)$ 生成的嵌入之间的余弦相似度计算。此外，我们用高斯相似度衡量 $v_i$ 与 $v_j$ 的位置邻近度，使位置更接近的文本块获得更高分。综合这两个方面，总体相似度得分定义为二者的线性插值：

$$
s(v_i,v_j)=\alpha\cdot
\frac{f_{\mathrm{emb}}(v_i)\cdot f_{\mathrm{emb}}(v_j)}
{\lVert f_{\mathrm{emb}}(v_i)\rVert\lVert f_{\mathrm{emb}}(v_j)\rVert}
+(1-\alpha)\cdot\exp\left(-\frac{(i-j)^2}{2\sigma^2}\right),
\tag{2}
$$

其中，$\alpha\in[0,1]$ 是权重系数，$i$ 和 $j$ 分别为 $v_i$ 和 $v_j$ 的位置索引，$\sigma$ 控制邻近度影响的衰减速度。对于每个接收到的文本块，我们找出相似度超过预定义阈值 $\theta$ 的 top-$k$ 个相关节点，并在它们之间建立新边，形成扩展后的基础图

$$
G'_0=(V',E')=(V\cup V_{\mathrm{new}},E\cup E_{\mathrm{new}}).
$$

#### 4.1.2 自我中心式解耦

记忆同化允许每个低层单元同时为多个高层抽象作出贡献。CAM 通过一种自我中心式解耦策略实现这种灵活性：依据每个节点的局部结构对其进行复制，从而分离该节点的不同贡献。

对于 $G_0$ 中的每个节点 $v\in V$，CAM 首先提取其自我网络 $G_0[N(v)]$，即由 $v$ 的邻居节点及这些邻居之间的边组成的子图 [39, 40]。[^1] 随后，CAM 从这一局部视角出发，把 $G_0[N(v)]$ 划分为连通分量 $\{C_v^1,C_v^2,\ldots,C_v^{t_v}\}$，其中 $t_v$ 表示 $v$ 的自我网络中连通分量的数量。这些分量反映节点 $v$ 的多重角色：它充当局部割点，连接原本彼此断开的上下文。为显式建模这些角色，系统创建 $v$ 的 $t_v$ 个副本，记为 $\{v^1,v^2,\ldots,v^{t_v}\}$；每个副本只与一个连通分量相关联。

设 $\widetilde V$ 为 $G_0$ 中所有节点副本构成的集合，每条原始边也会映射为其端点副本之间的连接：若 $(u,v)\in E$、$u\in C_v^j$ 且 $v\in C_u^i$，则向 $\widetilde E$ 添加边 $(u^i,v^j)$。由此得到的网络 $\widetilde G_0=(\widetilde V,\widetilde E)$ 通过节点复制有效解耦了重叠结构 [40]，因此即使使用简单的非重叠聚类算法，也能把信息汇总进多个抽象中。

[^1]: 此处 $v$ 的自我网络不包含节点 $v$ 本身 [39]。

值得注意的是，这一解耦过程天然适用于动态设置。当一组新节点 $V_{\mathrm{new}}$ 及相应边 $E_{\mathrm{new}}$ 被整合进 $G_0$（如第 4.1.1 节所述）后，CAM 只需构建或更新受影响节点

$$
A=V_{\mathrm{new}}\cup\{u\in V\mid \exists v\in V_{\mathrm{new}},(u,v)\in E_{\mathrm{new}}\}
$$

的副本，无需完整重建。此外，由于所有节点都只关注各自的自我网络，多个节点的复制过程本身即可并行执行。

#### 4.1.3 在线聚类更新

随后，CAM 在副本网络 $\widetilde G_0$ 上采用增量式标签传播算法，执行在线聚类更新。该过程只在 $\widetilde G_0$ 的一个局部子图上运行，因而能够高效适应新接收到的信息。

具体而言，当一组副本被同化进 $\widetilde G_0$（如第 4.1.2 节所述）时，CAM 首先追踪所有受影响节点 $\widetilde A$，其中包括新添加的副本，以及邻域发生变化的已有副本。新副本节点以各不相同的簇标签初始化，已有节点则保留先前标签。随后，CAM 在 $\widetilde A$ 的诱导子图内执行标签传播；每个节点 $v\in\widetilde A$ 的标签，按照其邻居 $N(v)$ 中占多数的标签迭代更新。若某个已有节点的簇标签发生变化，则将它的所有邻居加入下一轮更新的 $\widetilde A$，确保潜在的连锁影响得到完整捕捉。该过程递归持续，直至不再有标签变化，或达到预定义迭代上限。我们采用该算法是因为其可扩展性 [41]；也可依据具体需求，在此步骤中使用其他增量策略 [42]。

聚类过程收敛后，CAM 汇聚每个已修改簇中的节点，更新下一记忆层中对应的抽象节点。该汇聚过程借助基于 LLM 的文本摘要，把紧密连接的文本块转换为紧凑、连贯的摘要。所得超节点及簇间连接随后作为新节点和新边加入更高层记忆网络，触发记忆层级的进一步发展。

### 4.2 记忆检索

记忆结构构建完成后，阅读智能体面临的另一个关键问题，是如何响应用户查询，从记忆中检索相关信息。现有方法通常采用两类常见策略：层级遍历 [14, 35] 和全局检索 [14, 18, 30]（见附录 B）。然而，二者均存在固有局限。层级遍历只把输入查询与每个记忆层的一部分节点进行比较，因此容易漏掉相关信息，尤其是较低层的信息 [14]。全局检索虽然可以比较所有节点，却仅依赖一次性语义匹配，忽略了彼此关联的记忆结构。

本文为 CAM 配备了一种联想式检索策略：先从全局视角快速定位与查询相关的线索，再沿记忆结构递归联想。具体而言，该策略分为两个阶段：

1. **快速定位：** 对于输入查询 $q$，CAM 首先计算查询嵌入 $f_{\mathrm{emb}}(q)$ 与每个记忆节点嵌入 $f_{\mathrm{emb}}(v)$ 之间的余弦相似度。选择相似度最高的 top-$s$ 个节点，形成候选集 $D$。这一阶段对应于人类在回忆过程中快速锁定相关记忆片段的认知过程。
2. **联想探索：** 随后，CAM 使用 LLM 从 $D$ 中选择有助于回答查询的节点，形成激活集 $P\subseteq D$。接着，将所有已激活节点的同层邻居与低层子节点汇入新的候选集；LLM 继续从中选取可能有用的节点，以扩展激活集 $P$。重复此过程，直至 $P$ 不再增长，或达到最大迭代次数。最后，把所有已激活节点输入 LLM 进行推理，其方式类似于人类的联想思维。

这一剪枝-生长流程结合全局语义匹配和局部结构探索，使 LLM 能够自适应汇集与查询相关的记忆，为有上下文依据的推理提供支持。

## 5 实验

为全面验证 CAM 框架的有效性，我们在一系列单文档与多文档阅读理解任务上评估其性能，包括问答、基于查询的摘要和主张验证。在主实验（第 5.2 节）中，我们采用离线设置，使 CAM 能够一次性访问全部文本来构建记忆。随后，我们在批量在线设置下展示 CAM 的动态性（第 5.3 节）：CAM 分批访问文本。关于不同配置和变体的更多分析见第 5.4 节。

### 5.1 实验设置

**数据集。** 在单文档场景中，我们使用 NovelQA（问答）[43]、QMSum（摘要）[44] 和 FABLES（主张验证）[45]。在多文档场景中，我们在 MultiHop-RAG（问答）[46]、ODSum-Story（摘要）[47] 和 ODSum-Meeting（摘要）[47] 上进行评估。选择这些数据集，是因为它们质量高、类型多样且具有长文本特征。更多细节和统计数据见附录 C。我们也在更经典的数据集上展示了 CAM 的有效性；受篇幅限制，详情见附录 G。

**基线。** 我们将 CAM 与两类基线比较：（1）非结构化记忆，包括 FullContext（直接输入完整文档，超出限制时截断的朴素基线）、MemGPT [11] 和 ReadAgent [13]；（2）结构化基线，包括 RAPTOR [14]、GraphRAG [15]、HippoRAG [16] 和 MemTree [18]。这些基线的更多说明见附录 D。

**指标。** 考虑到 NovelQA、QMSum 和 ODSum 数据集的参考回答均为自由形式，我们报告两项广泛使用的指标 [13, 14, 18]：用于衡量词汇相似度的 ROUGE F 值 [48]，以及由 GPT-4o [49] 充当 LLM 评审所得的准确率（ACC-L）。对于参考答案明确且简短的 MultiHop-RAG（MH-RAG），我们报告精确匹配（EM）和 F1 分数。对于 FABLES，我们依照其原论文 [45]，分别报告正标签和负标签的 F1 分数，即 $\mathrm{F1_P}$ 与 $\mathrm{F1_N}$。

**实现细节。** 除非另有说明，我们以 GPT-4o-mini 作为 LLM 骨干，并以 `text-embedding-3-small` 作为嵌入模型 $f_{\mathrm{emb}}$。为公平比较，我们统一所有方法使用的 LLM 和嵌入模型，确保观察到的性能差异源自记忆设计，而不是 LLM 或嵌入模型差异。项目代码见 <https://github.com/rui9812/CAM>。更多实现细节见附录 E。

**表 2：阅读理解结果。** ACC-L 是 LLM 评审给出的准确率；R-1 和 R-L 是 ROUGE F 值。粗体表示最佳结果，下划线表示次佳结果。FullContext 天然不适用于多文档场景。“-”表示不适用或未报告。

| 方法 | NovelQA R-1 | NovelQA R-L | NovelQA ACC-L | QMSum R-1 | QMSum R-L | QMSum ACC-L | FABLES $\mathrm{F1_P}$ | FABLES $\mathrm{F1_N}$ | MH-RAG EM | MH-RAG F1 | ODSum-Story R-1 | ODSum-Story R-L | ODSum-Story ACC-L | ODSum-Meeting R-1 | ODSum-Meeting R-L | ODSum-Meeting ACC-L |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| FullContext | 19.5 | 17.2 | 38.2 | 26.3 | 17.5 | 15.2 | 79.5 | 39.1 | - | - | - | - | - | - | - | - |
| MemGPT [11] | 21.3 | 19.5 | 40.8 | 31.4 | 20.2 | 40.3 | 81.3 | 38.5 | 63.2 | 67.1 | 32.6 | 18.4 | 36.8 | 28.5 | 16.5 | 33.2 |
| ReadAgent [13] | 22.1 | 20.4 | 42.3 | 32.9 | 23.6 | 45.5 | 84.2 | 43.6 | 60.8 | 65.5 | 33.5 | 19.0 | 39.4 | 28.8 | 16.3 | 35.7 |
| RAPTOR [14] | <u>26.5</u> | <u>23.7</u> | <u>47.8</u> | 34.2 | 24.5 | 50.7 | 86.5 | <u>48.5</u> | <u>69.4</u> | <u>73.6</u> | <u>37.4</u> | 24.0 | 48.7 | 32.6 | 19.9 | 44.3 |
| GraphRAG [15] | 24.8 | 22.5 | 45.3 | <u>35.8</u> | <u>25.2</u> | <u>53.9</u> | <u>87.2</u> | 48.0 | 65.2 | 70.3 | 37.2 | <u>24.3</u> | <u>50.2</u> | <u>33.7</u> | <u>20.5</u> | <u>45.8</u> |
| HippoRAG [16] | 25.5 | 23.0 | 46.5 | 31.9 | 20.8 | 41.2 | 84.7 | 45.8 | 67.9 | 72.5 | 34.5 | 22.8 | 42.9 | 30.2 | 18.3 | 41.2 |
| MemTree [18] | 23.4 | 21.2 | 43.5 | 33.7 | 22.9 | 48.3 | 83.1 | 46.6 | 66.5 | 71.2 | 35.2 | 23.2 | 46.0 | 31.8 | 19.5 | 42.5 |
| **CAM** | **28.8** | **25.4** | **52.3** | **37.2** | **26.5** | **57.6** | **91.5** | **52.5** | **72.8** | **77.5** | **39.2** | **25.5** | **54.6** | **35.9** | **23.2** | **50.7** |
| $\Delta$（%） | +2.3 | +1.7 | +4.5 | +1.4 | +1.3 | +3.7 | +4.3 | +4.0 | +3.4 | +3.9 | +1.8 | +1.2 | +4.4 | +2.2 | +2.7 | +4.9 |

### 5.2 离线性能比较

表 2 给出了六个阅读理解基准上的主要结果。可以看到，CAM 在这些数据集的所有指标上始终优于基线。更深入地看，这些结果在结构性和灵活性两方面都与建构主义设计原则高度一致。

**记忆结构性。** 结构化记忆方法（如 RAPTOR、GraphRAG 和 CAM）明显优于没有显式结构的方法（如 ReadAgent 和 MemGPT），证实了记忆结构性在阅读理解任务中的重要性。层级结构对摘要数据集（QMSum 和 ODSum）尤其有效；RAPTOR、GraphRAG 和 MemTree 均优于非层级的 HippoRAG，便是佐证。值得注意的是，与 RAPTOR 相比，GraphRAG 和 HippoRAG 中由 LLM 驱动的知识图谱建模并未带来稳定收益；可能的原因是，从以叙事为中心的文本中抽取有信息量的实体非常困难。更多分析见第 5.4 节和附录 G。

**同化灵活性。** RAPTOR 和 MemTree 都使用类树记忆结构及全局检索，但 RAPTOR 在所有数据集上始终胜过 MemTree。二者所有指标平均相差 2.1%，这一差距直接源于同化灵活性的不同：RAPTOR 允许每个节点参与多个摘要，而 MemTree 强制执行严格的层级包含关系。这一比较证实，灵活同化对有效理解长文本至关重要。

**记忆检索。** 我们还注意到，RAPTOR 和 GraphRAG 在不同任务上的性能存在差异，这凸显了记忆检索策略对整体性能的影响。RAPTOR 的全局检索适合问答；GraphRAG 则整合所有记忆节点来生成回答，因此在全面摘要任务上更具优势。

**CAM 的整体优势。** CAM 遵循建构主义设计原则，兼具结构性和灵活性，并采用自适应剪枝-生长策略探索记忆层级。因此，CAM 不仅适用于问答任务，在摘要任务上也表现出显著优势。具体而言，CAM 在所有数据集上都取得最佳表现；相较最强基线（RAPTOR 和 GraphRAG），各项指标平均提升 3.0%。下一节还将重点介绍 CAM 的另一项独特优势：它适用于批量在线设置。

### 5.3 CAM 的动态性：从离线走向在线

真实应用场景（如阅读连载小说或追踪实时新闻）通常要求记忆系统在维持稳定性的同时，不断分批整合新文本。现有离线方法（如 RAPTOR 和 GraphRAG）每次更新都必然需要完整重建记忆。MemTree 虽然支持在线处理，却只允许逐块、顺序地整合。与之不同，CAM 天然支持批量同化与局部优先顺应，因而同时保证在线记忆发展的处理效率和推理稳定性。

**处理效率。** 图 3(a) 展示了不同记忆方法整合一批新文本块（每块含 512 个 token）的平均用时。离线 RAPTOR 和 GraphRAG 需要一个多小时才能重建完整记忆，因而不适合实时使用。MemTree 由于顺序处理，其耗时随批量大小线性增长；当新批次超过 400 个文本块时，它甚至比离线重建更慢。相比之下，CAM 借助可并行同化和局部顺应，在各种批量大小下都能维持高效整合。随着批量增大，CAM 的时间开销呈次线性增长，逐渐接近其离线水平（比 RAPTOR 和 GraphRAG 快 4 倍），其中 LLM 操作是主要开销。出现这种时间收敛是合理的，因为大批次本就需要大量记忆调整。

**推理稳定性。** 图 3(b) 报告了 CAM 在 NovelQA、ODSum-Story 和 ODSum-Meeting 上、采用不同批量大小时的 ACC-L 结果。[^2] 在线 CAM 在不同批量大小下的性能相对稳定，且相较表 2 中的基线仍具有竞争力。这说明 CAM 能够有效调整记忆结构，以顺应新文本。

[^2]: 之所以采用这些数据集，是因为它们包含极长文档（附录 C），并使用相同的评估指标。

总而言之，CAM 为在线场景中的长上下文阅读理解提供了一种高效、可靠的解决方案。所设计的同化与顺应过程有效满足了处理效率和推理稳定性这两项关键需求，使 CAM 成为第一个同时具备结构性、灵活性与动态性的智能体记忆框架。

**图 3：在线设置下，CAM 在插入效率和性能稳定性方面的优势。** （a）平均插入时间与在线批量大小的关系；比较 GraphRAG（离线）、RAPTOR（离线）、CAM（离线）、MemTree（在线）和 CAM（在线）。（b）ACC-L 性能与在线批量大小的关系；比较 RAPTOR 平均值、MemTree 平均值，以及 CAM 在 NovelQA、ODSum-Story 和 ODSum-Meeting 上的结果。

### 5.4 更多分析：配置与变体

**表 3：不同配置（第 2–5 行）与变体（第 6–10 行）下的 CAM 剖析。** 第一行为默认配置：GPT-4o-mini 与 `text-embedding-3-small`。

| 类别 | 配置或变体 | NovelQA R-L | NovelQA ACC-L | QMSum R-L | QMSum ACC-L | MH-RAG EM | MH-RAG F1 | ODSum-S R-L | ODSum-S ACC-L |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 默认 | CAM（GPT-4o-mini + `text-embedding-3-small`） | 25.4 | 52.3 | 26.5 | 57.6 | 72.8 | 77.5 | 25.5 | 54.6 |
| LLM 骨干 | Llama-3.1-8B-Instruct | 23.7 | 49.5 | 25.3 | 54.4 | 70.3 | 74.2 | 24.3 | 52.5 |
| LLM 骨干 | Qwen2.5-7B-Instruct | 24.1 | 50.6 | 25.5 | 55.0 | 70.9 | 75.6 | 24.2 | 53.2 |
| 嵌入模型 | `text-embedding-3-large` | 25.8 | 53.0 | 27.4 | 59.2 | 73.2 | 78.0 | 26.7 | 56.4 |
| 嵌入模型 | E5-Mistral-7B-Instruct | 26.4 | 54.3 | 27.1 | 58.7 | 73.5 | 78.4 | 26.3 | 55.7 |
| 检索策略 | 层级遍历 | 23.2 | 46.5 | 24.8 | 55.2 | 68.3 | 72.5 | 23.8 | 49.5 |
| 检索策略 | 全局检索 | 24.5 | 50.3 | 25.7 | 56.3 | 70.5 | 75.9 | 24.4 | 50.8 |
| 变体 | 使用细粒度建模 | 25.8 | 53.5 | 26.3 | 57.2 | 72.0 | 76.8 | 25.1 | 54.8 |
| 变体 | 不使用层级结构 | 22.9 | 46.7 | 23.2 | 46.6 | 68.5 | 73.4 | 23.5 | 49.3 |
| 变体 | 不使用灵活性 | 24.2 | 50.3 | 24.7 | 51.5 | 70.3 | 75.1 | 24.7 | 51.2 |

**LLM 骨干。** 为评估 CAM 的性能是否依赖 GPT-4o-mini 等商业 LLM，我们将其替换为两种流行的开源 LLM：Llama-3.1-8B-Instruct 和 Qwen2.5-7B-Instruct。如表 3 第 2–3 行所示，尽管参数规模相对较小，基于这两种 LLM 的 CAM 实现仍表现强劲，相较 GPT-4o-mini 版本（第 1 行）仅略有下降。这表明，CAM 的核心优势主要来自有效的记忆设计，而非对昂贵 LLM 骨干的依赖。

**嵌入模型。** 我们进一步研究嵌入模型对 CAM 性能的影响。除默认的 `text-embedding-3-small` 外，我们还采用 MTEB 排行榜 [50] 上更强的两种嵌入模型：`text-embedding-3-large` 和 E5-Mistral-7B-Instruct。表 3 第 4–5 行表明，更先进的嵌入模型能够提升 CAM 的性能，因为它们可为记忆基础网络的构建提供更准确的相似度测量。

**检索策略。** 表 3 第 6–7 行给出了 CAM 使用不同记忆检索策略的结果。我们发现，无论使用层级遍历还是全局检索，性能都会下降，由此验证了剪枝-生长策略的有效性。值得注意的是，CAM 的核心是记忆发展过程，因此天然兼容多种检索策略；在现实应用中，用户可以依据具体需求选择合适方案 [51]。

**按问题类型划分的性能。** 为进一步研究 CAM 面对不同问题特征时的表现，我们利用 NovelQA 提供的问题类型标注，开展深入的性能分析。具体而言，问题沿两个相互正交的维度分类，即基于复杂度与基于语义侧面的类别，分别反映推理复杂度和语义焦点。详细结果见附录 F。我们观察到，CAM 在处理复杂问题（如多跳、时间和跨度类型）时优势明显；这类问题通常要求整合语义上相距遥远的证据，并维持叙事连贯性。

**细粒度基础建模。** 近期若干工作（如 GraphRAG 和 HippoRAG）使用 LLM 从输入文本构建知识图谱，即执行实体识别和关系抽取。这种细粒度知识建模很容易集成进 CAM，用于构建更细致的基础网络。然而，由于需要大量调用 LLM，该过程会让计算成本增加到三倍以上；如表 3 第 8 行所示，它却未带来与成本相称的提升。我们观察到，即使是商业 LLM（GPT-4o-mini），也难以从长篇叙事中抽取有信息量的实体。更多分析见附录 G。

**消融变体。** 表 3 第 9–10 行给出了 CAM 两种消融变体的性能。一种完全去除层级聚类，只依靠基础语义网络进行推理；另一种绕过自我中心式解耦，直接应用标签传播进行层级聚类。可以看到，两种变体都会造成明显的性能下降，从而证实了设计原则中层级结构与灵活性的重要性。

## 6 局限性与讨论

**超越阅读理解。** 虽然建构主义理论为认知发展提供了广泛洞见，但本研究专门面向长文本阅读理解任务，如问答、基于查询的摘要和主张验证。如何把这套深刻的记忆设计原则扩展到行为规划、长序列生成和多模态任务等其他领域，仍未得到探索，但极具未来研究潜力。

**更多智能体行为。** 本研究主要致力于界定智能体记忆的关键特征，并开发原型来验证这些特征的重要性。然而，我们的框架尚未纳入自我提问、反思等其他智能体行为。整合这些能力，可能是推动基于 LLM 的智能体系统变得更加稳健的关键。

**幻觉传播。** CAM 在记忆发展过程中使用 LLM 生成摘要，因此存在幻觉风险，即生成不准确或虚构的信息。CAM 会生成层级摘要，低层节点中的错误或幻觉可能传播至更高层抽象，从而影响该框架在现实场景中的适用性。如何检测并缓解智能体记忆内部的此类幻觉，仍是一个开放挑战。

**不一致的信息来源。** 检测和调和矛盾信息，是人类认知的一项关键能力。与先前基于 LLM 的记忆系统（如 GraphRAG、RAPTOR 和 MemTree）一致，我们的框架同样假设源文本内部一致。然而，现实文档往往包含相互冲突的事实或观点，在复杂的开放域场景中尤其如此。应对这一挑战需要专门投入，例如构建新基准，并设计评估协议来衡量信息调和能力。我们把感知不一致性的记忆发展视为一个很有前景的未来研究方向。

**替代实现。** CAM 使用局部优先的增量式重叠聚类算法，具体实现建构主义设计原则。虽然这一实现具备目标属性，即结构化图式、灵活同化和动态顺应，但它并非实现这些目标的唯一路径。其他策略（如神经控制器和符号规划器）可能在可扩展性、可解释性和泛化性方面提供不同权衡。探索遵循建构主义原则的替代实现，可进一步丰富记忆系统的技术版图。

**学会记忆。** CAM 的实现依赖固定提示词和经调节的超参数，而非用于记忆同化与顺应的自适应策略。它既不会优化记忆结构，也不会根据下游反馈调整更新规则。转向可训练的记忆控制器（如学习应更新什么，或如何路由检索），可以进一步提升记忆发展的质量与效率。设计此类机制会带来并不简单的信用分配难题，但这是迈向更强大智能体记忆系统的关键一步。

## 7 结论

本文聚焦一个正在兴起的问题：如何为 LLM 设计记忆模块，以构建能够理解极长文档的阅读智能体。为此，我们借鉴 Jean Piaget 的建构主义理论，提出有效的记忆模块应当具备结构化图式、灵活同化和动态顺应。在这一蓝图基础上，我们开发了建构主义智能体记忆原型 CAM；它在广泛的长文本阅读理解基准上，同时展现出性能和效率优势。

我们希望，这一方案能够启发研究者更广泛地探索认知记忆架构，构建更强大的 LLM 智能体。此外，我们相信，建构主义设计原则也为把智能体记忆推进到文本理解之外提供了一项有说服力的基础，并有望应用于规划、推理和多模态领域。

## 致谢与经费披露

本工作得到以下项目的部分支持：中国国家自然科学基金（编号 62422215、62472427）；中国人民大学“双一流”重大创新规划交叉平台；中国人民大学公共计算云；中国人民大学世界一流大学（学科）建设经费；中国人民大学 2024 年拔尖创新人才培育资助计划；华为创新研究计划。我们感谢 MindSpore、CANN（神经网络计算架构）以及本研究所使用的昇腾 AI 处理器提供支持。

MindSpore：<https://www.mindspore.cn>

## 参考文献

> 为保证作者、题名、刊物及检索信息准确，参考文献按原文保留，不翻译论文题名。

[1] Tom Brown, Benjamin Mann, Nick Ryder, Melanie Subbiah, Jared D Kaplan, Prafulla Dhariwal, Arvind Neelakantan, Pranav Shyam, Girish Sastry, Amanda Askell, Sandhini Agarwal, Ariel Herbert-Voss, Gretchen Krueger, Tom Henighan, Rewon Child, Aditya Ramesh, Daniel Ziegler, Jeffrey Wu, Clemens Winter, Chris Hesse, Mark Chen, Eric Sigler, Mateusz Litwin, Scott Gray, Benjamin Chess, Jack Clark, Christopher Berner, Sam McCandlish, Alec Radford, Ilya Sutskever, and Dario Amodei. Language models are few-shot learners. In *Advances in Neural Information Processing Systems*, volume 33, pages 1877–1901, 2020.

[2] Aakanksha Chowdhery, Sharan Narang, Jacob Devlin, Maarten Bosma, Gaurav Mishra, Adam Roberts, Paul Barham, Hyung Won Chung, Charles Sutton, Sebastian Gehrmann, Parker Schuh, Kensen Shi, Sasha Tsvyashchenko, Joshua Maynez, Abhishek Rao, Parker Barnes, Yi Tay, Noam Shazeer, Vinodkumar Prabhakaran, Emily Reif, Nan Du, Ben Hutchinson, Reiner Pope, James Bradbury, Jacob Austin, Michael Isard, Guy Gur-Ari, Pengcheng Yin, Toju Duke, Anselm Levskaya, Sanjay Ghemawat, Sunipa Dev, Henryk Michalewski, Xavier Garcia, Vedant Misra, Kevin Robinson, Liam Fedus, Denny Zhou, Daphne Ippolito, David Luan, Hyeontaek Lim, Barret Zoph, Alexander Spiridonov, Ryan Sepassi, David Dohan, Shivani Agrawal, Mark Omernick, Andrew M. Dai, Thanumalayan Sankaranarayana Pillai, Marie Pellat, Aitor Lewkowycz, Erica Moreira, Rewon Child, Oleksandr Polozov, Katherine Lee, Zongwei Zhou, Xuezhi Wang, Brennan Saeta, Mark Diaz, Orhan Firat, Michele Catasta, Jason Wei, Kathy Meier-Hellstern, Douglas Eck, Jeff Dean, Slav Petrov, and Noah Fiedel. Palm: Scaling language modeling with pathways. *Journal of Machine Learning Research*, 24:240:1–240:113, 2023.

[3] Hugo Touvron, Louis Martin, Kevin Stone, Peter Albert, Amjad Almahairi, Yasmine Babaei, Nikolay Bashlykov, Soumya Batra, Prajjwal Bhargava, Shruti Bhosale, et al. Llama 2: Open foundation and fine-tuned chat models. arXiv preprint arXiv:2307.09288, 2023.

[4] Abhimanyu Dubey, Abhinav Jauhri, Abhinav Pandey, Abhishek Kadian, Ahmad Al-Dahle, Aiesha Letman, Akhil Mathur, Alan Schelten, Amy Yang, Angela Fan, et al. The llama 3 herd of models. arXiv preprint arXiv:2407.21783, 2024.

[5] Nelson F. Liu, Kevin Lin, John Hewitt, Ashwin Paranjape, Michele Bevilacqua, Fabio Petroni, and Percy Liang. Lost in the middle: How language models use long contexts. *Transactions of the Association for Computational Linguistics*, 12:157–173, 2024.

[6] Freda Shi, Xinyun Chen, Kanishka Misra, Nathan Scales, David Dohan, Ed H. Chi, Nathanael Schärli, and Denny Zhou. Large language models can be easily distracted by irrelevant context. In *International Conference on Machine Learning*, volume 202, pages 31210–31227, 2023.

[7] Tianle Li, Ge Zhang, Quy Duc Do, Xiang Yue, and Wenhu Chen. Long-context llms struggle with long in-context learning. arXiv preprint arXiv:2404.02060, 2024.

[8] Runchu Tian, Yanghao Li, Yuepeng Fu, Siyang Deng, Qinyu Luo, Cheng Qian, Shuo Wang, Xin Cong, Zhong Zhang, Yesai Wu, Yankai Lin, Huadong Wang, and Xiaojiang Liu. Distance between relevant information pieces causes bias in long-context llms. *CoRR*, abs/2410.14641, 2024.

[9] Zeyu Zhang, Xiaohe Bo, Chen Ma, Rui Li, Xu Chen, Quanyu Dai, Jieming Zhu, Zhenhua Dong, and Ji-Rong Wen. A survey on the memory mechanism of large language model based agents. arXiv preprint arXiv:2404.13501, 2024.

[10] Joon Sung Park, Joseph O’Brien, Carrie Jun Cai, Meredith Ringel Morris, Percy Liang, and Michael S Bernstein. Generative agents: Interactive simulacra of human behavior. In *Proceedings of the 36th Annual ACM Symposium on User Interface Software and Technology*, pages 1–22, 2023.

[11] Charles Packer, Vivian Fang, Shishir G Patil, Kevin Lin, Sarah Wooders, and Joseph E Gonzalez. Memgpt: Towards llms as operating systems. arXiv preprint arXiv:2310.08560, 2023.

[12] Wanjun Zhong, Lianghong Guo, Qiqi Gao, He Ye, and Yanlin Wang. Memorybank: Enhancing large language models with long-term memory. In *Thirty-Eighth AAAI Conference on Artificial Intelligence*, pages 19724–19731, 2024.

[13] Kuang-Huei Lee, Xinyun Chen, Hiroki Furuta, John Canny, and Ian Fischer. A human-inspired reading agent with gist memory of very long contexts. In *International Conference on Machine Learning*, pages 26396–26415. PMLR, 2024.

[14] Parth Sarthi, Salman Abdullah, Aditi Tuli, Shubh Khanna, Anna Goldie, and Christopher D Manning. Raptor: Recursive abstractive processing for tree-organized retrieval. In *The Twelfth International Conference on Learning Representations*, 2024.

[15] Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt, and Jonathan Larson. From local to global: A graph rag approach to query-focused summarization. arXiv preprint arXiv:2404.16130, 2024.

[16] Bernal Jimenez Gutierrez, Yiheng Shu, Yu Gu, Michihiro Yasunaga, and Yu Su. HippoRAG: Neurobiologically inspired long-term memory for large language models. In *The Thirty-eighth Annual Conference on Neural Information Processing Systems*, 2024.

[17] Haoyu Han, Yu Wang, Harry Shomer, Kai Guo, Jiayuan Ding, Yongjia Lei, Mahantesh Halappanavar, Ryan A Rossi, Subhabrata Mukherjee, Xianfeng Tang, et al. Retrieval-augmented generation with graphs (graphrag). arXiv preprint arXiv:2501.00309, 2024.

[18] Alireza Rezazadeh, Zichao Li, Wei Wei, and Yujia Bao. From isolated conversations to hierarchical schemas: Dynamic tree memory representation for LLMs. In *The Thirteenth International Conference on Learning Representations*, 2025.

[19] Jean Piaget. *Piaget’s theory*. Springer, 1976.

[20] Jean Piaget. *The psychology of intelligence*. Routledge, 2005.

[21] Sandra Waite-Stupiansky. Jean piaget’s constructivist theory of learning. In *Theories of early childhood education*, pages 3–18. Routledge, 2022.

[22] Jean Piaget. Piaget’s theory of cognitive development. *Childhood cognitive development: The essential readings*, 2(7):33–47, 2000.

[23] Kathleen Stassen Berger. *The developing person through the life span*. ed. New York: Worth, 1994.

[24] Yukang Chen, Shengju Qian, Haotian Tang, Xin Lai, Zhijian Liu, Song Han, and Jiaya Jia. Longlora: Efficient fine-tuning of long-context large language models. arXiv preprint arXiv:2309.12307, 2023.

[25] Iz Beltagy, Matthew E Peters, and Arman Cohan. Longformer: The long-document transformer. arXiv preprint arXiv:2004.05150, 2020.

[26] Manzil Zaheer, Guru Guruganesh, Kumar Avinava Dubey, Joshua Ainslie, Chris Alberti, Santiago Ontanon, Philip Pham, Anirudh Ravula, Qifan Wang, Li Yang, and Amr Ahmed. Big bird: Transformers for longer sequences. In *Advances in Neural Information Processing Systems*, volume 33, pages 17283–17297, 2020.

[27] Mandy Guo, Joshua Ainslie, David C. Uthus, Santiago Ontañón, Jianmo Ni, Yun-Hsuan Sung, and Yinfei Yang. Longt5: Efficient text-to-text transformer for long sequences. In *Findings of the Association for Computational Linguistics: NAACL 2022*, pages 724–736, 2022.

[28] Joshua Ainslie, Tao Lei, Michiel de Jong, Santiago Ontañón, Siddhartha Brahma, Yury Zemlyanskiy, David C. Uthus, Mandy Guo, James Lee-Thorp, Yi Tay, Yun-Hsuan Sung, and Sumit Sanghai. Colt5: Faster long-range transformers with conditional computation. In *Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing*, pages 5085–5100, 2023.

[29] Yi Tay, Mostafa Dehghani, Dara Bahri, and Donald Metzler. Efficient transformers: A survey. *ACM Computing Surveys*, 55(6):1–28, 2022.

[30] Patrick S. H. Lewis, Ethan Perez, Aleksandra Piktus, Fabio Petroni, Vladimir Karpukhin, Naman Goyal, Heinrich Küttler, Mike Lewis, Wen-tau Yih, Tim Rocktäschel, Sebastian Riedel, and Douwe Kiela. Retrieval-augmented generation for knowledge-intensive NLP tasks. In *Advances in Neural Information Processing Systems*, 2020.

[31] Peng Xu, Wei Ping, Xianchao Wu, Lawrence McAfee, Chen Zhu, Zihan Liu, Sandeep Subramanian, Evelina Bakhturina, Mohammad Shoeybi, and Bryan Catanzaro. Retrieval meets long context large language models. arXiv preprint arXiv:2310.03025, 2023.

[32] Yunfan Gao, Yun Xiong, Xinyu Gao, Kangxiang Jia, Jinliu Pan, Yuxi Bi, Yi Dai, Jiawei Sun, and Haofen Wang. Retrieval-augmented generation for large language models: A survey. arXiv preprint arXiv:2312.10997, 2023.

[33] Ali Modarressi, Abdullatif Köksal, Ayyoob Imani, Mohsen Fayyaz, and Hinrich Schütze. Mem-llm: Finetuning llms to use an explicit read-write memory. arXiv preprint arXiv:2404.11672, 2024.

[34] Bing Wang, Xinnian Liang, Jian Yang, Hui Huang, Shuangzhi Wu, Peihao Wu, Lu Lu, Zejun Ma, and Zhoujun Li. Enhancing large language model with self-controlled memory framework. arXiv preprint arXiv:2304.13343, 2023.

[35] Howard Chen, Ramakanth Pasunuru, Jason Weston, and Asli Celikyilmaz. Walking down the memory maze: Beyond context limit through interactive reading. arXiv preprint arXiv:2310.05029, 2023.

[36] Shilong Li, Yancheng He, Hangyu Guo, Xingyuan Bu, Ge Bai, Jie Liu, Jiaheng Liu, Xingwei Qu, Yangguang Li, Wanli Ouyang, et al. Graphreader: Building graph-based agent to enhance long-context abilities of large language models. arXiv preprint arXiv:2406.14550, 2024.

[37] Aditya Krishna Menon, Anand Rajagopalan, Baris Sumengen, Gui Citovsky, Qin Cao, and Sanjiv Kumar. Online hierarchical clustering approximations. arXiv preprint arXiv:1909.09667, 2019.

[38] Javier Borge-Holthoefer and Alex Arenas. Semantic networks: Structure and dynamics. *Entropy*, 12(5):1264–1302, 2010.

[39] Michele Coscia, Giulio Rossetti, Fosca Giannotti, and Dino Pedreschi. DEMON: a local-first discovery method for overlapping communities. In *The 18th ACM SIGKDD International Conference on Knowledge Discovery and Data Mining*, pages 615–623, 2012.

[40] Alessandro Epasto, Silvio Lattanzi, and Renato Paes Leme. Ego-splitting framework: from non-overlapping to overlapping clusters. In *Proceedings of the 23rd ACM SIGKDD International Conference on Knowledge Discovery and Data Mining*, pages 145–154, 2017.

[41] Jierui Xie, Mingming Chen, and Boleslaw K Szymanski. Labelrankt: Incremental community detection in dynamic networks via label propagation. In *Proceedings of the workshop on dynamic networks management and mining*, pages 25–32, 2013.

[42] Rui Xu and Donald Wunsch. Survey of clustering algorithms. *IEEE Transactions on neural networks*, 16(3):645–678, 2005.

[43] Cunxiang Wang, Ruoxi Ning, Boqi Pan, Tonghui Wu, Qipeng Guo, Cheng Deng, Guangsheng Bao, Xiangkun Hu, Zheng Zhang, Qian Wang, and Yue Zhang. NovelQA: Benchmarking question answering on documents exceeding 200k tokens. In *The Thirteenth International Conference on Learning Representations*, 2025.

[44] Ming Zhong, Da Yin, Tao Yu, Ahmad Zaidi, Mutethia Mutuma, Rahul Jha, Ahmed Hassan Awadallah, Asli Celikyilmaz, Yang Liu, Xipeng Qiu, and Dragomir R. Radev. Qmsum: A new benchmark for query-based multi-domain meeting summarization. In *Proceedings of the 2021 Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies*, pages 5905–5921, 2021.

[45] Yekyung Kim, Yapei Chang, Marzena Karpinska, Aparna Garimella, Varun Manjunatha, Kyle Lo, Tanya Goyal, and Mohit Iyyer. FABLES: Evaluating faithfulness and content selection in book-length summarization. In *First Conference on Language Modeling*, 2024.

[46] Yixuan Tang and Yi Yang. Multihop-rag: Benchmarking retrieval-augmented generation for multi-hop queries. arXiv preprint arXiv:2401.15391, 2024.

[47] Yijie Zhou, Kejian Shi, Wencai Zhang, Yixin Liu, Yilun Zhao, and Arman Cohan. Odsum: New benchmarks for open domain multi-document summarization. arXiv preprint arXiv:2309.08960, 2023.

[48] Chin-Yew Lin. Rouge: A package for automatic evaluation of summaries. In *Text summarization branches out*, pages 74–81, 2004.

[49] Jiawei Gu, Xuhui Jiang, Zhichao Shi, Hexiang Tan, Xuehao Zhai, Chengjin Xu, Wei Li, Yinghan Shen, Shengjie Ma, Honghao Liu, et al. A survey on llm-as-a-judge. arXiv preprint arXiv:2411.15594, 2024.

[50] Kenneth Enevoldsen, Isaac Chung, Imene Kerboua, Márton Kardos, Ashwin Mathur, David Stap, Jay Gala, Wissam Siblini, Dominik Krzemiński, Genta Indra Winata, Saba Sturua, Saiteja Utpala, Mathieu Ciancone, Marion Schaeffer, Gabriel Sequeira, Diganta Misra, Shreeya Dhakal, Jonathan Rystrøm, Roman Solomatin, Ömer Çağatan, Akash Kundu, Martin Bernstorff, Shitao Xiao, Akshita Sukhlecha, Bhavish Pahwa, Rafał Poświata, Kranthi Kiran GV, Shawon Ashraf, Daniel Auras, Björn Plüster, Jan Philipp Harries, Loïc Magne, Isabelle Mohr, Mariya Hendriksen, Dawei Zhu, Hippolyte Gisserot-Boukhlef, Tom Aarsen, Jan Kostkan, Konrad Wojtasik, Taemin Lee, Marek Šuppa, Crystina Zhang, Roberta Rocca, Mohammed Hamdy, Andrianos Michail, John Yang, Manuel Faysse, Aleksei Vatolin, Nandan Thakur, Manan Dey, Dipam Vasani, Pranjal Chitale, Simone Tedeschi, Nguyen Tai, Artem Snegirev, Michael Günther, Mengzhou Xia, Weijia Shi, Xing Han Lù, Jordan Clive, Gayatri Krishnakumar, Anna Maksimova, Silvan Wehrli, Maria Tikhonova, Henil Panchal, Aleksandr Abramov, Malte Ostendorff, Zheng Liu, Simon Clematide, Lester James Miranda, Alena Fenogenova, Guangyu Song, Ruqiya Bin Safi, Wen-Ding Li, Alessia Borghini, Federico Cassano, Hongjin Su, Jimmy Lin, Howard Yen, Lasse Hansen, Sara Hooker, Chenghao Xiao, Vaibhav Adlakha, Orion Weller, Siva Reddy, and Niklas Muennighoff. Mmteb: Massive multilingual text embedding benchmark. arXiv preprint arXiv:2502.13595, 2025. doi: 10.48550/arXiv.2502.13595.

[51] Gautier Izacard, Mathilde Caron, Lucas Hosseini, Sebastian Riedel, Piotr Bojanowski, Armand Joulin, and Edouard Grave. Unsupervised dense information retrieval with contrastive learning. *Transactions on Machine Learning Research*, 2022.

[52] Vincent A Traag, Ludo Waltman, and Nees Jan Van Eck. From louvain to leiden: guaranteeing well-connected communities. *Scientific reports*, 9(1):1–12, 2019.

[53] Ruosen Li and Xinya Du. Leveraging structured information for explainable multi-hop question answering and reasoning. In *Findings of the Association for Computational Linguistics: EMNLP*, pages 6779–6789, 2023.

[54] Yanming Liu, Xinyue Peng, Tianyu Du, Jianwei Yin, Weihao Liu, and Xuhong Zhang. Era-cot: Improving chain-of-thought through entity relationship analysis. In *Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)*, pages 8780–8794, 2024.

[55] Jinyoung Park, Ameen Patel, Omar Zia Khan, Hyunwoo J Kim, and Joo-Kyung Kim. Graph-guided reasoning for multi-hop question answering in large language models. arXiv e-prints arXiv:2311.09762, 2023.

[56] Xiaoxia Cheng, Zeqi Tan, and Weiming Lu. Information re-organization improves reasoning in large language models. arXiv preprint arXiv:2404.13985, 2024.

[57] Xin Su, Tiep Le, Steven Bethard, and Phillip Howard. Semi-structured chain-of-thought: Integrating multiple sources of knowledge for improved language model reasoning. arXiv preprint arXiv:2311.08505, 2023.

[58] Jianing Wang, Qiushi Sun, Xiang Li, and Ming Gao. Boosting language models reasoning with chain-of-knowledge prompting. arXiv preprint arXiv:2306.06427, 2023.

[59] Zhilin Yang, Peng Qi, Saizheng Zhang, Yoshua Bengio, William W. Cohen, Ruslan Salakhutdinov, and Christopher D. Manning. Hotpotqa: A dataset for diverse, explainable multi-hop question answering. In *Proceedings of the 2018 Conference on Empirical Methods in Natural Language Processing*, pages 2369–2380, 2018.

[60] Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, and Ashish Sabharwal. Musique: Multihop questions via single-hop question composition. *Transactions of the Association for Computational Linguistics*, 10:539–554, 2022.

[61] Xanh Ho, Anh-Khoa Duong Nguyen, Saku Sugawara, and Akiko Aizawa. Constructing A multi-hop QA dataset for comprehensive evaluation of reasoning steps. In *Proceedings of the 28th International Conference on Computational Linguistics*, pages 6609–6625, 2020.

[62] Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, and Ashish Sabharwal. Interleaving retrieval with chain-of-thought reasoning for knowledge-intensive multi-step questions. In *Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)*, pages 10014–10037, 2023.

## 附录 A Jean Piaget 建构主义理论概述

Jean Piaget 的建构主义理论 [19–21] 最初用于解释儿童的认知发展，也为理解读者如何领会复杂文本提供了一套有价值的框架。该理论强调，阅读理解并不只是被动接收信息，而是涉及主动的建构过程。这一过程的核心是图式、同化和顺应；三者协同推动认知发展。

**图式。** 图式是组织和解释信息的认知结构或心理框架，是认知活动的基本构件。在阅读理解中，这种分层组织让读者能够以不同粒度处理信息，从基本的词语识别到更高阶的概念分析。例如，儿童对“动物”的基础图式可能包括“会移动”或“会进食”等宽泛特征，之后再分化成有关“狗”或“鸟”的更具体图式。这些层级结构提供稳定性，让人能够高效处理熟悉的经验；其分层组织则支持整合日益抽象和复杂的知识。

**同化。** 同化是以最小调整，把新经验整合进已有图式的过程。它体现了认知灵活性，使个体能够把熟悉框架应用于新情境。例如，儿童遇到一种新的犬种时，可能会依据毛发或吠叫等共同特征，把它同化进自己的“狗”图式。同化对学习至关重要，因为它以图式的层级结构为基础，在不破坏认知组织的情况下强化并扩展已有知识。然而，当新经验挑战这些框架时，还需要一项互补过程。

**顺应。** 顺应是修改已有图式或创建新图式，以解释无法纳入当前框架的经验。该动态过程通过重构图式的层级组织以使其符合现实，从而推动认知成长。例如，如果儿童把猫误认为狗，他可能会创建新的“猫”图式，或细化“狗”图式以排除猫。顺应对于解决认知冲突至关重要，它让个体能够在层级系统中发展出更准确、更复杂的图式。

**同化与顺应的相互作用。** Piaget 的平衡化概念描述了同化与顺应之间的动态互动；这一互动维持认知平衡。当新经验造成失衡时，个体通过同化纳入相容信息，并通过顺应调整或扩展其层级图式。该循环推动个体经历 Piaget 所提出的各个发展阶段（感知运动阶段、前运算阶段、具体运算阶段和形式运算阶段），使图式不断分化，并变得更有组织。

总而言之，Piaget 的建构主义理论强调：图式的层级组织是知识的基础，而灵活同化与动态顺应则使适应和成长成为可能。这些过程协同细化并扩展认知层级，使 Piaget 理论成为理解记忆发展的有力视角。

## 附录 B 相关工作讨论

表 1 从建构主义视角概括了当前代表性记忆工作的特征。本节详细介绍这些方法。

**表格式记忆。** MemGPT [11] 和 ReadAgent [13] 等非结构化方法，把单个文本块分别存储或压缩进表格式记忆。由于设计简单，这些方法天然支持批量处理新文本。然而，它们忽视了记忆结构性的重要意义，因此也无从讨论灵活性和动态性。

**RAPTOR [14]。** 该方法认识到高层摘要的重要性，采用自底向上的聚类与摘要策略形成层级记忆，与结构化图式的概念一致。RAPTOR 使用基于高斯混合模型的软聚类：节点可属于多个簇，也无需预先固定簇数，因此满足记忆同化的灵活性。然而，该方法完全以离线方式运行，无法增量更新记忆，即不具备动态顺应。此外，其聚类方法依赖高斯假设，而这一假设可能并不完全符合文本数据的性质，进而影响它在不同文本类型上的适用性。

**GraphRAG [15] 与 HippoRAG [16]。** GraphRAG 利用 LLM 的语言能力，从每个文本块中抽取细粒度知识三元组并构建知识图谱。随后，它采用 Leiden 算法 [52] 进行社区检测，并使用 LLM 生成摘要。该方法满足结构性，却缺乏灵活性，因为 Leiden 算法会生成互不相交的社区结构，使每个低层节点只能属于一个高层节点。此外，GraphRAG 以离线方式运行；新文本到来时，必须重建整个记忆。HippoRAG 同样会从长文档构建知识图谱记忆，却忽略了高层语义摘要的重要性。实验中我们观察到，LLM 仍难以完成此类细粒度知识建模，主要原因是很难在长文本范围内维持实体与关系的一致性。附录 G 的进一步分析表明，这种知识建模更适合事实型问答，而非叙事型问答。

**MemTree [18]。** MemTree 是首个考虑在线场景的方法，它使用自顶向下的层级聚类算法发展记忆。然而，它以逐块方式处理新输入文本，每次只能向记忆结构插入一个文本块。处理大量文本块时，这种设计效率低下，如图 3(a) 所示。MemTree 只把每个文本块分配到一个位置，因此缺乏灵活性。此外，它不会为了顺应每次新插入而调整记忆结构，可能导致记忆结构失衡，并对文本块的插入顺序敏感。

**其他相关工作。** 还有一些研究 [53–56] 使用结构化方法增强 LLM。SG-Prompt [53] 首先通过从全部已检索文本中抽取信息，构建语义图结构，再使用这种符号信息增强 LLM 的推理质量。GE-Reasoning [55] 和 Semi-Structured CoT [57] 重点把输入问题解析为带掩码的结构化链，再依据预定义知识图谱或纯文本数据库补全每个不完整的知识三元组。CoK [58] 从预定义知识图谱中检索知识三元组，再将其与人工标注组合，旨在构建合理的示例，以激发 LLM 的知识生成能力。上述方法大多可视为 GraphRAG 的变体；考虑到评估预算，我们未将其纳入比较。

综上，没有任何一种上述方法能够同时满足建构主义理论关于结构性、灵活性和动态性的要求。CAM 框架有效填补了这一空白，为批量在线场景中的长上下文阅读理解提供了一种高效、可靠的解决方案。

## 附录 C 数据集统计

我们在一系列单文档和多文档阅读理解任务上评估 CAM 的性能，任务包括问答、基于查询的摘要和主张验证。

在单文档场景中，我们使用：（1）NovelQA [43]，一个长篇小说问答数据集，每部小说平均超过 20 万个 token；（2）QMSum [44]，一个基于查询的多领域会议摘要数据集，每份会议转录平均约 1 万词；（3）FABLES [45]，一个主张验证数据集，要求模型依据长篇文档判断给定主张的忠实性，每份文档平均超过 12 万个 token。

在多文档场景中，我们还使用：（4）MultiHop-RAG [46]，一个近期提出的多跳问答数据集，语料库包含六个类别的 609 篇新闻文章，每篇平均约 2,000 个 token；（5）ODSum [47]，一个多文档、基于查询的摘要数据集，包含 ODSum-Story 和 ODSum-Meeting 两个子集。

对于 FABLES 和 MultiHop-RAG，我们过滤掉不可回答的主张和问题。全部数据集的统计见表 4。

**表 4：六个阅读理解基准的统计数据。** “文档数”一行保留原文的语料规模表达；“Token 数”是平均值或“平均值 × 文档数”。

| 数据集统计 | NovelQA | FABLES | QMSum | MultiHop-RAG | ODSum-Story | ODSum-Meeting |
|---|---:|---:|---:|---:|---:|---:|
| 场景 | 单文档 | 单文档 | 单文档 | 多文档 | 多文档 | 多文档 |
| 任务 | 问答 | 验证 | 摘要 | 问答 | 摘要 | 摘要 |
| 文档数 | 89 | 26 | 232 | 609 | 1,190 | 232 |
| Token 数 | 200K | 121K | 10K | 2,046 × 609 | 809 × 1,190 | 7,176 × 232 |
| 查询数 | 2,305 | 3,158 | 1,808 | 2,255 | 635 | 436 |

## 附录 D 基线说明

我们将 CAM 与一系列基于 LLM 的阅读理解方法比较：（1）FullContext，一项朴素基线，直接把完整文档输入 LLM，并截断任何超出上下文窗口的内容；（2）MemGPT [10, 11]，直接检索与查询相关的文本块，不构造高层表示；（3）ReadAgent [13]，为所有文本块生成摘要，把长文档压缩进 LLM 的上下文窗口；（4）RAPTOR [14]，递归构建具有不同摘要层级的树结构；（5）GraphRAG [15]，从文档中抽取知识图谱，并将其划分为独立社区以形成摘要；（6）HippoRAG [16]，以类似方式组织知识图谱，但缺乏高层抽象；（7）MemTree [18]，以自顶向下的方式依次把文本整合进一棵树，并据此更新摘要。关于这些基线以及它们与本文设计差异的更多说明见附录 B。

## 附录 E 实现细节

**骨干模型。** 实验中，我们统一所有方法使用的 LLM 和嵌入模型，确保观察到的性能差异来自记忆设计，而非 LLM 能力或嵌入差异。默认情况下，我们采用温度为 0 的 GPT-4o-mini 作为 LLM 骨干，并采用 `text-embedding-3-small` 作为嵌入模型 $f_{\mathrm{emb}}$。我们也考察了其他模型配置，以展示本框架的适用性。

**超参数选择。** CAM 框架包含若干控制记忆发展的关键超参数。我们从 $\{0.5,0.7,0.9\}$ 中选择权重系数 $\alpha$，从 $\{1.0,1.5,2.0\}$ 中选择衰减率 $\sigma$，并从 $\{0.5,0.7\}$ 中选择相似度阈值 $\theta$。依据数据集特征，文本块大小设为 256 或 512 个 token。记忆扩展时设 $k=10$，记忆检索时设 $s=5$。LLM 温度设为 0。需要注意的是，对于不属于同一文档的文本块，式（2）中的位置邻近相似度设为 0。所有 LLM 提示词均列于代码仓库中。正如第 6 节所讨论的，尽管这些超参数较为常用且经验上有效，在 CAM 中仍需人工配置。未来一个很有前景的方向，是开发自动化机制（如强化学习），减少对人工调参的依赖，并增强系统面对不同任务时的适应能力。

**记忆范围。** 对于单文档阅读理解数据集（NovelQA、QMSum 和 FABLES），CAM 为每份文档构建独立的记忆结构，以便在整份文档范围内进行推理。对于多文档阅读理解数据集（MultiHop-RAG、ODSum-Story 和 ODSum-Meeting），CAM 为所有涉及的文档构建统一记忆表示，使其在回答复杂查询时能够从多份文档检索并整合信息。

**指标。** 考虑到 NovelQA、QMSum 和 ODSum 的参考回答均为自由形式，我们使用两项广泛采用的指标进行评估 [13, 14, 18]：（1）ROUGE F 值 [48]，用于衡量模型输出与参考回答的词汇相似度；（2）LLM 评审 [49]，使用 GPT-4o 比较输出和参考答案，得到准确率分数 ACC-L。对于事实问答数据集 MultiHop-RAG 和 HotpotQA，我们使用精确匹配（EM）及 F1 分数衡量性能。在主张验证数据集 FABLES 上，我们遵循其原论文 [45]，分别报告正标签和负标签的 F1 分数，即 $\mathrm{F1_P}$ 和 $\mathrm{F1_N}$。

**计算资源。** 在分析实验（表 3）中，我们在 NVIDIA A100 GPU 上运行开源 LLM（Llama-3.1-8B-Instruct 与 Qwen2.5-7B-Instruct）。

## 附录 F 按问题类型划分的性能

为更细致地评估 CAM，我们还利用 NovelQA 数据集的问题类型标注进行了细粒度性能分析。借助这些标注，我们可以沿两个相互正交的维度研究模型行为：（1）基于复杂度的类别，包括单跳（42.8%）、多跳（35.0%）和细节（22.2%）问题，反映得出答案所需的推理深度；（2）基于语义侧面的类别，涵盖时间、含义、跨度、场景、关系、角色和情节，反映每项查询的语义焦点。

**表 5：基于复杂度类别的评估结果（ACC-L）。**

| 方法 | 单跳 | 多跳 | 细节 | 平均 |
|---|---:|---:|---:|---:|
| ReadAgent | 45.3 | 39.5 | 41.0 | 42.3 |
| RAPTOR | 49.1 | 46.3 | 47.6 | 47.8 |
| GraphRAG | 47.7 | 44.2 | 42.5 | 45.3 |
| MemTree | 45.8 | 41.5 | 42.3 | 43.5 |
| CAM | **53.4** | **51.3** | **51.8** | **52.3** |

**表 6：基于语义侧面类别的评估结果（ACC-L）。**

| 方法 | 时间 | 含义 | 跨度 | 场景 | 关系 | 角色 | 情节 | 平均 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| ReadAgent | 32.5 | 31.7 | 30.3 | 48.2 | 43.5 | 49.1 | 49.5 | 42.3 |
| RAPTOR | 39.7 | 36.2 | 33.5 | 53.3 | 47.8 | 53.9 | 55.3 | 47.8 |
| GraphRAG | 37.1 | 33.4 | 32.8 | 50.5 | **52.0** | 48.6 | 53.4 | 45.3 |
| MemTree | 30.8 | 33.0 | 31.2 | 49.3 | 49.5 | 50.3 | 51.7 | 43.5 |
| CAM | **45.2** | **41.5** | **39.3** | **58.5** | 51.2 | **57.6** | **59.1** | **52.3** |

我们在相同设置下，将 CAM 与四种代表性基线（ReadAgent、RAPTOR、GraphRAG 和 MemTree）进行比较。如表 5 和表 6 所示，CAM 在这些问题类别上始终优于基线；对于要求整合语义上相距遥远的证据并维持上下文连贯性的复杂问题，其优势尤其明显。这些结果证实，CAM 能够很好地泛化到不同推理深度和语义焦点，进一步表明它适合处理上下文密集型推理任务。

## 附录 G 事实问答性能

为更全面地验证 CAM 的有效性，我们还在三个广泛使用的问答基准上开展实验：HotpotQA [59]、MuSiQue [60] 和 2WikiMultiHopQA（2Wiki）[61]。对于各数据集的文本语料，我们沿用 HippoRAG [16] 和 IRCoT [62] 的做法，收集所有候选段落，包括支持段落和干扰段落。我们采用主实验中的同一模型配置（GPT-4o-mini 和 `text-embedding-3-small`）构建记忆结构，并报告标准精确匹配（EM）与 F1 分数。

**表 7：三个事实问答数据集上的评估结果。** “CAM w/ FM”表示使用细粒度建模的 CAM。

| 方法 | HotpotQA EM | HotpotQA F1 | 2Wiki EM | 2Wiki F1 | MuSiQue EM | MuSiQue F1 |
|---|---:|---:|---:|---:|---:|---:|
| RAPTOR | 59.2 | 73.4 | 58.4 | 66.2 | 23.3 | 34.8 |
| HippoRAG | 58.5 | 73.1 | 63.8 | 71.0 | 21.8 | 32.2 |
| CAM | 60.7 | 75.4 | 61.4 | 70.2 | 24.6 | 37.2 |
| CAM w/ FM | **62.1** | **76.5** | **65.8** | **75.3** | **26.8** | **38.5** |

表 7 报告了 CAM 和两种代表性基线的性能。CAM 在 HotpotQA 和 MuSiQue 上始终优于基线，并在 2Wiki 上取得有竞争力的结果。我们观察到，HippoRAG 在这些数据集上表现突出，尤其是 2Wiki。这可能源于它使用了基于 LLM 的细粒度知识建模；这种方法对以实体为中心的数据集尤为有效 [16]。为探究这一点，我们进一步用细粒度知识建模扩展 CAM 并研究其影响。如表 7 所示，这一扩展显著提升了 CAM 在 2Wiki 和 MuSiQue 上的性能。

此外，我们还人工检查了 LLM 在 2Wiki 上抽取的实体质量，发现与主实验中的叙事性文档相比，LLM 更善于从以实体为中心的文本中识别有信息量的实体与关系。这表明，细粒度知识建模技术尤其适合以实体丰富的上下文为基础的事实知识问答任务。

## NeurIPS 论文检查表

### 1. 主张

**问题：** 摘要和引言中的主要主张是否准确反映了论文的贡献与范围？

**回答：** 是。

**理由：** 摘要和引言包含本文的主要主张，准确反映了论文的贡献和范围。我们还在引言末尾突出说明了本文贡献。

**指南：**

- 回答 `NA` 表示摘要和引言没有包含论文所提出的主张。
- 摘要和/或引言应清楚说明论文提出的主张，包括所作贡献、重要假设和局限。对本问题回答“否”或 `NA` 不会给评审留下良好印象。
- 所提出的主张应与理论及实验结果相符，并反映这些结果预计能够在多大程度上泛化到其他设置。
- 可以把前瞻性目标作为研究动机，但必须明确说明论文尚未实现这些目标。

### 2. 局限性

**问题：** 论文是否讨论了作者所开展工作的局限性？

**回答：** 是。

**理由：** 第 6 节讨论了本文工作的局限性。

**指南：**

- 回答 `NA` 表示论文没有局限；回答“否”则表示论文存在局限，但未加讨论。
- 鼓励作者在论文中单设“局限性”一节。
- 论文应指出所有强假设，并说明结果在假设受到违反时的稳健性，例如独立性假设、无噪声设置、模型设定正确，或仅在局部成立的渐近近似。作者应反思这些假设在实践中可能如何被违反，以及会造成什么影响。
- 作者应反思论文主张的适用范围。例如，如果方法只在少量数据集上测试或只运行了少数几次，就应加以说明。一般而言，实证结果往往依赖隐含假设，而这些假设应明确阐述。
- 作者应反思影响方法性能的因素。例如，人脸识别算法可能在图像分辨率较低或光照不足时表现不佳；语音转文本系统可能无法可靠地为在线课程生成字幕，因为它不能处理专业术语。
- 作者应讨论所提算法的计算效率，以及它如何随数据集规模扩展。
- 如适用，作者应讨论方法在隐私和公平性方面可能存在的局限。
- 作者或许担心，完全坦陈局限会成为评审拒稿的理由；但更糟的结果可能是评审发现作者没有承认的局限。作者应运用最佳判断，并认识到：每一项支持透明度的个体行动，都有助于形成维护研究共同体诚信的规范。评审将收到明确指示，不得因作者坦诚说明局限而予以惩罚。

### 3. 理论假设与证明

**问题：** 对于每项理论结果，论文是否给出了完整的假设集合以及完整且正确的证明？

**回答：** `NA`。

**理由：** 本文不包含理论结果。

**指南：**

- 回答 `NA` 表示论文不包含理论结果。
- 所有定理、公式和证明都应编号并交叉引用。
- 任何定理所依赖的全部假设，都应在定理陈述中明确说明或引用。
- 证明可以出现在论文正文或补充材料中；若出现在补充材料，鼓励作者提供简短的证明草图以建立直觉。
- 相反，核心正文中的任何非形式化证明，都应在附录或补充材料中配有形式化证明。
- 证明所依赖的定理与引理应得到恰当引用。

### 4. 实验结果的可复现性

**问题：** 就影响论文主要主张和/或结论的部分而言，无论是否提供代码和数据，论文是否充分披露了复现主要实验结果所需的全部信息？

**回答：** 是。

**理由：** 我们在第 5.1 节和附录 E 中提供了框架的实现细节。我们还在一个匿名仓库中提供了源代码与 LLM 提示词（链接见第 5.1 节）。

**指南：**

- 回答 `NA` 表示论文不包含实验。
- 如果论文包含实验，对本问题回答“否”不会给评审留下良好印象：无论是否提供代码和数据，确保论文可复现都很重要。
- 如果贡献是数据集和/或模型，作者应说明为使他人能够复现或验证结果所采取的步骤。
- 可复现性可根据贡献性质通过不同方式实现。例如，如果贡献主要是新架构，完整描述架构或许就已足够；如果贡献是具体模型和实证评估，则可能还需让他人能够用同一数据集复现模型，或提供模型访问途径。一般而言，发布代码和数据往往是实现可复现性的好办法；此外，也可提供详细复现说明、托管模型的访问方式（如大语言模型），发布模型检查点，或采用其他适合本研究的方法。
- 尽管 NeurIPS 不要求发布代码，但会议要求所有投稿提供某种合理的复现途径；具体方式可以取决于贡献性质。例如：
  - 若主要贡献是新算法，论文应明确说明如何复现该算法。
  - 若主要贡献是新模型架构，论文应清楚、完整地描述该架构。
  - 若贡献是新模型（如大语言模型），则应提供模型访问方式以便复现，或提供复现模型的途径（如开放数据集，或数据集构建说明）。
  - 我们认识到，在某些情况下实现可复现性并不容易；作者可以说明自己所提供的具体复现途径。对于闭源模型，其访问可能受到某种限制（如仅限注册用户），但其他研究者仍应有办法复现或验证结果。

### 5. 数据与代码的开放获取

**问题：** 论文是否开放提供数据和代码，并在补充材料中给出足够说明，以忠实复现主要实验结果？

**回答：** 是。

**理由：** 我们在匿名仓库中提供了数据集访问说明，以及复现结果所需的源代码（链接见第 5.1 节）。

**指南：**

- 回答 `NA` 表示论文不包含需要代码的实验。
- 更多详情请参阅 [NeurIPS 代码与数据提交指南](https://nips.cc/public/guides/CodeSubmissionPolicy)。
- 我们鼓励发布代码和数据，但理解这有时无法实现，因此回答“否”是可以接受的。论文不能仅仅因为没有包含代码而被拒稿，除非这对其贡献至关重要，例如论文提出了新的开源基准。
- 说明中应包含复现结果所需的准确命令与环境。更多详情见 NeurIPS 代码与数据提交指南。
- 作者应提供数据访问与准备说明，包括如何访问原始数据、预处理数据、中间数据和生成数据等。
- 作者应提供脚本，以复现新方法及基线的全部实验结果。如果只能复现实验的一部分，应说明遗漏了哪些实验以及原因。
- 投稿时，为维持匿名性，作者应发布匿名版本（如适用）。
- 建议在补充材料中提供尽可能多的信息，但也允许包含数据和代码的 URL。

### 6. 实验设置与细节

**问题：** 论文是否明确给出了理解实验结果所需的全部训练与测试细节，例如数据划分、超参数、超参数选择方式和优化器类型等？

**回答：** 是。

**理由：** 我们在第 5.1 节和附录 E 中给出了 CAM 框架的实现细节。全部 LLM 提示词均包含在代码仓库中。

**指南：**

- 回答 `NA` 表示论文不包含实验。
- 核心正文应以足以理解实验和合理解读结果的详细程度说明实验设置。
- 完整细节可以在代码、附录或补充材料中提供。

### 7. 实验的统计显著性

**问题：** 论文是否报告了定义适当且正确的误差条，或其他关于实验统计显著性的恰当信息？

**回答：** 是。

**理由：** 图 3(b) 包含误差条，用以展示本框架在在线设置下的性能稳定性。

**指南：**

- 回答 `NA` 表示论文不包含实验。
- 如果至少为支持主要主张的实验提供了误差条、置信区间或统计显著性检验，作者应回答“是”。
- 应明确说明误差条捕捉的变异因素，例如训练/测试划分、初始化、某项参数的随机抽取，或给定实验条件下的整次运行。
- 应说明误差条的计算方法，例如闭式公式、库函数或 bootstrap。
- 应给出所作假设，例如误差服从正态分布。
- 应明确说明误差条表示标准差还是均值标准误。
- 可以报告一个标准差的误差条；但如果尚未验证误差正态性，相较把两个标准差的误差条称为 96% 置信区间，最好只明确说明它是两个标准差的误差条。
- 对于非对称分布，作者应避免在表格或图中展示会使结果超出合理范围（如出现负错误率）的对称误差条。
- 如果表格或图中报告了误差条，应在正文中解释其计算方式，并引用相应图表。

### 8. 实验计算资源

**问题：** 对于每项实验，论文是否提供了足够的信息来说明复现实验所需的计算机资源，包括计算工作节点类型、内存和运行时间？

**回答：** 是。

**理由：** 我们在附录 E 中提供了计算资源信息。

**指南：**

- 回答 `NA` 表示论文不包含实验。
- 论文应指出使用的计算工作节点类型（CPU 或 GPU）、内部集群或云服务商，以及相关内存与存储信息。
- 论文应给出每次独立实验运行所需的计算量，并估算总计算量。
- 论文应披露整个研究项目是否使用了多于已报告实验的计算资源，例如未纳入论文的初步实验或失败实验。

### 9. 伦理准则

**问题：** 论文所开展的研究是否在各个方面都符合 [NeurIPS 伦理准则](https://neurips.cc/public/EthicsGuidelines)？

**回答：** 是。

**理由：** 本研究在各个方面均符合 NeurIPS 伦理准则。

**指南：**

- 回答 `NA` 表示作者尚未审阅 NeurIPS 伦理准则。
- 如果回答“否”，作者应说明必须偏离该伦理准则的特殊情形。
- 作者应确保维持匿名性，例如某些法律或其所在司法辖区规定造成特殊考虑时。

### 10. 更广泛影响

**问题：** 论文是否讨论了所开展工作可能产生的正面和负面社会影响？

**回答：** 是。

**理由：** 第 6 节讨论了本研究的潜在影响。

**指南：**

- 回答 `NA` 表示所开展的工作没有社会影响。
- 若回答 `NA` 或“否”，作者应解释为何工作没有社会影响，或为何论文未讨论社会影响。
- 负面社会影响的例子包括潜在的恶意或非预期用途（如虚假信息、生成虚假身份、监控）、公平性问题（如部署后作出对特定群体不公平的决定）、隐私问题和安全问题。
- 会议预计许多论文属于基础研究，与具体应用无关，更谈不上部署。然而，如果研究存在通往负面应用的直接路径，作者应加以指出。例如，提高生成模型质量可能被用于生成传播虚假信息的深度伪造，这一点值得说明；而泛用神经网络优化算法可能让人更快训练深度伪造生成模型，则无需特意指出。
- 作者应考虑：技术按预期使用且正常运行时可能造成的伤害；按预期使用却给出错误结果时可能造成的伤害；有意或无意误用所造成的伤害。
- 如果存在负面社会影响，作者还可讨论可能的缓解策略，例如限制模型发布、在提供攻击手段时同时提供防御措施、建立监测滥用的机制、监测系统如何随反馈学习的机制，以及提升机器学习的效率和可及性。

### 11. 防护措施

**问题：** 对于误用风险较高的数据或模型（如预训练语言模型、图像生成器或抓取所得的数据集），论文是否说明了为负责任发布而采取的防护措施？

**回答：** `NA`。

**理由：** 本文不构成此类风险。

**指南：**

- 回答 `NA` 表示论文不构成此类风险。
- 对误用或双重用途风险较高的已发布模型，应配套必要防护措施以实现受控使用，例如要求用户遵守使用指南或访问限制，或实现安全过滤器。
- 从互联网抓取的数据集可能构成安全风险。作者应说明如何避免发布不安全图像。
- 我们认识到，实现有效防护并不容易，且许多论文并不需要防护；但仍鼓励作者认真考虑并尽最大努力。

### 12. 既有资产的许可证

**问题：** 对于论文使用的既有资产（如代码、数据和模型），其创作者或原所有者是否得到恰当注明，其许可证和使用条款是否得到明确说明并受到恰当遵守？

**回答：** 是。

**理由：** 我们引用了产生实验数据集的原始论文。

**指南：**

- 回答 `NA` 表示论文没有使用既有资产。
- 作者应引用产生所用代码包或数据集的原始论文。
- 作者应说明资产版本，并尽可能提供 URL。
- 每项资产都应包含许可证名称，如 CC-BY 4.0。
- 对从特定来源（如网站）抓取的数据，应提供该来源的版权与服务条款。
- 如果发布资产，应在资产包中提供许可证、版权信息和使用条款。对于常用数据集，paperswithcode.com/datasets 已整理部分许可证；其许可证指南可以辅助判断数据集许可证。
- 对重新打包的既有数据集，应同时提供原许可证和衍生资产的许可证（如果后者已改变）。
- 如果无法在线获取这些信息，鼓励作者联系资产创作者。

### 13. 新资产

**问题：** 论文是否充分记录了所引入的新资产，并随资产一同提供这些文档？

**回答：** `NA`。

**理由：** 本文未发布新资产。

**指南：**

- 回答 `NA` 表示论文未发布新资产。
- 研究者应使用结构化模板，把数据集、代码或模型的细节作为投稿材料的一部分加以说明，包括训练、许可证和局限等信息。
- 论文应讨论是否以及如何从资产所涉及的人员处获得同意。
- 投稿时，请务必将资产匿名化（如适用）；可以提供匿名 URL，也可附匿名压缩包。

### 14. 众包与人类受试者研究

**问题：** 对于众包实验和涉及人类受试者的研究，论文是否给出了向参与者提供的完整说明文本、适用时的截图，以及报酬（如有）的详细信息？

**回答：** `NA`。

**理由：** 本文不涉及众包实验。

**指南：**

- 回答 `NA` 表示论文不涉及众包或人类受试者研究。
- 可以在补充材料中包含这些信息；但如果论文的主要贡献涉及人类受试者，核心正文中应尽可能多地提供细节。
- 根据 NeurIPS 伦理准则，参与数据收集、整理或其他劳动的工作者，应至少获得数据收集者所在国家规定的最低工资。

### 15. 机构审查委员会（IRB）批准或人类受试者研究的同等批准

**问题：** 论文是否说明了研究参与者面临的潜在风险、这些风险是否已向受试者披露，以及是否取得了机构审查委员会（IRB）批准，或依据作者所在国家和机构要求取得了同等批准/审查？

**回答：** `NA`。

**理由：** 本文不涉及众包实验。

**指南：**

- 回答 `NA` 表示论文不涉及众包或人类受试者研究。
- 依国家不同，任何人类受试者研究都可能需要 IRB 或同等批准。若已获得批准，应在论文中明确说明。
- 我们认识到，不同机构和地区的相关程序可能存在显著差异；我们期望作者遵守 NeurIPS 伦理准则及其所在机构的指南。
- 初次投稿时，不得包含会破坏匿名性的信息，例如开展审查的机构名称（如适用）。

### 16. LLM 使用声明

**问题：** 如果 LLM 的使用是本研究核心方法的重要、原创或非标准组成部分，论文是否对此进行了说明？若 LLM 仅用于写作、编辑或格式化，且不影响核心方法、科学严谨性或原创性，则无需声明。

**回答：** `NA`。

**理由：** 本文核心方法的开发不涉及 LLM 的使用。

**指南：**

- 回答 `NA` 表示本研究的核心方法开发没有把 LLM 用作任何重要、原创或非标准组成部分。
- 关于哪些内容应当或不应当说明，请参阅 [NeurIPS 2025 LLM 政策](https://neurips.cc/Conferences/2025/LLM)。

## Sources

- `papers/agent-memory/CAM - A Constructivist View of Agentic Memory for LLM-Based Reading Comprehension/CAM - A Constructivist View of Agentic Memory for LLM-Based Reading Comprehension.pdf`
