# 从 RAG 到记忆：大语言模型的非参数持续学习

## 译文说明

本译文依据本地论文 PDF 全文精译，覆盖正文、图表题注、脚注及附录。模型名、数据集名、算法名与引用采用原文写法；表格数值、符号和强调方式按原文保留；参考文献保留原始书目信息，以便准确检索。图中承载方法含义的文字已译出。原文个别表述或数值若彼此不一致，译文忠实保留，并以译注指出。

Bernal Jiménez Gutiérrez$^{*1}$，Yiheng Shu$^{*1}$，Weijian Qi$^1$，Sizhe Zhou$^2$，Yu Su$^1$

$^1$ 美国俄亥俄州立大学，俄亥俄州哥伦布市<br>
$^2$ 美国伊利诺伊大学厄巴纳-香槟分校，伊利诺伊州

通讯作者：Bernal Jiménez Gutiérrez <jimenezgutierrez.1@osu.edu>，Yiheng Shu <shu.251@osu.edu>，Yu Su <su.809@osu.edu>

$^*$ 同等贡献。

第 42 届国际机器学习大会（ICML 2025），加拿大温哥华。PMLR 267，2025。版权归作者所有。

arXiv:2502.14802v2 [cs.CL]，2025 年 6 月 19 日

## 摘要

持续获取、组织并运用知识的能力，是人类智能的一项关键特征；AI 系统若要充分释放潜力，就必须逼近这种能力。鉴于利用大语言模型（LLM）进行持续学习面临诸多挑战，检索增强生成（RAG）已成为引入新信息的主流方式。然而，RAG 对向量检索的依赖，使其难以模拟人类长期记忆所具有的动态性和互联性。近期的 RAG 方法在向量嵌入之外引入知识图谱等多种结构，以弥补其中一些缺口，尤其是意义建构与联想性方面的不足；但它们在更基础的事实记忆任务上的表现，却远低于标准 RAG。

本文着手解决这一非预期的性能退化，并提出 HippoRAG 2：一个在事实记忆、意义建构和联想记忆任务上全面优于标准 RAG 的框架。HippoRAG 2 以 HippoRAG 所采用的个性化 PageRank 算法为基础，通过更深度地整合段落，并在在线阶段更有效地使用 LLM 加以增强。这一组合使该 RAG 系统在效果上更接近人类长期记忆：相较最先进的嵌入模型，它在联想记忆任务上提升 7%，同时还展现出更强的事实知识与意义建构记忆能力。本工作为 LLM 的非参数持续学习铺平了道路。代码和数据见 <https://github.com/OSU-NLP-Group/HippoRAG>。

## 1 引言

世界瞬息万变，而持续吸收、整合和运用知识的能力，是人类智能最重要的特征之一。无论是律师应对不断变化的法律框架，还是研究人员追踪多层面的科学进展，我们的生产力在很大程度上都依赖于这种非凡的持续学习能力。AI 系统若要成为真正有用、达到人类水平的助手，就必须逼近这种能力。

近年来，大语言模型（LLM）在人类智能的许多方面取得了显著进展。然而，由于其参数知识具有复杂的分布式特性，赋予这些模型类似人类、能够不断演化的长期记忆能力，一直面临两方面的重大挑战：既难以充分吸收新知识（Zhong et al., 2023；Hoelscher-Obermaier et al., 2023），又难以避免灾难性遗忘（Cohen et al., 2024；Gu et al., 2024）。检索增强生成（RAG）由此成为规避这些障碍的一种方式：它无需改变 LLM 的参数表示，便能让 LLM 以非参数方式访问新信息。凭借简洁性和稳健性（Zhong et al., 2023；Xie et al., 2024），RAG 很快成为生产级 LLM 系统事实上的持续学习方案。

然而，RAG 依赖简单的向量检索，因而无法捕捉人类互联式长期记忆系统的两个关键方面：**意义建构**（sense-making；Klein et al., 2006），即解释规模更大、更复杂或存在不确定性的上下文的能力；以及**联想性**（associativity；Suzuki, 2005），即在彼此分散的知识片段之间建立多跳联系的能力。

近期，人们提出了若干让 LLM 显式组织检索语料库的 RAG 框架，以应对上述局限。为了增强意义建构能力，此类结构增强 RAG 方法让 LLM 生成摘要（Edge et al., 2024；Sarthi et al., 2024；Chen et al., 2023），或生成知识图谱（KG）结构（Guo et al., 2024），从而把彼此分散但相互关联的段落连接起来，提升 RAG 系统理解长篇故事等更长、更复杂话语的能力。为了弥补联想性缺口，HippoRAG（Gutiérrez et al., 2024）的作者利用个性化 PageRank 算法（Haveliwala, 2002）以及 LLM 自动构建 KG 的能力，使检索过程具备多跳推理能力。

> **图 1 内文字：**<br>
> **事实记忆：**“George Rankin 的职业是什么？”；“Oliver Badman 是政治家”（错误）；“George Rankin 是政治家”（正确）；“Thomas Marwick 是政治家”（错误）。<br>
> **意义建构：**“Cinderella 如何迎来幸福结局？”；“Cinderella 参加了皇家舞会”（正确）；“王子用遗失的玻璃鞋在王国中搜寻”（正确）；“鞋子完全合脚后，Cinderella 与王子重逢”（正确）。<br>
> **联想性：**“Erik Hort 的出生地属于哪个县？”；“Erik Hort 的出生地是 Montebello”（正确）；“Marina 出生于 Minsk”（错误）；“Montebello 属于 Rockland County”（正确）；二者“相关联”。<br>
> 图例：RAPTOR、GraphRAG、HippoRAG、NV-Embed-v2、HippoRAG 2。

**图 1：在三个关键维度上评估持续学习能力。** 事实记忆（NaturalQuestions、PopQA）、意义建构（NarrativeQA）和联想性（MuSiQue、2Wiki、HotpotQA、LV-Eval）。HippoRAG 2 在所有基准类别上均优于其他方法，向真正的长期记忆系统又迈进一步。

尽管这些方法在两类更具挑战性的记忆任务中表现强劲，但要让 RAG 真正接近人类长期记忆，还必须在更简单的记忆任务上保持稳健。为了判断这些系统能否达到这种稳健性，我们开展了综合实验：不仅同时通过多跳问答和大规模话语理解来评估其联想性与意义建构能力，还通过标准 RAG 本就擅长处理的简单问答任务，检验其事实记忆能力。

如图 1 所示，我们的评估揭示：在全部三类基准上，以往所有结构增强方法都不及当前最强的基于嵌入的 RAG 方法。或许并不意外，每类方法在其原始实验设置之外的任务上都出现了最大的性能衰减。例如，HippoRAG 缺少基于查询的上下文化，因此其性能在大规模话语理解上下降最明显；而 RAPTOR 的 LLM 摘要机制会向检索语料库中引入噪声，致使其在简单问答和多跳问答任务上的表现大幅恶化。

在本工作中，我们利用这一实验设置，帮助解决这些创新方法的稳健性局限，同时避免只关注单一任务所带来的问题。我们提出的 HippoRAG 2 继承了 HippoRAG 的 OpenIE 与个性化 PageRank（PPR）方法，同时针对其基于查询的上下文化不足作出改进：将段落纳入 PPR 图搜索过程，使查询更深入地参与 KG 三元组的选择，并在在线检索过程中调用 LLM 来识别所检索的三元组何时与查询无关。

大量实验表明，这一设计使 HippoRAG 2 在各类任务上都能持续优于最强的标准 RAG 方法。具体而言，在联想性任务上，本方法平均领先标准 RAG 7 个百分点；在事实记忆和意义建构任务上不仅没有退化，甚至略有提升。此外，我们还表明，该方法对不同检索器以及强大的开源和闭源 LLM 均具稳健性，从而提供了很高的使用灵活性。所有结果都说明，HippoRAG 2 是迈向更类人化的 LLM 非参数持续学习系统的一项有前景的进展。

## 2 相关工作

### 2.1 LLM 的持续学习

随着 LLM 越来越多地用于现实应用，它们能否随时间获取并整合新知识，同时保留既有信息，变得愈发重要；这一方向已有许多基准评测工作（Zhong et al., 2023；Liska et al., 2022；Kim et al., 2024；Roth et al., 2024；Li et al., 2024）。由于完整预训练 LLM 的计算成本极高，人们采用多种技术来赋予模型持续学习能力。这些方法大致分为三类：持续微调、模型编辑和 RAG（Shi et al., 2024）。

**持续微调**是指周期性地在新数据上训练 LLM，可以通过持续预训练（Jin et al., 2022）、指令微调（Zhang et al., 2023）和对齐微调（Zhang et al., 2024）等方式实现。尽管持续微调能够有效纳入新的语言模式与推理技能，却会遭遇灾难性遗忘（Huang et al., 2024）：随着新数据的引入，先前学到的知识会丢失。此外，其计算开销也使频繁更新在现实应用中难以实现。

**模型编辑**技术（Yao et al., 2023）直接修改模型中的特定参数来更新知识，提供了一种更轻量的替代方案。然而，研究发现这类更新高度局部化，对本应随之改变的相关信息几乎不起作用。

**RAG** 已成为一种可扩展且实用的持续学习替代方案。RAG 不修改 LLM 本身，而是在推理时检索相关的外部信息，从而实时适应新知识。下一节将讨论这一面向 LLM 的非参数持续学习方案的若干方面。

### 2.2 LLM 的非参数持续学习

**编码器模型的改进**，特别是采用 LLM 骨干网络的改进，能够生成更好地捕捉语义关系的高质量嵌入，显著增强 RAG 系统，并改善用于 LLM 生成的检索质量。近期模型（Li et al., 2023；Muennighoff et al., 2025；Lee et al., 2025）借助 LLM、大规模语料库、改进的架构和指令微调，实现了显著的检索增益。NV-Embed-v2（Lee et al., 2025）是本文的主要对比对象。

**意义建构**是理解大规模或复杂事件、经历或数据的能力（Koli et al., 2024）。标准 RAG 方法在这方面能力有限，因为此类任务需要整合来自分散段落的信息；因此，人们提出了若干 RAG 框架加以解决。RAPTOR（Sarthi et al., 2024）和 GraphRAG（Edge et al., 2024）都会生成整合检索语料库的摘要，但它们检测“应该总结什么”和“以何种粒度总结”的流程不同：RAPTOR 使用高斯混合模型检测待总结的文档簇；GraphRAG 则使用图社区检测算法，可以总结文档、带关系的实体簇，或这些元素的组合。LightRAG（Guo et al., 2024）采用双层检索机制，把图结构与向量检索相结合，以增强低层和高层知识的综合信息检索能力。

尽管 GraphRAG 和 LightRAG 与 HippoRAG 2 一样都使用 KG，但我们的 KG 用于辅助检索过程，而不是扩充检索语料库。因此，HippoRAG 2 引入的 LLM 生成噪声更少；正是这类噪声导致上述方法在单跳和多跳问答任务上的性能下降。

**联想性**是为实现高效检索而在彼此分散的事实之间建立多跳联系的能力。它是持续学习的重要组成部分；标准 RAG 依赖彼此独立的向量检索，因而无法模拟这一特性。HippoRAG（Gutiérrez et al., 2024）是唯一通过在显式构建的开放 KG 上运用 PPR 算法来解决这一特性的 RAG 框架。HippoRAG 2 深受 HippoRAG 启发，因此在多跳问答任务上表现很好；但它对段落、查询和三元组的整合更为全面，因而在意义建构和事实记忆任务上也能取得更加全面的表现。

## 3 HippoRAG 2

### 3.1 概览

HippoRAG（Gutiérrez et al., 2024）是一个受神经生物学启发的 LLM 长期记忆框架，其中每个组件都参照了人类记忆中对应的神经生物学结构。该框架包含三个主要组件：1）充当人工新皮层的 LLM；2）模拟海马体自联想特性的 KG 和个性化 PageRank 算法；3）连接前两者的检索编码器，对应海马旁区的一项功能。这些组件协同工作，以复现人类长期记忆中观察到的交互。

HippoRAG 的离线索引过程使用 LLM 将段落处理为 KG 三元组，再将其纳入 KG，即人工海马索引。与此同时，检索编码器负责检测同义关系，以连接不同信息。在 HippoRAG 的在线检索过程中，充当新皮层的 LLM 从查询中抽取命名实体，检索编码器则在 KG 中寻找与其最相似的对应项。随后，使用 KG 中与这些实体相对应的节点——本文称为**种子节点**——运行个性化 PageRank（PPR）算法。更具体地说，这些种子节点用于设定 PPR 中的重置概率；PPR 在原始 PageRank 算法的基础上，将概率向种子节点及其邻域分配，从而实现 HippoRAG 基于上下文的检索。尽管 HippoRAG 试图从非参数 RAG 构建记忆，但一个关键缺陷阻碍了其有效性：以实体为中心的方法会在索引和推理过程中造成上下文丢失，也会带来语义匹配困难。

> **图 2 内文字：**<br>
> **离线索引：** 段落 → 三元组 → 知识图谱；① LLM 执行 OpenIE；② 基于嵌入检测同义关系；③ 稠密-稀疏整合。<br>
> **在线检索与问答：** 查询 → 排序后的段落 / 排序后的三元组 → 过滤后的三元组 → 知识图谱 → 答案；① 检索段落和三元组；② 识别记忆（三元组过滤）；③ 分配种子节点权重；④ PPR 图搜索；⑤ 使用所选段落进行问答阅读。<br>
> 图例：短语节点、段落节点、种子节点、关系边、同义边、上下文边。

**图 2：HippoRAG 2 方法。** 在离线索引阶段，我们使用 LLM 从段落中抽取开放 KG 三元组，并对短语节点进行同义词检测。这些短语与段落共同构成开放 KG。在在线检索阶段，嵌入模型同时为段落和三元组评分，以识别两类可供个性化 PageRank（PPR）算法使用的种子节点。识别记忆使用 LLM 过滤排名靠前的三元组。随后，PPR 算法在 KG 上执行基于上下文的检索，为最终问答任务提供最相关的段落。上图 KG 节点的不同颜色代表其概率质量；颜色越深，表示 PPR 过程所诱导的概率越高。

HippoRAG 2 建立在 HippoRAG（Gutiérrez et al., 2024）提出、受神经生物学启发的长期记忆框架之上，结构同样包含图 2 所示的两个阶段：离线索引和在线检索。不过，HippoRAG 2 还引入了若干关键改进，使其与人类记忆机制更加一致：1）在 KG 中无缝整合概念信息和上下文信息，提升所构建索引的全面性与原子性（第 3.2 节）；2）不再局限于孤立的 KG 节点，而是利用 KG 结构实现更具上下文感知能力的检索（第 3.3 节）；3）引入识别记忆，以改善图搜索的种子节点选择（第 3.4 节）。下面将更详细地介绍整体流程和每项改进。

**离线索引。** 1）与 HippoRAG 一样，HippoRAG 2 使用 LLM，通过 OpenIE 从每个段落中抽取三元组；关系和实体的生成不受任何约束或模式限制。随后，这些三元组被组织为无模式 KG，即海马索引。本文将三元组的主语或宾语称为**短语**，将连接二者的边称为**关系边**。2）接着，检索编码器对 KG 中的短语对进行评估，检测向量相似度高于预设阈值的短语对，并在它们之间添加**同义边**。这一过程使 KG 能够跨越不同段落连接同义词，从而在学习过程中促进新旧知识的整合。3）最后，将这个基于短语的 KG 与原始段落相结合，使最终的开放 KG 同时包含概念信息和上下文信息（第 3.2 节）。

**在线检索。** 1）使用编码器将查询链接至相关三元组与段落，识别可作为图搜索种子节点的节点（第 3.3 节）。2）在三元组链接过程中，识别记忆充当过滤器，确保从检索集合中只保留相关三元组作为最终种子节点（第 3.4 节）。3）随后利用这些最终种子节点设置 PPR 算法中的重置概率，使其能够执行上下文感知检索，并对链接结果进行精炼，以检索最相关的段落。4）最后，检索到的段落作为最终问答任务的上下文输入。下面详细介绍 HippoRAG 2 的各项改进。

### 3.2 稠密-稀疏整合

HippoRAG KG 中的节点主要由描述概念的短语构成，本文将其称为**短语节点**。这种图结构受到“概念-上下文权衡”的制约：概念简洁、容易泛化，却往往伴随信息损失；上下文则提供塑造概念解释与应用的具体情境，能够丰富语义，但也会增加复杂性。然而，在人类记忆中，概念与上下文紧密互联。稠密编码与稀疏编码理论揭示了大脑如何以不同粒度表示和处理信息（Beyeler et al., 2019）。稠密编码通过大量神经元的同步激活来编码信息，形成分布式且冗余的表示；相反，稀疏编码只依赖最低限度的神经激活，仅调动一小部分神经元，以提高效率和存储紧凑性。

受人脑中稠密-稀疏整合的启发，我们将短语节点视为对抽取概念的一种稀疏编码，同时把稠密编码纳入 KG，以表示这些概念所源自的上下文。首先，我们使用嵌入模型，采用与短语编码相似的方法对上下文进行编码；随后，以特定方式在 KG 中整合这两类编码。HippoRAG 的文档集成只是聚合图搜索分数与嵌入匹配分数；与之不同，我们在 KG 中引入**段落节点**，从而更无缝地整合上下文信息。该方法保留了 HippoRAG 原有的离线索引流程，但在构图时加入与段落有关的节点和边，丰富了图结构。具体而言，语料库中的每个段落都被视为一个段落节点；标记为“contains（包含）”的**上下文边**则将该段落连接至从中抽取的所有短语。

### 3.3 更深度的上下文化

沿着上述概念-上下文权衡继续分析，我们注意到 HippoRAG 依靠命名实体识别（NER）解析查询，主要以概念为中心，往往忽视查询与 KG 之间的上下文对齐。这种以实体为中心的抽取和索引方法强烈偏向概念，导致大量上下文信号未被充分利用（Gutiérrez et al., 2024）。为克服这一局限，我们探索并评估不同的查询-KG 链接方式，希望让查询语义与图搜索起始节点更有效地对齐。具体考虑以下三种方法：

1. **NER 到节点（NER to Node）：** HippoRAG 的原始方法；先从查询中抽取实体，再使用文本嵌入将实体与 KG 节点匹配。
2. **查询到节点（Query to Node）：** 不抽取单个实体，而是使用文本嵌入直接将完整查询与 KG 中的节点匹配。
3. **查询到三元组（Query to Triple）：** 为了纳入 KG 中更丰富的上下文信息，使用文本嵌入将完整查询与图中的三元组匹配。三元组封装了概念之间的基本上下文关系，因此该方法能更全面地理解查询意图。

HippoRAG 2 默认采用查询到三元组方法；第 6.1 节将评估全部三种方法。

### 3.4 识别记忆

回忆（recall）与识别（recognition）是人类记忆检索的两个互补过程（Uner & Roediger III, 2022）。回忆是指在没有外部线索时主动提取信息，识别则依靠外部刺激来辨认信息。受此启发，我们把查询到三元组的检索建模为一个两步过程：

1. **查询到三元组：** 如第 3.3 节所述，使用嵌入模型检索图中排名前 $k$ 的三元组集合 $T$。
2. **三元组过滤：** 使用 LLM 对检索到的 $T$ 进行过滤，生成三元组集合 $T'\subseteq T$。详细提示词见附录 A。

### 3.5 在线检索

在介绍上述改进后，本节总结 HippoRAG 2 的在线检索过程。该任务需要选择种子节点，并为检索分配重置概率。HippoRAG 2 从“查询到三元组”和识别记忆所产生的过滤后三元组中识别短语节点。如果没有可用三元组，则直接使用嵌入模型检索排名靠前的段落；否则，根据短语节点在其所出现的全部过滤后三元组中的平均排序分数，最多选择 $k$ 个短语节点。

所有段落节点也都会作为种子节点，因为更广泛的激活能够改善多跳推理。短语节点按其排序分数分配重置概率；段落节点获得与其嵌入相似度成正比的分数，并使用权重因子（第 6.2 节）加以调整，以平衡短语节点与段落节点的影响。随后执行 PPR 搜索，按 PageRank 分数对段落排序，并将排名最高的段落用于下游问答。流程示例见附录 B，PPR 初始化细节见附录 G.1。

## 4 实验设置

### 4.1 基线

我们选择三类基线进行比较。第一类包括三个简单基线：经典的 BM25（Robertson & Walker, 1994），以及两个常用的稠密嵌入检索器 Contriever（Izacard et al., 2022）和 GTR（Ni et al., 2022）。

第二类基线包括 BEIR 排行榜（Thakur et al., 2021）上表现优异的若干大型嵌入模型（7B）：Alibaba-NLP/GTE-Qwen2-7B-Instruct（Li et al., 2023）、GritLM/GritLM-7B（Muennighoff et al., 2025）和 nvidia/NV-Embed-v2（Lee et al., 2025）。

最后一类基线包括四种结构增强 RAG 方法。RAPTOR（Sarthi et al., 2024）根据语义相似度，把检索语料库组织成层次结构。GraphRAG（Edge et al., 2024）和 LightRAG（Guo et al., 2024）与我们一样利用 KG 结构，生成语料库中概念的高层摘要。HippoRAG（Gutiérrez et al., 2024）同样使用 KG，但通过 PPR 而非摘要来整合知识。

### 4.2 数据集

为评估 RAG 系统能否在增强联想性和意义建构能力的同时保留事实记忆，我们选择了对应三类关键挑战的数据集：

1. **简单问答**主要评估准确回忆和检索事实知识的能力。
2. **多跳问答**要求模型连接多条信息以推导答案，用于衡量联想性。
3. **话语理解**通过测试解释长篇复杂叙事并据此推理的能力，评估意义建构。

下面列出各类别所选数据集并作详细说明；采样数据集的统计汇总于表 1。

**表 1：数据集统计。**

| 统计项 | NQ | PopQA | MuSiQue | 2Wiki | HotpotQA | LV-Eval | NarrativeQA |
|---|---:|---:|---:|---:|---:|---:|---:|
| 查询数 | 1,000 | 1,000 | 1,000 | 1,000 | 1,000 | 124 | 293 |
| 段落数 | 9,633 | 8,676 | 11,656 | 6,119 | 9,811 | 22,849 | 4,111 |

**简单问答。** 这类常见问答任务主要围绕单个实体提出问题，因此特别适合嵌入模型直观地检索相关上下文。我们从 NaturalQuestions（NQ）数据集（由 Wang et al., 2024 收集）中随机选取 1,000 个查询；该数据集包含主题广泛的真实用户问题。我们还从 PopQA（Mallen et al., 2023）中选取 1,000 个查询，其语料库取自 2021 年 12 月的 Wikipedia 转储。[^1] 两个数据集都提供直接明了的问答对，可用于评估 RAG 系统的单跳问答能力。值得注意的是，源自 Wikipedia 的 PopQA 尤其以实体为中心，其中实体的出现频率低于 NaturalQuestions，因此非常适合评估简单问答任务中的实体识别与检索。

[^1]: <https://github.com/facebookresearch/atlas?tab=readme-ov-file#corpora>

**多跳问答。** 遵循 HippoRAG（Gutiérrez et al., 2024）的设置，我们从 MuSiQue、2WikiMultihopQA 和 HotpotQA 中分别随机选取 1,000 个查询；这些查询都需要跨段落推理。此外，我们纳入 LV-Eval（hotpotwikiqa-mixup 256k）（Yuan et al., 2024）的全部 124 个查询。该数据集通过替换关键词和短语，力求最大限度减少知识泄漏并降低过拟合。因此，与基于 Wikipedia 的数据集相比，LV-Eval 能更好地评估模型有效综合不同来源知识的能力。构建语料库时，我们把 LV-Eval 的长篇上下文切分为较短段落，同时保持与其他多跳数据集相同的 RAG 设置。

**话语理解。** 这一类别仅包含 NarrativeQA。该问答数据集中的问题要求对一部长篇小说形成连贯的整体理解。它专注于大规模话语理解，因此可用于评估所选基线和本文方法的意义建构能力。我们从 NarrativeQA 中随机选择 10 篇长文档及其对应的 293 个查询，并像上述 LV-Eval 数据集一样构建检索语料库。

### 4.3 指标

遵循 HippoRAG（Gutiérrez et al., 2024），我们使用段落召回率 recall@5 评估检索任务。问答任务则遵循 MuSiQue（Trivedi et al., 2022）的评估指标，计算基于词元的 F1 分数。

### 4.4 实现细节

在 HippoRAG 2 中，我们使用开源 Llama-3.3-70B-Instruct（AI@Meta, 2024）同时作为抽取模型（NER 和 OpenIE）与三元组过滤模型，并使用 nvidia/NV-Embed-v2 作为检索器。为保证公平比较，我们还使用相同的抽取器和检索器复现了各结构增强 RAG 对比方法。对于三元组过滤器，我们使用 DSPy（Khattab et al., 2024）的 MIPROv2 优化器和 Llama-3.3-70B-Instruct，对包含指令和演示样例的提示词进行调优；所得提示词见附录 A。我们使用检索器排序前 5 的三元组进行过滤。问答模块把检索到的前 5 个段落作为上下文，交给 LLM（GPT-4o-mini 或 Llama-3.3-70B-Instruct）生成最终答案。超参数沿用 HippoRAG 的默认设置。更多实现和超参数细节见附录 G。

## 5 结果

下面给出主要的问答与检索实验结果，其中问答过程以检索结果为上下文。更详细的实验结果见附录 C；所有构建出的 KG 的统计信息见附录 D。[^译注1]

[^译注1]: 原文正文写作“Appendix A”，但图统计实际位于附录 D（表 10），故此处按论文实际结构译出。

**表 2：以 Llama-3.3-70B-Instruct 作为问答阅读器时，各 RAG 基准上的问答性能（F1 分数）。** “无检索”表示评估阅读器的参数知识。所有结构增强 RAG 基线和 HippoRAG 2 均使用 Llama-3.3-70B-Instruct 生成其结构，并使用 NV-Embed-v2 作为检索器。本表及后续各表以粗体和下划线分别标出最佳与次佳结果。作者使用 bootstrap 统计检验评估显著性；$\dagger$ 表示 HippoRAG 2 显著优于最佳 NV-Embed-v2 基线（$p<0.05$）。

| 类别 | 检索方法 | NQ | PopQA | MuSiQue | 2Wiki | HotpotQA | LV-Eval | NarrativeQA | 平均 |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 简单基线 | 无 | 54.9 | 32.5 | 26.1 | 42.8 | 47.3 | 6.0 | 12.9 | 38.4 |
| 简单基线 | Contriever | 58.9 | 53.1 | 31.3 | 41.9 | 62.3 | 8.1 | 19.7 | 46.9 |
| 简单基线 | BM25 | 59.0 | 49.9 | 28.8 | 51.2 | 63.4 | 5.9 | 18.3 | 47.7 |
| 简单基线 | GTR（T5-base） | 59.9 | 56.2 | 34.6 | 52.8 | 62.8 | 7.1 | 19.9 | 50.4 |
| 大型嵌入模型 | GTE-Qwen2-7B-Instruct | <u>62.0</u> | **56.3** | 40.9 | 60.0 | 71.0 | 7.1 | 21.3 | 54.9 |
| 大型嵌入模型 | GritLM-7B | 61.3 | 55.8 | 44.8 | 60.6 | 73.3 | 9.8 | 23.9 | 56.1 |
| 大型嵌入模型 | NV-Embed-v2（7B） | 61.9 | 55.7 | <u>45.7</u> | 61.5 | <u>75.3</u> | 9.8 | <u>25.7</u> | <u>57.0</u> |
| 结构增强 RAG | RAPTOR | 50.7 | <u>56.2</u> | 28.9 | 52.1 | 69.5 | 5.0 | 21.4 | 48.8 |
| 结构增强 RAG | GraphRAG | 46.9 | 48.1 | 38.5 | 58.6 | 68.6 | <u>11.2</u> | 23.0 | 49.6 |
| 结构增强 RAG | LightRAG | 16.6 | 2.4 | 1.6 | 11.6 | 2.4 | 1.0 | 3.7 | 6.6 |
| 结构增强 RAG | HippoRAG | 55.3 | 55.9 | 35.1 | **71.8** | 63.5 | 8.4 | 16.3 | 53.1 |
| 结构增强 RAG | HippoRAG 2 | **63.3$^\dagger$** | <u>56.2</u> | **48.6$^\dagger$** | <u>71.0$^\dagger$</u> | **75.5** | **12.9$^\dagger$** | **25.9** | **59.8** |

**问答性能。** 表 2 给出了以 Llama-3.3-70B-Instruct 作为问答阅读器时，不同检索器在多个 RAG 基准上的问答表现。HippoRAG 2 取得最高的平均 F1，显示出跨不同设置的稳健性。大型嵌入模型优于较小模型；NV-Embed-v2（7B）的平均得分比 GTR（T5-base）高 6.6%。这些模型还以更低的计算成本胜过结构增强 RAG 方法，但它们主要擅长简单问答，在复杂情形中表现不佳。值得注意的是，HippoRAG 2 在 2Wiki 上的 F1 比 NV-Embed-v2 高 9.5%，在具有挑战性的 LV-Eval 数据集上高 3.1%。相比 HippoRAG，HippoRAG 2 的提升更大，验证了其受神经心理学启发的方法。这些结果表明，HippoRAG 2 是一个最先进的 RAG 系统，能够同时提升检索和问答性能，并可由开源模型有效驱动。

附录 C 的表 8 还给出了以 Llama 或 GPT-4o-mini 作为问答阅读器、抽取器或三元组过滤器时的更多问答结果（EM 和 F1）。GPT-4o-mini 呈现与 Llama 相同的趋势：除 HippoRAG 在多跳问答中的情况外，NV-Embed-v2 在多数场景中优于结构增强方法；HippoRAG 2 则在几乎所有设置下始终优于其他方法。各方法所需计算资源（词元数、时间和内存）分析见附录 F。

**表 3：RAG 基准上的检索性能（段落 recall@5）。** $^*$ 表示原论文报告的结果。为保证公平比较，结构增强 RAG 对比方法均使用与本文相同的 LLM 和检索器复现。GraphRAG 和 LightRAG 不在表中，因为它们不直接产生段落检索结果。

| 类别 | 检索方法 | NQ | PopQA | MuSiQue | 2Wiki | HotpotQA | 平均 |
|---|---|---:|---:|---:|---:|---:|---:|
| 简单基线 | BM25 | 56.1 | 35.7 | 43.5 | 65.3 | 74.8 | 55.1 |
| 简单基线 | Contriever | 54.6 | 43.2 | 46.6 | 57.5 | 75.3 | 55.4 |
| 简单基线 | GTR（T5-base） | 63.4 | 49.4 | 49.1 | 67.9 | 73.9 | 60.7 |
| 大型嵌入模型 | GTE-Qwen2-7B-Instruct | 74.3 | 50.6 | 63.6 | 74.8 | 89.1 | 70.5 |
| 大型嵌入模型 | GritLM-7B | <u>76.6</u> | 50.1 | 65.9 | 76.0 | 92.4 | 72.2 |
| 大型嵌入模型 | NV-Embed-v2（7B） | 75.4 | 51.0 | <u>69.7</u> | 76.5 | <u>94.5</u> | <u>73.4</u> |
| 结构增强 RAG | RAPTOR | 68.3 | 48.7 | 57.8 | 66.2 | 86.9 | 65.6 |
| 结构增强 RAG | HippoRAG$^*$ | - | - | 51.9 | 89.1 | 77.7 | - |
| 结构增强 RAG | HippoRAG（复现） | 44.4 | **53.8** | 53.2 | **90.4** | 77.3 | 63.8 |
| 结构增强 RAG | HippoRAG 2 | **78.0** | <u>51.7</u> | **74.7** | **90.4** | **96.3** | **78.2** |

**检索性能。** 对于带有支持段落标注的数据集和显式检索段落的模型，检索结果见表 3。大型嵌入模型（7B）显著优于 Contriever、GTR 等经典小型语言模型，得分至少高 9.8%。我们使用 Llama-3.3-70B-Instruct 和 NV-Embed-v2 复现的 HippoRAG 虽比原论文略有提升，但增益很小，得分仅增加 1.3%。HippoRAG 擅长以实体为中心的检索，在 PopQA 上取得最高 recall@5，但总体上落后于近期的稠密检索器和 HippoRAG 2。值得注意的是，HippoRAG 2 在大多数数据集上取得最高召回率；与最强稠密检索器 NV-Embed-v2 相比，它在 MuSiQue 和 2Wiki 上的 recall@5 分别大幅提升 5.0% 和 13.9%。[^译注2]

[^译注2]: 原文此处称 2Wiki 提升 13.9%，但表 3 中 90.4 与 76.5 的差值为 13.9 个百分点；MuSiQue 的差值为 5.0 个百分点。译文保留原文百分号表述。

## 6 讨论

### 6.1 消融研究

我们针对所提出的链接方法、构图方法和三元组过滤方法设计消融实验，结果见表 4。每项新增机制都能提升 HippoRAG 2。首先，具有更深度上下文化的链接方法带来了显著的性能提升。需要说明的是，我们没有对 NER 到节点或查询到节点方法应用过滤过程；不过，无论是否进行过滤，查询到三元组方法都始终优于另外两种链接策略。平均而言，与 NER 到节点相比，查询到三元组将 recall@5 提高了 12.5%。此外，查询到节点并未优于 NER 到节点，因为查询和 KG 节点处于不同的粒度层级，而 NER 结果与 KG 节点都对应短语级表示。

**表 4：消融实验。** 在多跳问答基准上报告段落 recall@5，比较最终设计在图链接、图构建和三元组过滤方面的若干替代方案。

| 方法 | MuSiQue | 2Wiki | HotpotQA | 平均 |
|---|---:|---:|---:|---:|
| HippoRAG 2 | **74.7** | 90.4 | **96.3** | **87.1** |
| 使用 NER 到节点 | 53.8 | **91.2** | 78.8 | 74.6 |
| 使用查询到节点 | 44.9 | 65.5 | 68.3 | 59.6 |
| 不使用段落节点 | 63.7 | 90.3 | 88.9 | 81.0 |
| 不使用过滤 | <u>73.0</u> | <u>90.7</u> | <u>95.4</u> | <u>86.4</u> |

### 6.2 控制重置概率

在运行 PPR 前设置重置概率时，我们发现必须在短语节点和段落节点这两类节点之间平衡重置概率。具体而言，所有段落节点的重置概率都乘以一个权重因子，以平衡两类节点在 PPR 中的重要性。表 5 给出了验证集结果，说明该因子对 PPR 结果至关重要。综合考虑模型在不同场景中的表现，我们默认将该因子设为 0.05。

**表 5：重置概率因子。** 在 MuSiQue 开发集和 NaturalQuestions（NQ）开发集上，采用不同段落节点权重因子时的段落 recall@5；每个集合包含 1,000 个查询。

| 权重 | 0.01 | 0.05 | 0.1 | 0.3 | 0.5 |
|---|---:|---:|---:|---:|---:|
| MuSiQue | 79.9 | **80.5** | 79.8 | 78.4 | 77.9 |
| NQ | 75.6 | **76.9** | **76.9** | 76.7 | 76.4 |

### 6.3 对语料库扩张的稳健性

随着 RAG 系统在现实世界中得到更广泛采用，它们越来越需要适应检索语料库持续增长的持续学习场景。为了解 HippoRAG 2 相比标准 RAG 处理这种设置的能力，我们设计了一项实验：将 NQ 和 MuSiQue 各自划分为四个等大的分段，每个分段包含约 250 个问题的标准文档和干扰文档。随后选择其中一个分段进行评估，并逐步加入其余分段；随着新知识不断加入，测量性能如何变化，从而模拟持续学习设置。图 3 给出了 HippoRAG 2 与最强基线 NV-Embed-v2 的 F1 分数。

**图 3：持续学习实验。** 我们将 NQ 和 MuSiQue 数据集划分为 4 个分段；随机选择一个分段，在其余 3 个分段逐步加入检索语料库时报告该分段上的 F1，以模拟不断演化的语料库。横轴为“占文档总数的百分比”，纵轴为“F1 分数”；实线表示 NQ，虚线表示 MuSiQue。

如图 3 所示，无论在简单（NQ）还是联想性（MuSiQue）持续学习设置中，HippoRAG 2 相对 NV-Embed-v2 的优势都保持得非常稳定。我们还注意到，随着知识增加，两种方法在简单问答上的表现（实线）均保持强劲；但在更复杂的联想任务中（虚线），二者性能都以相近速度下降。这种差异凸显了未来持续学习基准纳入不同任务复杂度的重要性。

### 6.4 稠密检索器的灵活性

如表 7 所示，面对不同检索器，HippoRAG 2 都始终优于直接稠密检索。值得注意的是，无论使用何种具体稠密检索器，这种性能增益都保持稳健。

**表 7：对不同稠密检索器的稳健性。** MuSiQue 子集上的段落 recall@5。

| 检索器 | 稠密检索 | HippoRAG 2 |
|---|---:|---:|
| GTE-Qwen2-7B-Instruct | 63.6 | 68.8 |
| GritLM-7B | 66.0 | 71.6 |
| NV-Embed-v2（7B） | 69.7 | 74.7 |

### 6.5 定性分析

表 6 给出了 PopQA 和 MuSiQue 的示例。对于第一个问题“ I. P. Paul 出生于哪座城市？”，NV-Embed-v2 把查询中提到的实体“I. P. Paul”排在第 1 位，而该段落已足以回答问题。不过，HippoRAG 2 做得更好：它在链接三元组时直接找到答案“Thrissur”，并在后续图搜索中把与该实体对应的段落排在第 2 位，形成了理想的检索结果。

对于第二个多跳问题“Erik Hort 的出生地属于哪个县？”，NV-Embed-v2 同样轻易识别出被提及的人物“Erik Hort”。然而，该问题需要两步推理，仅凭这一段落不足以完整作答。相比之下，HippoRAG 2 在查询到三元组步骤中检索到标题为“Montebello”的段落，其中包含能够推出答案的地理信息；在后续图搜索中，该段落也被排在前列。HippoRAG 2 的更多错误分析见附录 E。

**表 6：不同问题类型上 HippoRAG 2 与 NV-Embed-v2 的代表性检索结果（段落标题）。** 粗体表示支持段落的标题。为保证可读性，表内将原文的明显排印误差 “Monstebello” 还原为 “Montebello”。

| 类型 | 问题 | NV-Embed-v2 结果 | HippoRAG 2 过滤后的三元组 | HippoRAG 2 结果 |
|---|---|---|---|---|
| 简单问答 | I. P. Paul 出生于哪座城市？ | 1. **I. P. Paul**<br>2. Yinka Ayefele - Early life<br>3. Paul Parker (singer) | (I. P. Paul, 来自, Thrissur)<br>(I. P. Paul, 曾任市长, Thrissur municipal corporation) | 1. **I. P. Paul**<br>2. **Thrissur**<br>3. Yinka Ayefele |
| 多跳问答 | Erik Hort 的出生地属于哪个县？ | 1. **Erik Hort**<br>2. Horton Park (Saint Paul, Minnesota)<br>3. Hertfordshire | (Erik Hort, 出生于, Montebello)<br>(Erik Hort, 出生于, New York) | 1. **Erik Hort**<br>2. Horton Park (Saint Paul, Minnesota)<br>3. **Montebello, New York** |

## 7 结论

本文提出 HippoRAG 2，一个旨在解决现有 RAG 系统难以逼近人类长期记忆之动态性与互联性的新框架。它结合了个性化 PageRank 算法、更深度的段落整合，以及对 LLM 的有效在线使用。HippoRAG 2 在事实记忆、意义建构和联想记忆任务上全面优于标准 RAG 方法，展现出以往方法或被忽视、或未能在全面评估中实现的能力，从而为 LLM 持续学习与长期记忆研究开辟了新方向。未来工作可以考虑利用基于图的检索方法，进一步增强 LLM 在长对话中的情景记忆能力。

## 影响声明

本文研究检索增强生成（RAG），旨在推动大语言模型长期记忆领域的发展。尽管本工作可能产生多方面的社会影响，但除大语言模型和信息检索系统通常涉及的问题外，我们未发现任何需要特别强调的顾虑。

## 致谢

我们还要感谢俄亥俄州立大学 NLP 小组的同事提出建设性意见。本工作部分得到 ARL W911NF2220144、NSF 2112606 以及 Cisco 捐赠的支持。我们也感谢 Ohio Supercomputer Center 提供计算资源。本文所载观点和结论仅代表作者，不应解释为美国政府明示或暗示的官方政策。无论本文是否包含任何版权声明，美国政府均有权出于政府用途复制和分发其重印本。

## 参考文献

以下书目信息按原文保留，论文题名不翻译，以便准确检索。

- AI@Meta. Llama 3 model card. 2024. URL: <https://github.com/meta-llama/llama3/blob/main/MODEL_CARD.md>.
- Beyeler, M., Rounds, E. L., Carlson, K. D., Dutt, N., and Krichmar, J. L. Neural correlates of sparse coding and dimensionality reduction. *PLoS Comput Biol*, 15(6):e1006908, 2019. doi: 10.1371/journal.pcbi.1006908.
- Chen, H., Pasunuru, R., Weston, J., and Celikyilmaz, A. Walking down the memory maze: Beyond context limit through interactive reading, 2023. URL: <https://arxiv.org/abs/2310.05029>.
- Cohen, R., Biran, E., Yoran, O., Globerson, A., and Geva, M. Evaluating the ripple effects of knowledge editing in language models. *Transactions of the Association for Computational Linguistics*, 12:283-298, 2024. doi: 10.1162/tacl_a_00644. URL: <https://aclanthology.org/2024.tacl-1.16/>.
- Edge, D., Trinh, H., Cheng, N., Bradley, J., Chao, A., Mody, A., Truitt, S., and Larson, J. From local to global: A graph rag approach to query-focused summarization, 2024. URL: <https://arxiv.org/abs/2404.16130>.
- Gu, J.-C., Xu, H.-X., Ma, J.-Y., Lu, P., Ling, Z.-H., Chang, K.-W., and Peng, N. Model editing harms general abilities of large language models: Regularization to the rescue. In *Proceedings of the 2024 Conference on Empirical Methods in Natural Language Processing*, pp. 16801-16819, Miami, Florida, USA, November 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.emnlp-main.934. URL: <https://aclanthology.org/2024.emnlp-main.934/>.
- Guo, Z., Xia, L., Yu, Y., Ao, T., and Huang, C. LightRAG: Simple and fast retrieval-augmented generation, 2024. URL: <https://arxiv.org/abs/2410.05779>.
- Gutiérrez, B. J., Shu, Y., Gu, Y., Yasunaga, M., and Su, Y. Hipporag: Neurobiologically inspired long-term memory for large language models. In *The Thirty-eighth Annual Conference on Neural Information Processing Systems*, 2024. URL: <https://openreview.net/forum?id=hkujvAPVsg>.
- Haveliwala, T. H. Topic-sensitive pagerank. In *Proceedings of the Eleventh International World Wide Web Conference*, WWW 2002, May 7-11, 2002, Honolulu, Hawaii, USA, pp. 517-526. ACM, 2002. doi: 10.1145/511446.511513. URL: <https://dl.acm.org/doi/10.1145/511446.511513>.
- Hoelscher-Obermaier, J., Persson, J., Kran, E., Konstas, I., and Barez, F. Detecting edit failures in large language models: An improved specificity benchmark. In *Findings of the Association for Computational Linguistics: ACL 2023*, pp. 11548-11559, Toronto, Canada, July 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.findings-acl.733. URL: <https://aclanthology.org/2023.findings-acl.733/>.
- Huang, J., Cui, L., Wang, A., Yang, C., Liao, X., Song, L., Yao, J., and Su, J. Mitigating catastrophic forgetting in large language models with self-synthesized rehearsal. In *Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)*, pp. 1416-1428, Bangkok, Thailand, August 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.77. URL: <https://aclanthology.org/2024.acl-long.77/>.
- Izacard, G., Caron, M., Hosseini, L., Riedel, S., Bojanowski, P., Joulin, A., and Grave, E. Unsupervised dense information retrieval with contrastive learning. *Trans. Mach. Learn. Res.*, 2022. URL: <https://openreview.net/forum?id=jKN1pXi7b0>.
- Jin, X., Zhang, D., Zhu, H., Xiao, W., Li, S.-W., Wei, X., Arnold, A., and Ren, X. Lifelong pretraining: Continually adapting language models to emerging corpora. In *Proceedings of the 2022 Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies*, pp. 4764-4780, Seattle, United States, July 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.naacl-main.351. URL: <https://aclanthology.org/2022.naacl-main.351/>.
- Khattab, O., Singhvi, A., Maheshwari, P., Zhang, Z., Santhanam, K., Vardhamanan, S., Haq, S., Sharma, A., Joshi, T. T., Moazam, H., Miller, H., Zaharia, M., and Potts, C. DSPy: Compiling declarative language model calls into self-improving pipelines. 2024.
- Kim, Y., Yoon, J., Ye, S., Bae, S., Ho, N., Hwang, S. J., and Yun, S.-Y. Carpe diem: On the evaluation of world knowledge in lifelong language models. In *Proceedings of the 2024 Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies (Volume 1: Long Papers)*, pp. 5401-5415, Mexico City, Mexico, June 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.naacl-long.302. URL: <https://aclanthology.org/2024.naacl-long.302/>.
- Klein, G., Moon, B., and Hoffman, R. R. Making sense of sensemaking 1: Alternative perspectives. *IEEE Intelligent Systems*, 21(4):70-73, 2006.
- Koli, V., Yuan, J., and Dasgupta, A. Sensemaking of socially-mediated crisis information. In *Proceedings of the Third Workshop on Bridging Human-Computer Interaction and Natural Language Processing*, pp. 74-81, Mexico City, Mexico, June 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.hcinlp-1.7. URL: <https://aclanthology.org/2024.hcinlp-1.7/>.
- Kwon, W., Li, Z., Zhuang, S., Sheng, Y., Zheng, L., Yu, C. H., Gonzalez, J. E., Zhang, H., and Stoica, I. Efficient memory management for large language model serving with pagedattention. In *Proceedings of the ACM SIGOPS 29th Symposium on Operating Systems Principles*, 2023.
- Lee, C., Roy, R., Xu, M., Raiman, J., Shoeybi, M., Catanzaro, B., and Ping, W. NV-embed: Improved techniques for training LLMs as generalist embedding models. In *The Thirteenth International Conference on Learning Representations*, 2025. URL: <https://openreview.net/forum?id=lgsyLSsDRe>.
- Li, J., Armandpour, M., Mirzadeh, S. I., Mehta, S., Shankar, V., Vemulapalli, R., Tuzel, O., Farajtabar, M., Pouransari, H., and Faghri, F. Tic-LM: A multi-year benchmark for continual pretraining of language models. In *NeurIPS 2024 Workshop on Scalable Continual Learning for Lifelong Foundation Models*, 2024. URL: <https://openreview.net/forum?id=PpSDVE5rAy>.
- Li, Z., Zhang, X., Zhang, Y., Long, D., Xie, P., and Zhang, M. Towards general text embeddings with multi-stage contrastive learning. *arXiv preprint arXiv:2308.03281*, 2023.
- Liska, A., Kocisky, T., Gribovskaya, E., Terzi, T., Sezener, E., Agrawal, D., De Masson D’Autume, C., Scholtes, T., Zaheer, M., Young, S., Gilsenan-Mcmahon, E., Austin, S., Blunsom, P., and Lazaridou, A. StreamingQA: A benchmark for adaptation to new knowledge over time in question answering models. In *Proceedings of the 39th International Conference on Machine Learning*, volume 162 of *Proceedings of Machine Learning Research*, pp. 13604-13622. PMLR, 17-23 July 2022. URL: <https://proceedings.mlr.press/v162/liska22a.html>.
- Lù, X. H. BM25S: Orders of magnitude faster lexical search via eager sparse scoring, 2024. URL: <https://arxiv.org/abs/2407.03618>.
- Mallen, A., Asai, A., Zhong, V., Das, R., Khashabi, D., and Hajishirzi, H. When not to trust language models: Investigating effectiveness of parametric and non-parametric memories. In *Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)*, pp. 9802-9822, Toronto, Canada, July 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.acl-long.546. URL: <https://aclanthology.org/2023.acl-long.546/>.
- Muennighoff, N., Su, H., Wang, L., Yang, N., Wei, F., Yu, T., Singh, A., and Kiela, D. Generative representational instruction tuning. In *The Thirteenth International Conference on Learning Representations*, 2025. URL: <https://openreview.net/forum?id=BC4lIvfSzv>.
- Ni, J., Qu, C., Lu, J., Dai, Z., Hernandez Abrego, G., Ma, J., Zhao, V., Luan, Y., Hall, K., Chang, M.-W., and Yang, Y. Large dual encoders are generalizable retrievers. In *Proceedings of the 2022 Conference on Empirical Methods in Natural Language Processing*, pp. 9844-9855, Abu Dhabi, United Arab Emirates, December 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.emnlp-main.669. URL: <https://aclanthology.org/2022.emnlp-main.669/>.
- Paszke, A., Gross, S., Massa, F., Lerer, A., Bradbury, J., Chanan, G., Killeen, T., Lin, Z., Gimelshein, N., Antiga, L., Desmaison, A., Köpf, A., Yang, E. Z., DeVito, Z., Raison, M., Tejani, A., Chilamkurthy, S., Steiner, B., Fang, L., Bai, J., and Chintala, S. PyTorch: An imperative style, high-performance deep learning library. In *Advances in Neural Information Processing Systems 32*, pp. 8024-8035, 2019.
- Robertson, S. E. and Walker, S. Some simple effective approximations to the 2-poisson model for probabilistic weighted retrieval. In *Proceedings of the 17th Annual International ACM-SIGIR Conference on Research and Development in Information Retrieval*, Dublin, Ireland, 3-6 July 1994, pp. 232-241. ACM/Springer, 1994. doi: 10.1007/978-1-4471-2099-5_24.
- Roth, K., Udandarao, V., Dziadzio, S., Prabhu, A., Cherti, M., Vinyals, O., Henaff, O. J., Albanie, S., Bethge, M., and Akata, Z. A practitioner’s guide to continual multimodal pretraining. In *NeurIPS 2024 Workshop on Scalable Continual Learning for Lifelong Foundation Models*, 2024. URL: <https://openreview.net/forum?id=gkyosluSbR>.
- Sarthi, P., Abdullah, S., Tuli, A., Khanna, S., Goldie, A., and Manning, C. D. RAPTOR: recursive abstractive processing for tree-organized retrieval. In *The Twelfth International Conference on Learning Representations*, ICLR 2024, Vienna, Austria, May 7-11, 2024. OpenReview.net, 2024. URL: <https://openreview.net/forum?id=GN921JHCRw>.
- Shi, H., Xu, Z., Wang, H., Qin, W., Wang, W., Wang, Y., Wang, Z., Ebrahimi, S., and Wang, H. Continual learning of large language models: A comprehensive survey. *arXiv preprint arXiv:2404.16789*, 2024.
- Suzuki, W. A. Associative learning and the hippocampus. *Psychological Science Agenda*, February 2005.
- Thakur, N., Reimers, N., Rücklé, A., Srivastava, A., and Gurevych, I. BEIR: A heterogeneous benchmark for zero-shot evaluation of information retrieval models. In *Thirty-fifth Conference on Neural Information Processing Systems Datasets and Benchmarks Track (Round 2)*, 2021. URL: <https://openreview.net/forum?id=wCu6T5xFjeJ>.
- Trivedi, H., Balasubramanian, N., Khot, T., and Sabharwal, A. MuSiQue: Multihop questions via single-hop question composition. *Transactions of the Association for Computational Linguistics*, 10:539-554, 2022. doi: 10.1162/tacl_a_00475. URL: <https://aclanthology.org/2022.tacl-1.31/>.
- Uner, O. and Roediger III, H. L. Do recall and recognition lead to different retrieval experiences? *The American Journal of Psychology*, 135(1):33-43, 2022.
- Wang, Y., Ren, R., Li, J., Zhao, X., Liu, J., and Wen, J. REAR: A relevance-aware retrieval-augmented framework for open-domain question answering. In *Proceedings of the 2024 Conference on Empirical Methods in Natural Language Processing*, EMNLP 2024, Miami, FL, USA, November 12-16, 2024, pp. 5613-5626. Association for Computational Linguistics, 2024. URL: <https://aclanthology.org/2024.emnlp-main.321>.
- Wolf, T., Debut, L., Sanh, V., Chaumond, J., Delangue, C., Moi, A., Cistac, P., Rault, T., Louf, R., Funtowicz, M., and Brew, J. Huggingface’s transformers: State-of-the-art natural language processing. *CoRR*, abs/1910.03771, 2019. URL: <http://arxiv.org/abs/1910.03771>.
- Xie, J., Zhang, K., Chen, J., Lou, R., and Su, Y. Adaptive chameleon or stubborn sloth: Revealing the behavior of large language models in knowledge conflicts. In *The Twelfth International Conference on Learning Representations*, 2024. URL: <https://openreview.net/forum?id=auKAUJZMO6>.
- Yao, Y., Wang, P., Tian, B., Cheng, S., Li, Z., Deng, S., Chen, H., and Zhang, N. Editing large language models: Problems, methods, and opportunities. In *Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing*, pp. 10222-10240, Singapore, December 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.emnlp-main.632. URL: <https://aclanthology.org/2023.emnlp-main.632/>.
- Yuan, T., Ning, X., Zhou, D., Yang, Z., Li, S., Zhuang, M., Tan, Z., Yao, Z., Lin, D., Li, B., Dai, G., Yan, S., and Wang, Y. LV-Eval: A balanced long-context benchmark with 5 length levels up to 256k, 2024. URL: <https://arxiv.org/abs/2402.05136>.
- Zhang, H., Gui, L., Zhai, Y., Wang, H., Lei, Y., and Xu, R. Copr: Continual learning human preference through optimal policy regularization, 2024. URL: <https://arxiv.org/abs/2310.15694>.
- Zhang, Z., Fang, M., Chen, L., and Namazi-Rad, M.-R. CITB: A benchmark for continual instruction tuning. In *Findings of the Association for Computational Linguistics: EMNLP 2023*, pp. 9443-9455, Singapore, December 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.findings-emnlp.633. URL: <https://aclanthology.org/2023.findings-emnlp.633/>.
- Zhong, Z., Wu, Z., Manning, C., Potts, C., and Chen, D. MQuAKE: Assessing knowledge editing in language models via multi-hop questions. In *Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing*, pp. 15686-15702, Singapore, December 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.emnlp-main.971. URL: <https://aclanthology.org/2023.emnlp-main.971/>.

## 附录

本补充材料进一步阐述以下方面：

- 附录 A：LLM 提示词
- 附录 B：HippoRAG 2 流程示例
- 附录 C：详细实验结果
- 附录 D：图统计
- 附录 E：错误分析
- 附录 F：成本与效率
- 附录 G：实现细节与超参数

### A LLM 提示词

图 4 给出了用于三元组过滤器的 LLM 提示词，包括指令、少样本演示和输入格式。

> **指令：**<br>
> 你是一个高风险问答系统中的关键组件；世界各地的顶尖研究人员和决策者都在使用该系统。你的任务是根据事实与给定查询的相关性对事实进行过滤，确保把最关键的信息呈现给这些利益相关者。该查询需要仔细分析，并可能需要多跳推理来连接不同的信息片段。<br>
> 你必须从所提供的候选列表中选择至多 4 条与查询存在强关联的相关事实，以辅助推理并给出准确答案。<br>
> 输出应采用 JSON 格式，例如 `{"fact": [["s1", "p1", "o1"], ["s2", "p2", "o2"]]}`；如果没有相关事实，则返回空列表 `{"fact": []}`。<br>
> 回答的准确性至关重要，因为它将直接影响这些高级利益相关者作出的决定。你只能使用候选列表中的事实，不得生成新事实。关键决策的未来取决于你准确过滤并呈现相关信息的能力。

**演示 1**

- 问题：Imperial River（Florida）和 Amaradia（Dolj）是否都位于同一个国家？
- 过滤前事实：`{"fact": [["imperial river", "位于", "florida"], ["imperial river", "是……的一条河流", "united states"], ["imperial river", "可能指", "south america"], ["amaradia", "流经", "ro ia de amaradia"], ["imperial river", "可能指", "united states"]]}`
- 过滤后事实：`{"fact": [["imperial river", "位于", "florida"], ["imperial river", "是……的一条河流", "united states"], ["amaradia", "流经", "ro ia de amaradia"]]}`

**演示 2**

- 问题：电影 The Ancestor 的导演生日是什么时候？
- 过滤前事实：`{"fact": [["jean jacques annaud", "出生于", "1 october 1943"], ["tsui hark", "出生于", "15 february 1950"], ["pablo trapero", "出生于", "4 october 1971"], ["the ancestor", "导演是", "guido brignone"], ["benh zeitlin", "出生于", "october 14 1982"]]}`
- 过滤后事实：`{"fact": [["the ancestor", "导演是", "guido brignone"]]}`

**演示 3**

- 问题：Teafuone 所在国家位于哪个地理区域？
- 过滤前事实：`{"fact": [["teafuaniua", "位于", "east"], ["motuloa", "位于……之间", "teafuaniua"], ["motuloa", "位于……之间", "teafuanonu"], ["teafuone", "是", "islet"], ["teafuone", "位于", "nukufetau"]]}`
- 过滤后事实：`{"fact": [["teafuone", "是", "islet"], ["teafuone", "位于", "nukufetau"]]}`

**演示 4**

- 问题：电影 S.O.B. (Film) 的导演何时去世？
- 过滤前事实：`{"fact": [["allan dwan", "逝世于", "28 december 1981"], ["s o b", "编剧兼导演是", "blake edwards"], ["robert aldrich", "逝世于", "december 5 1983"], ["robert siodmak", "逝世于", "10 march 1973"], ["bernardo bertolucci", "逝世于", "26 november 2018"]]}`
- 过滤后事实：`{"fact": [["s o b", "编剧兼导演是", "blake edwards"]]}`

**演示 5**

- 问题：Gloria (1980 Film) 和 A New Life (Film) 两部电影的导演是否来自同一个国家？
- 过滤前事实：`{"fact": [["sebasti n lelio watt", "因执导……广受好评", "gloria"], ["gloria", "是", "1980 american thriller crime drama film"], ["a brand new life", "导演是", "ounie lecomte"], ["gloria", "编剧兼导演是", "john cassavetes"], ["a new life", "导演是", "alan alda"]]}`
- 过滤后事实：`{"fact": [["gloria", "是", "1980 american thriller crime drama film"], ["gloria", "编剧兼导演是", "john cassavetes"], ["a new life", "导演是", "alan alda"]]}`

**演示 6**

- 问题：电影 The Old Guard (1960 Film) 的导演逝世日期是什么？
- 过滤前事实：`{"fact": [["the old guard", "是", "1960 french comedy film"], ["gilles grangier", "执导", "the old guard"], ["the old guard", "导演是", "gilles grangier"], ["the old fritz", "导演是", "gerhard lamprecht"], ["oswald albert mitchell", "执导", "old mother riley series of films"]]}`
- 过滤后事实：`{"fact": [["the old guard", "是", "1960 french comedy film"], ["gilles grangier", "执导", "the old guard"], ["the old guard", "导演是", "gilles grangier"]]}`

**演示 7**

- 问题：电影 Aulad (1968 Film) 的作曲家生日是什么时候？
- 过滤前事实：`{"fact": [["aulad", "音乐作曲者为", "chitragupta shrivastava"], ["aadmi sadak ka", "音乐作者为", "ravi"], ["ravi shankar sharma", "曾为……作曲", "hindi films"], ["gulzar", "出生于", "18 august 1934"], ["aulad", "是", "1968 hindi language drama film"]]}`
- 过滤后事实：`{"fact": [["aulad", "音乐作曲者为", "chitragupta shrivastava"], ["aulad", "是", "1968 hindi language drama film"]]}`

> **输入格式：**<br>
> 问题：`{}`<br>
> 过滤前事实：`{}`<br>
> 过滤后事实：`{}`

**图 4：用于三元组过滤（识别记忆）的 LLM 提示词。**

### B 流程示例

图 5 展示 HippoRAG 2 在线检索的一个流程示例，包括查询到三元组、三元组过滤，以及使用种子节点执行 PPR。带星号的说明表示 PPR 返回的高排名节点已突出显示，带双星号的说明表示该问题的最终答案已突出显示。

**问题：** Erik Hort 的出生地属于哪个县？

**查询到三元组：**

- (`Erik Hort`, `出生于`, `Montebello`)
- (`Erik Hort`, `出生于`, `New York`)
- (`Erik Hort`, `是`, `American`)
- (`Erik Hort`, `出生日期`, `February 16, 1987`)
- (`Erik Hort`, `是`, `Soccer player`)

**过滤后的三元组：**

- (`Erik Hort`, `出生于`, `Montebello`)
- (`Erik Hort`, `出生于`, `New York`)

**PPR 种子节点：**

- 种子短语节点：(`Montebello`, 1.0)、(`Erik Hort`, 0.995)、(`New York`, 0.989)
- 种子段落节点（标题）：(`Erik Hort`, 0.05)、(`Horton Park (Saint Paul, Minnesota)`, 0.031)、(`Hertfordshire`, 0.028)、……

**返回的前列段落：**

1. **Erik Hort。** Erik Hort（1987 年 2 月 16 日出生于纽约州 Montebello）是一名美国足球运动员，目前为自由球员。
2. **Horton Park (Saint Paul, Minnesota)。** Horton Park 是美国明尼苏达州 Saint Paul 的一座小型植物园，主要以树种繁多闻名，现已成为 Saint Paul Midway 社区的象征。
3. **Montebello, New York。** Montebello（意大利语意为“美丽的山”）是美国纽约州 Rockland County 的 Ramapo 镇辖下的一个建制村，位于 Suffern 以北、Hillburn 以东、Wesley Hills 以南、Airmont 以西。2010 年人口普查时人口为 4,526。最终答案是 **Rockland County**。
4. **Hertfordshire。** Hertfordshire 是紧邻伦敦北部的一个郡，属于英格兰东部地区——这一地区主要是统计单位。各区都有相当一部分人口通勤至伦敦金融城。其东邻 Essex，西邻 Buckinghamshire，北邻 Bedfordshire 和 Cambridgeshire。
5. **Hull County, Quebec。** Hull County 是加拿大 Quebec 的一个历史郡，以英格兰 East Yorkshire 的同名城镇 Hull（即 Kingston-upon-Hull）命名。它位于 Ottawa River 北岸，是 Québec 约 12 个历史地区之一 Outaouais 的一部分。

**图 5：HippoRAG 2 流程示例。**

### C 详细实验结果

本节给出使用闭源模型 GPT-4o-mini 时的问答性能和检索性能，以及更多评估指标，见表 8 和表 9。

**问答性能。** 如表 8 所示，使用 GPT-4o-mini 进行索引和问答阅读时，HippoRAG 2 在大多数数据集上始终取得有竞争力的 EM 和 F1，尤其在 MuSiQue 和 2Wiki 基准上领先。本文方法在 NarrativeQA 和 LV-Eval 任务上也表现更好。与强大的 NV-Embed-v2 检索器相比，HippoRAG 2 的 F1 相当或更高，尤其擅长知识泄漏更少的 LV-Eval 数据集。

**表 8：RAG 基准上的问答性能（EM / F1）。** “无检索”表示评估阅读器的参数知识。HippoRAG（及 HippoRAG 2）使用所标注的 LLM 执行 OpenIE（三元组过滤）和问答阅读。

| 阅读器 | 检索方法 | NQ | PopQA | MuSiQue | 2Wiki | HotpotQA | LV-Eval | NarrativeQA | 平均 |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Llama-3.3-70B-Instruct | 无 | 40.2 / 54.9 | 28.2 / 32.5 | 17.6 / 26.1 | 36.5 / 42.8 | 37.0 / 47.3 | 4.0 / 6.0 | 3.4 / 12.9 | 29.7 / 38.4 |
|  | Contriever | 45.0 / 58.9 | 41.6 / 53.1 | 24.0 / 31.3 | 38.1 / 41.9 | 51.3 / 62.3 | 5.7 / 8.1 | 6.5 / 19.7 | 37.4 / 46.9 |
|  | BM25 | 44.7 / 59.0 | 39.1 / 49.9 | 20.3 / 28.8 | 47.9 / 51.2 | 52.0 / 63.4 | 4.0 / 5.9 | 4.4 / 18.3 | 38.0 / 47.7 |
|  | GTR（T5-base） | 45.5 / 59.9 | <u>43.2 / 56.2</u> | 25.8 / 34.6 | 49.2 / 52.8 | 50.6 / 62.8 | 4.8 / 7.1 | 6.8 / 19.9 | 40.0 / 50.4 |
|  | GTE-Qwen2-7B-Instruct | 46.6 / <u>62.0</u> | **43.5 / 56.3** | 30.6 / 40.9 | 55.1 / 60.0 | 58.6 / 71.0 | 5.7 / 7.1 | 7.9 / 21.3 | 43.8 / 54.9 |
|  | GritLM-7B | 46.8 / 61.3 | 42.8 / 55.8 | 33.6 / 44.8 | 55.8 / 60.6 | 60.7 / 73.3 | <u>7.3 / 9.8</u> | 8.2 / 23.9 | 44.9 / 56.1 |
|  | NV-Embed-v2（7B） | <u>47.3</u> / 61.9 | 42.9 / 55.7 | <u>34.7 / 45.7</u> | <u>57.5</u> / 61.5 | **62.8** / <u>75.3</u> | <u>7.3 / 9.8</u> | **8.9** / <u>25.7</u> | <u>45.9 / 57.0</u> |
|  | RAPTOR | 36.9 / 50.7 | 43.1 / <u>56.2</u> | 20.7 / 28.9 | 47.3 / 52.1 | 56.8 / 69.5 | 2.4 / 5.0 | 5.1 / 21.4 | 38.1 / 48.8 |
|  | GraphRAG | 30.8 / 46.9 | 31.4 / 48.1 | 27.3 / 38.5 | 51.4 / 58.6 | 55.2 / 68.6 | 4.8 / 11.2 | 6.8 / 23.0 | 36.7 / 49.6 |
|  | LightRAG | 8.6 / 16.6 | 2.1 / 2.4 | 0.5 / 1.6 | 9.4 / 11.6 | 2.0 / 2.4 | 0.8 / 1.0 | 1.0 / 3.7 | 4.2 / 6.6 |
|  | HippoRAG | 43.0 / 55.3 | 42.7 / 55.9 | 26.2 / 35.1 | 65.0 / 71.8 | 52.6 / 63.5 | 6.5 / 8.4 | 4.4 / 16.3 | 42.8 / 53.1 |
|  | HippoRAG 2 | **48.6 / 63.3** | 42.9 / <u>56.2</u> | **37.2 / 48.6** | **65.0** / <u>71.0</u> | <u>62.7</u> / **75.5** | **9.7 / 12.9** | **8.9 / 25.9** | **48.0 / 59.8** |
| GPT-4o-mini | 无 | 35.2 / 52.7 | 16.1 / 22.7 | 11.2 / 22.0 | 30.2 / 36.3 | 28.6 / 41.0 | 3.2 / 5.0 | 2.7 / 14.1 | 22.6 / 33.1 |
|  | NV-Embed-v2（7B） | **43.5** / <u>59.9</u> | 41.7 / 55.8 | 32.8 / <u>46.0</u> | 54.4 / 60.8 | **57.3** / <u>71.0</u> | <u>7.3</u> / 10.0 | 5.1 / <u>24.2</u> | <u>42.9 / 55.7</u> |
|  | RAPTOR | 37.8 / 54.5 | 41.9 / 55.1 | 27.7 / 39.2 | 39.7 / 48.4 | 50.6 / 64.7 | 5.6 / 9.2 | 4.1 / 21.8 | 36.9 / 49.7 |
|  | GraphRAG | 38.0 / 55.5 | 30.7 / 51.3 | 27.0 / 42.0 | 45.7 / 61.0 | 51.4 / 67.6 | 4.9 / <u>11.0</u> | <u>5.4</u> / 20.9 | 36.0 / 52.6 |
|  | LightRAG | 2.8 / 15.4 | 1.9 / 14.8 | 2.0 / 9.3 | 2.5 / 12.1 | 9.9 / 20.2 | 0.9 / 5.0 | 1.0 / 9.0 | 3.6 / 13.9 |
|  | HippoRAG | 37.2 / 52.2 | **42.5 / 56.2** | 24.0 / 35.9 | <u>59.4 / 67.3</u> | 46.3 / 60.0 | 4.8 / 7.6 | 2.1 / 16.1 | 38.9 / 51.2 |
|  | HippoRAG 2 | <u>43.4</u> / **60.0** | 41.7 / 55.7 | **35.0 / 49.3** | **60.5 / 69.7** | <u>56.3</u> / **71.1** | **10.5 / 14.0** | **5.8 / 25.2** | **44.3 / 58.1** |

**检索性能。** 如表 9 所示，HippoRAG 2 在 recall@2 上的提升趋势与 recall@5 相似。

**表 9：RAG 基准上的段落 recall@2 / recall@5。** $^*$ 表示原论文报告的结果；HippoRAG 结果则由我们使用对齐的 LLM 和检索器复现。

| 类别 | 检索方法 | NQ | PopQA | MuSiQue | 2Wiki | HotpotQA | 平均 |
|---|---|---:|---:|---:|---:|---:|---:|
| 简单基线 | Contriever | 29.1 / 54.6 | 27.0 / 43.2 | 34.8 / 46.6 | 46.6 / 57.5 | 58.4 / 75.3 | 39.2 / 55.4 |
|  | BM25 | 28.2 / 56.1 | 24.0 / 35.7 | 32.4 / 43.5 | 55.3 / 65.3 | 57.3 / 74.8 | 39.4 / 55.1 |
|  | GTR（T5-base） | 35.0 / 63.4 | 40.1 / 49.4 | 37.4 / 49.1 | 60.2 / 67.9 | 59.3 / 73.9 | 46.4 / 60.7 |
| 大型嵌入模型 | GTE-Qwen2-7B-Instruct | 44.7 / 74.3 | **47.7** / 50.6 | 48.1 / 63.6 | 66.7 / 74.8 | 75.8 / 89.1 | 56.6 / 70.5 |
|  | GritLM-7B | **46.2** / <u>76.6</u> | 44.0 / 50.1 | 49.7 / 65.9 | 67.3 / 76.0 | 79.2 / 92.4 | 57.3 / 72.2 |
|  | NV-Embed-v2（7B） | 45.3 / 75.4 | 45.3 / 51.0 | 52.7 / 69.7 | 67.1 / 76.5 | **84.1** / <u>94.5</u> | 58.9 / 73.4 |
| 结构增强 RAG | RAPTOR（GPT-4o-mini） | 40.5 / 69.4 | 37.2 / 48.1 | 49.1 / 61.0 | 58.4 / 66.0 | 78.6 / 90.2 | 52.8 / 67.0 |
|  | RAPTOR（Llama-3.3-70B-Instruct） | 40.3 / 68.3 | 40.2 / 48.7 | 47.0 / 57.8 | 58.3 / 66.2 | 76.8 / 86.9 | 52.5 / 65.6 |
|  | HippoRAG$^*$ | - | - | 40.9 / 51.9 | 70.7 / 89.1 | 60.5 / 77.7 | - |
|  | HippoRAG（GPT-4o-mini） | 21.6 / 45.1 | 36.5 / 52.2 | 41.8 / 52.4 | 68.4 / 87.0 | 60.1 / 78.5 | 45.7 / 63.0 |
|  | HippoRAG（Llama-3.3-70B-Instruct） | 21.3 / 44.4 | 40.0 / 53.8 | 41.2 / 53.2 | 71.9 / 90.4 | 60.4 / 77.3 | 47.0 / 63.8 |
|  | HippoRAG 2（GPT-4o-mini） | 44.4 / 76.4 | 43.5 / <u>52.2</u> | <u>53.5 / 74.2</u> | <u>74.6 / 90.2</u> | 80.5 / <u>95.7</u> | <u>59.3 / 77.7</u> |
|  | HippoRAG 2（Llama-3.3-70B-Instruct） | <u>45.6</u> / **78.0** | 43.9 / 51.7 | **56.1 / 74.7** | **76.2 / 90.4** | <u>83.5</u> / **96.3** | **61.1 / 78.2** |

### D 图统计

表 10 给出了使用 Llama-3.3-70B-Instruct 或 GPT-4o-mini 执行 OpenIE 时的知识图谱统计。

**表 10：使用不同 LLM 执行 OpenIE 时的知识图谱统计。** 节点和三元组按唯一值计数。

| LLM | 统计项 | NQ | PopQA | MuSiQue | 2Wiki | HotpotQA | LV-Eval | NarrativeQA |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| Llama-3.3-70B-Instruct | 短语节点数 | 68,375 | 76,539 | 85,288 | 44,004 | 81,200 | 175,195 | 9,224 |
|  | 段落节点数 | 9,633 | 8,676 | 11,656 | 6,119 | 9,811 | 22,849 | 4,111 |
|  | 节点总数 | 78,008 | 85,215 | 96,944 | 50,123 | 91,011 | 198,044 | 13,335 |
|  | 抽取边数 | 125,777 | 124,579 | 140,830 | 68,881 | 130,058 | 314,324 | 26,208 |
|  | 同义边数 | 899,031 | 845,014 | 1,125,951 | 593,298 | 994,187 | 2,674,833 | 72,494 |
|  | 上下文边数 | 126,757 | 118,909 | 132,586 | 64,132 | 122,437 | 375,424 | 33,395 |
|  | 边总数 | 1,151,565 | 1,088,502 | 1,399,367 | 726,311 | 1,246,682 | 3,364,581 | 132,097 |
| GPT-4o-mini | 短语节点数 | 86,904 | 85,744 | 101,641 | 49,544 | 95,105 | 217,085 | 15,365 |
|  | 段落节点数 | 9,633 | 8,676 | 11,656 | 6,119 | 9,811 | 22,849 | 4,111 |
|  | 节点总数 | 96,537 | 94,420 | 113,297 | 55,663 | 104,916 | 239,934 | 19,476 |
|  | 抽取边数 | 114,900 | 108,989 | 125,903 | 62,626 | 119,630 | 303,491 | 24,373 |
|  | 同义边数 | 1,094,651 | 901,528 | 1,304,605 | 715,763 | 1,126,501 | 3,268,084 | 14,075 |
|  | 上下文边数 | 142,419 | 127,568 | 146,293 | 68,348 | 133,220 | 404,210 | 38,632 |
|  | 边总数 | 1,351,970 | 494,082 | 1,576,801 | 846,737 | 1,379,351 | 3,975,785 | 77,080 |

> **译注：** 表中 GPT-4o-mini / PopQA 的“边总数”原文为 494,082，但同表三类边数之和为 1,138,085；译文不擅自修正，保留原文数值。

### E 错误分析

我们对 HippoRAG 2 生成的 100 个 recall@5 小于 1.0 的样本进行错误分析。其中，2 跳、3 跳和 4 跳问题分别占 26%、41% 和 33%。三元组过滤与图搜索算法是两大主要错误来源。

**识别记忆。** 在 7% 的样本中，三元组过滤前，由查询到三元组阶段获得的短语没有任何一个能与支持文档中的短语匹配；过滤后，这一比例升至 26%。经过三元组过滤步骤，8% 的样本中，三元组内能与支持段落短语匹配的短语占比下降。例如，表 11 的第一个案例在三元组过滤后得到空列表，所有相关短语都被消除。此外，18% 的样本在过滤后不剩任何三元组。尽管这不一定是过滤错误，却说明尝试链接至三元组的过程失败；此时 HippoRAG 2 会直接改用稠密检索结果。总体而言，尽管识别记忆是一个必要组件，三元组过滤器的精度仍有改进空间。

**图构建。** 图构建很难评估，但我们发现，只有 2% 的样本在已链接节点的一跳邻域中完全不含来自支持段落的短语。鉴于采用了稠密-稀疏整合，可以认为所构建的图总体上包含了大多数潜在可利用信息。

**个性化 PageRank。** 在 50% 的样本中，至少一半已链接的短语节点出现在支持文档中，但由于图搜索组件，最终结果仍不理想。例如，在表 11 的第二个案例中，识别记忆从查询中识别出关键短语“Philippe, Duke of Orléans”，但图搜索未能在检索到的前 5 个段落中返回完美结果。[^译注3]

[^译注3]: 原文对第二例的叙述与表 11 所列“查询到三元组”内容明显不一致：表中实际出现的是 Bank of America / FleetBoston Financial，而非 Philippe。译文忠实保留二者并提示该矛盾。

**表 11：MuSiQue 上段落 recall@5 小于 1.0 的两个示例。**

**示例 1**

- 查询：那位曾试图改革并处理 Bernhard Lichtenberg 所属宗教的人，在去世前曾就圣母敬礼布道；该布道所在的地区位于哪里？
- 答案：Saxony-Anhalt
- 支持段落（标题）：1. Mary, mother of Jesus；2. Reformation；3. Wittenberg (district)；4. Bernhard Lichtenberg
- 检索段落（标题）：1. **Bernhard Lichtenberg**；2. **Mary, mother of Jesus**；3. Ambroise-Marie Carré；4. **Reformation**；5. Henry Scott Holland（recall@5 为 0.75）
- 查询到三元组（前 5）：(`Bernhard Lichtenberg`, `是`, `Roman Catholic Priest`)；(`Bernhard Lichtenberg`, `由……宣福`, `Catholic Church`)；(`Bernhard Lichtenberg`, `逝世于`, `5 November 1943`)；(`Catholic Church`, `为……宣福`, `Bernhard Lichtenberg`)；(`Bernhard Lichtenberg`, `是`, `Theologian`)。以上所有主语和宾语均出现在支持段落中。
- 过滤后的三元组：空。

**示例 2**

- 查询：Philippe, Duke of Orléans 的外祖母是谁？
- 答案：Marie de’ Medici
- 支持段落（标题）：1. Philippe I, Duke of Orléans；2. Leonora Dori
- 检索段落（标题）：1. **Philippe I, Duke of Orléans**；2. Louise Élisabeth d’Orléans；3. Philip III of Spain；4. Anna of Lorraine；5. Louis Philippe I（recall@5 为 0.5）
- 查询到三元组（前 5）：(`Bank of America`, `收购`, `Fleetboston Financial`)；(`Fleetboston Financial`, `被……收购`, `Bank of America`)；(`Bank of America`, `收购`, `Fleetboston Financial`)；(`Bank of America`, `宣布收购`, `Fleetboston Financial`)；(`Bank of America`, `与……合并`, `Fleetboston Financial`)。原文称以上所有主语和宾语均出现在支持段落中。
- 过滤后的三元组：(`Bank of America`, `收购`, `Fleetboston Financial`)；(`Fleetboston Financial`, `被……收购`, `Bank of America`)。原文同样称以上所有主语和宾语均出现在支持段落中。

### F 成本与效率

部署 LLM 时，我们在一台配备 4 张 NVIDIA H100 GPU 的机器上运行 Llama-3.3-70B-Instruct，并通过 vLLM（Kwon et al., 2023）采用张量并行。

为了与基线进行详细比较，我们在 MuSiQue 语料库（11,656 个文档）上使用 Llama-3.3-70B-Instruct 进行索引和问答时，记录计算资源用量，包括词元数、索引时间、每个查询的耗时以及 GPU 内存，并在表 12 中将 HippoRAG 2 与 NV-Embed-v2（Lee et al., 2025）、RAPTOR（Sarthi et al., 2024）、LightRAG（Guo et al., 2024）、HippoRAG（Gutiérrez et al., 2024）和 GraphRAG（Edge et al., 2024）进行比较。内存需求不计模型权重所占内存，因为所有系统共享该部分。

HippoRAG 2 不仅在问答和检索性能上优于这些 RAG 方法，而且使用的词元远少于 LightRAG 和 GraphRAG。就时间而言，HippoRAG 2 比 GraphRAG 和 LightRAG 高效得多，只略逊于 RAPTOR 和 HippoRAG。HippoRAG 2 使用事实嵌入，确实使其内存需求高于基线；但鉴于本方法带来的性能收益，我们认为这是可以接受的权衡。此外，尽管所有方法在时间和内存效率上都落后于标准 RAG，HippoRAG 2 却是唯一大幅优于这一强基线的方法。

**表 12：在 MuSiQue 语料库（11,656 个段落）上，各 RAG 基线的计算资源需求。** 指标包括索引词元数、索引时间、每个查询的耗时和问答期间的 GPU 内存；括号内为相对于 HippoRAG 2 指标（100%）的百分比。

| 指标 | NV-Embed-v2 | RAPTOR | LightRAG | GraphRAG | HippoRAG | HippoRAG 2 |
|---|---:|---:|---:|---:|---:|---:|
| 输入词元 | - | 1.7M（18.5%） | 68.5M（744.6%） | 115.5M（1255.4%） | 9.2M（100.0%） | 9.2M（100.0%） |
| 输出词元 | - | 0.2M（6.7%） | 18.3M（610.0%） | 36.1M（1203.3%） | 3.0M（100.0%） | 3.0M（100.0%） |
| 索引时间（分钟） | 12.1（12.3%） | 100.5（101.0%） | 235.0（236.2%） | 277.0（278.4%） | 57.5（57.7%） | 99.5（100.0%） |
| 每查询问答耗时（秒） | 0.3（25.0%） | 0.6（50.0%） | 13.3（1008.3%） | 10.7（891.7%） | 0.9（75.0%） | 1.2（100.0%） |
| 问答 GPU 内存（GB） | 1.7（17.2%） | 1.4（14.1%） | 4.5（45.5%） | 3.7（37.4%） | 6.0（60.6%） | 9.9（100.0%） |

> **译注：** 表 12 中若干时间百分比与绝对值及 HippoRAG 2 基准值并不严格一致（如 RAPTOR 索引时间 100.5 分钟对应 101.0%，但 HippoRAG 2 为 99.5 分钟）；译文保留原表。

### G 实现细节与超参数

#### G.1 HippoRAG 2

本节详细说明 HippoRAG 2 所用的 PPR 初始化过程。核心目标是确定 PPR 搜索的种子节点并分配恰当的重置概率，以保证检索过程有效。

**种子节点选择。** PPR 搜索的种子节点分为短语节点和段落节点两类。下述嵌入模型给出的所有分数都使用归一化嵌入计算。

1. **短语节点：** 从识别记忆组件得到的过滤后三元组中选择短语节点。如果识别记忆给出空的三元组列表、没有可用的短语节点，HippoRAG 2 不执行图搜索，而是直接使用嵌入模型返回排名靠前的段落。否则，最多保留 5 个短语节点作为种子节点；每个短语节点的排序分数，是它所出现的全部过滤后三元组分数的平均值。
2. **段落节点：** 首先使用基于嵌入的相似度为每个段落节点评分，再按下述方式处理这些分数。所有段落节点都会作为种子节点；我们发现，相比只关注排名靠前的段落，激活范围更广的潜在段落更有利于发现多跳推理链上的段落。

**重置概率分配。** 确定种子节点后，我们分配重置概率，以控制随机游走期间 PPR 返回这些节点的可能性。规则如下：1）短语节点直接以其排序分数作为重置概率；2）段落节点获得与其嵌入相似度分数成正比的重置概率。为平衡短语节点与段落节点的影响，我们为段落节点分数应用权重因子，即将其乘以第 6.2 节讨论的权重因子。这可确保段落节点和短语节点恰当地参与检索过程。

**PPR 执行与段落排序。** 完成种子节点及其重置概率的初始化后，我们在构建的图上运行 PPR。段落的最终排序由段落节点的 PageRank 分数决定，排名最高的段落随后作为下游问答阅读过程的输入。我们使用 `python-igraph` 库[^2] 管理 KG 并运行 PPR 算法。

[^2]: <https://python.igraph.org/en/stable/>

通过在 PPR 初始化中同时纳入短语节点与段落节点，本方法能更有效地检索相关段落，尤其适用于多跳推理任务。

**超参数。** 我们在 MuSiQue 训练数据的 100 个样本上进行超参数调优。超参数见表 13。

**表 13：HippoRAG 2 的超参数设置。**

| 超参数 | 值 |
|---|---:|
| 同义词阈值 | 0.8 |
| PPR 阻尼因子 | 0.5 |
| 温度 | 0.0 |

#### G.2 对比方法

我们使用 PyTorch（Paszke et al., 2019）和 HuggingFace（Wolf et al., 2019）实现稠密检索器，使用 BM25s（Lù, 2024）实现 BM25。对于 GraphRAG（Edge et al., 2024）和 LightRAG（Guo et al., 2024），我们采用其默认超参数和提示词。为保证评估一致，我们使用 HippoRAG 2 从 HippoRAG（Gutiérrez et al., 2024）继承的同一问答提示词，对 GraphRAG 和 LightRAG 的原始回答进行改写。

**超参数。** GraphRAG 和 LightRAG 的索引过程保持默认超参数。问答阶段则使用与附录 G.1 相同的 100 个样本进行超参数调优。

**表 14：GraphRAG 和 LightRAG 的超参数设置。**

| 超参数 | GraphRAG | LightRAG |
|---|---|---|
| 模式 | Local | Local |
| 回答类型 | 短语 | 短语 |
| 用于问答的 top-$k$ 短语数 | 60 | 60 |
| 分块词元数 | 1,200 | 1,200 |
| 分块重叠词元数 | 100 | 100 |
| 社区报告最大长度 | 2,000 | - |
| 最大输入长度 | 8,000 | - |
| 最大聚类大小 | 10 | - |
| 实体摘要最大词元数 | - | 500 |

## Sources

- `papers/agent-memory/From RAG to Memory - Non-Parametric Continual Learning for Large Language Models/From RAG to Memory - Non-Parametric Continual Learning for Large Language Models.pdf`
