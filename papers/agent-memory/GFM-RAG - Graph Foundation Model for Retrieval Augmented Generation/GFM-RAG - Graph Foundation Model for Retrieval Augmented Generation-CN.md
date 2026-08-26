# GFM-RAG：用于检索增强生成的图基础模型

## 译文说明

本译文依据本地 PDF 全文翻译，保留原文的公式编号、表格数值、图表编号、引用编号、模型名与数据集名。参考文献保留原文，以便准确检索。原文中少量明显拼写问题已在不改变含义的前提下校正；彼此不一致的数值或符号则按原文保留，并在相应位置以译注标明。

Linhao Luo$^{1*}$，Zicheng Zhao$^{2*}$，Gholamreza Haffari$^1$，Chen Gong$^3$，Dinh Phung$^1$，Shirui Pan$^{4\dagger}$

$^1$ Monash University，$^2$ Nanjing University of Science and Technology，  
$^3$ Shanghai Jiao Tong University，$^4$ Griffith University

{Linhao.Luo,gholamreza.haffari,dinh.phung}@monash.edu，  
zicheng.zhao@njust.edu.cn，chen.gong@sjtu.edu.cn，s.pan@griffith.edu.au

项目主页：https://rmanluo.github.io/gfm-rag

arXiv:2502.01113v3 [cs.IR]，2025 年 12 月 11 日

$^*$ 同等贡献。  
$^\dagger$ 通讯作者。

第 39 届神经信息处理系统大会（NeurIPS 2025）。

## 摘要

检索增强生成（RAG）已被证明能够有效地将知识整合到大型语言模型（LLM）中。然而，传统 RAG 难以捕捉不同知识片段之间的复杂关系，因而限制了其在需要整合多个来源知识的复杂推理任务中的表现。近期，图增强检索增强生成（GraphRAG）通过构建图结构来显式建模这些关系，从而实现更有效、更高效的检索器。尽管如此，图结构中的噪声和不完整性仍会阻碍其性能。为解决这一问题，我们提出 GFM-RAG，一种面向检索增强生成的新型图基础模型（GFM）。GFM-RAG 由一种创新的图神经网络驱动，该网络在图结构上进行推理，以捕捉复杂的查询-知识关系。这个拥有 8M 参数的 GFM 在大规模数据集上经历两阶段训练；这些数据集包含 60 个知识图谱、超过 14M 个三元组以及 700k 篇文档。由此，GFM-RAG 获得了出色的性能与泛化能力，成为首个无需任何领域特定微调、便可在未见数据集上执行检索的图基础模型。我们在三个多跳问答数据集和七个领域特定 RAG 数据集上开展了广泛实验。结果表明，GFM-RAG 在保持高效率的同时取得了最先进的性能，并且符合神经缩放定律，展现出进一步提升的潜力。

## 1 引言

大型语言模型（LLM）近期取得的进展 [47, 42, 70] 极大地推动了自然语言处理的发展，并使其成为通用人工智能（AGI）的基础模型。尽管 LLM 具备卓越的推理能力 [48]，但仍无法访问实时信息，也缺少预训练语料库之外的领域特定知识。为克服这些限制，检索增强生成（RAG）[12] 已成为向静态 LLM 添加新知识的一种流行范式：它将相关文档检索到 LLM 的生成上下文中。

现有 RAG 方法通常彼此独立地检索文档，因此难以捕捉知识片段之间的复杂关系 [30, 5, 43]。这一局限会妨碍 LLM 跨越文档边界整合知识，尤其是在多跳推理任务 [72, 63]，以及法律裁判 [28]、医学诊断 [25] 等需要基于多个来源进行推理的现实应用中。虽然近期方法已将检索过程扩展为多个步骤，并引入 LLM 推理，但由于需要借助 LLM 反复进行检索与推理，它们仍面临很高的计算成本 [64, 59, 26]。

近期，GraphRAG [51, 17] 作为一种新颖方案崭露头角：它构建图结构，以显式建模知识之间错综复杂的关系。这使得研究者能够开发图增强检索器，利用图来识别相关信息。图的结构特性使 GraphRAG 能够捕捉全局上下文以及文档间的依赖关系，从而显著改善跨多个来源的推理 [9]。HippoRAG [16] 等方法采用个性化 PageRank 算法，借助图来定位相关知识，从而增强检索。然而，这些算法仅依赖图结构，而图结构往往含有噪声或并不完整，因而限制了整体性能。另一些方法 [41, 18] 将图神经网络（GNN）引入检索过程。得益于 GNN 在图上强大的多跳推理能力 [73]，这些方法展现出令人瞩目的性能。尽管如此，它们的泛化能力仍然有限，因为面对新数据集时必须从头训练。

如今，寻找一种能够在不同数据集之间迁移并泛化的基础 GNN 模型，已经成为活跃的研究课题。理想情况下，基础 GNN 或图基础模型（GFM）能够从大规模训练中获益，并在不同图之间实现泛化 [40, 37]。已有研究尝试寻找可迁移的图词元（例如模体、子树和关系图）[11, 66, 68]，使其能够在不同图之间共享，以用于 GFM 设计。然而，这些方法主要关注图相关任务（例如节点分类和链接预测），而如何设计 GFM 来增强 LLM 的推理能力仍未得到探索。

为填补这一空白，本文提出一种有效、高效且通用的检索增强生成图基础模型（GFM-RAG），以增强 LLM 的推理能力。如图 1 所示，我们从每个数据集的文档中创建 KG-index（知识图谱索引）。KG-index 由相互连接、且指向原始文档的事实三元组构成，可作为跨多个来源的结构化知识索引，从而加强复杂推理任务对多样化知识的整合 [16]。随后，我们提出由查询依赖型 GNN 驱动的图基础模型检索器（GFM 检索器）；该 GNN 能在语义与图结构构成的统一、可迁移空间中捕捉复杂的查询-知识关系。通过多层消息传递，GFM 检索器能够在单个步骤内实现高效的多跳检索，超越以往的多步骤方法。拥有 8M 参数的 GFM 检索器经历两阶段训练：首先进行自监督知识图谱补全预训练，随后在大规模数据集上进行有监督文档检索微调；这些数据集包括 60 个知识图谱、超过 14M 个三元组以及 700k 篇文档。这种大规模训练确保 GFM 检索器能够泛化到未见数据集，无需进一步训练即可应用。

> **图 1 内文字：** 文档；KG-index；文档排序器；检索到的文档；查询（Q）；GFM 检索器；LLM；答案（A）。

**图 1：GFM-RAG 的总体框架。**

实验结果表明，GFM-RAG 在三个多跳问答数据集上均取得最先进的性能，证明了其在多跳推理中的有效性与效率。它还能够在生物医学、客户服务和通用知识等不同领域的七个 RAG 数据集上良好泛化，而无需额外训练。此外，GFM-RAG 遵循神经缩放定律 [19]，其性能可受益于训练数据规模和模型规模的增长，这凸显了它作为基础模型在未来持续改进的潜力。本文的主要贡献如下：

- 我们提出用于检索增强生成的图基础模型 GFM-RAG。它由一种新颖的查询依赖型 GNN 驱动，能够在单个步骤内完成高效的多跳检索。

- 我们训练了一个拥有 8M 参数的大规模模型，使其成为首个可直接应用于多种未见数据集、执行检索增强生成的图基础模型（GFM）。

- 我们在三个多跳问答数据集和七个领域特定 RAG 数据集上评估 GFM-RAG，并在所有数据集上取得最先进的性能，证明了其有效性、效率、泛化能力，以及作为基础模型进一步增强的潜力。

## 2 相关工作

**检索增强生成（RAG）** [12] 通过检索相关文档来辅助 LLM 生成，为大型语言模型（LLM）整合外部知识提供了一种有效途径。早期工作采用预训练的稠密嵌入模型，将文档编码为彼此独立的向量 [30, 5, 34, 43]，随后通过计算这些向量与查询的相似度来进行检索。这些方法虽然高效且具备泛化能力，却难以捕捉复杂的文档关系。后续研究探索了多步骤检索，由 LLM 引导迭代过程，对多篇文档进行检索与推理 [64, 24, 58]。然而，这种方法的计算成本很高。

**GraphRAG** [51, 17] 是一种新颖方法，它通过构建图来显式建模知识之间的复杂关系，从而促进全面的检索与推理。早期研究侧重于从 WikiData [65] 和 Freebase [3] 等现有知识图谱（KG）中检索信息，即识别相关事实或推理路径 [33, 38, 50]。近期研究将文档与 KG 相结合，以提升知识覆盖范围和检索效果 [9, 35]。这些方法从文档中构建图结构，帮助识别与 LLM 生成相关的内容 [8]。在图的基础上，LightRAG [15] 将图结构融入文本索引与检索，从而实现对实体及其关系的高效检索。HippoRAG [16] 使用个性化 PageRank 算法，通过图来定位相关知识，从而增强多跳检索。然而，图结构可能包含噪声且不完整，导致性能并非最优。将 GNN 引入 GraphRAG 的尝试 [41, 18] 已取得令人瞩目的结果，这得益于 GNN 在处理不完整图时具备多跳图推理能力 [73]。尽管如此，由于缺少图基础模型，这些方法的泛化能力仍然受限。

**图基础模型（GFM）** 旨在成为一种能够泛化到多种数据集的大规模模型 [40, 37]。设计 GFM 的主要挑战，是识别能够捕捉不同图数据之间不变性的图词元。例如，ULTRA [11] 利用知识图谱（KG）中的四种基本关系交互，创建了一个用于链接预测、参数量为 0.2M 的 GFM。OpenGraph [68] 开发了一种图分词器，将图转换为统一的节点词元表示，从而构建可用于链接预测、节点分类等任务的类 Transformer GFM。GFT [66] 引入可迁移的树词表，构建了一个在图学习的多种任务和领域中均展现出有效性的 GFM。尽管这些尝试取得了成功，但大多数方法主要关注传统图相关任务；类 Transformer GFM [61, 60] 则难以处理大规模图并捕捉逻辑关联 [52]。如何设计基于 GNN 的 GFM 来增强 LLM 的推理能力，仍是一个开放问题。

## 3 方法

所提出的 GFM-RAG 本质上实现了一种 GraphRAG 范式：从文档构建图，并使用图增强检索器检索相关文档。

**GFM-RAG 概览。** 给定文档集合 $\mathcal{D}=\{D_1,D_2,\ldots,D_{|\mathcal{D}|}\}$，我们构建知识图谱 $\mathcal{G}=\{(e,r,e')\in\mathcal{E}\times\mathcal{R}\times\mathcal{E}\}$，其中 $e,e'\in\mathcal{E}$ 和 $r\in\mathcal{R}$ 分别表示从 $\mathcal{D}$ 中提取的实体集合与关系集合。对于用户查询 $q$，我们的目标是设计一个图增强检索器，利用知识图谱 $\mathcal{G}$ 从 $\mathcal{D}$ 中获得相关文档。整个 GFM-RAG 过程可表示为：

$$
\mathcal{G}=\operatorname{KG-index}(\mathcal{D}), \tag{1}
$$

$$
\mathcal{D}^{K}=\operatorname{GFM-Retriever}(q,\mathcal{D},\mathcal{G}), \tag{2}
$$

$$
a=\operatorname{LLM}(q,\mathcal{D}^{K}). \tag{3}
$$

第一步，$\operatorname{KG-index}(\cdot)$ 从文档语料库 $\mathcal{D}$ 构建知识图谱索引 $\mathcal{G}$；随后，由我们提出、在大规模数据集上完成预训练的图基础模型检索器（GFM-Retriever），基于任意用户查询 $q$ 和知识图谱索引 $\mathcal{G}$ 检索 top-$K$ 篇文档。检索到的文档 $\mathcal{D}^{K}$ 与查询 $q$ 随后被一同输入大型语言模型（LLM），以生成最终答案 $a$。GFM-RAG 的这三个主要组件如图 2 所示，下面将分别详述。

> **图 2 内文字：**  
> **用于检索增强生成的图基础模型：** 文档 1：“Barack Obama（1961 年 8 月 4 日出生于 Honolulu）是一名 American politician……他与 Michelle Obama 结婚。”；文档语料库；① KG-index 构建；知识图谱；文档 2：“Honolulu 是美国 Hawaii 州的首府和人口最多的城市，位于 Pacific Ocean。”；实体到文档的倒排索引。  
> **查询与图推理：** 查询嵌入；“问：Barack Obama 是哪个国家的政治家？”；② 查询初始化；实体 Barack Obama、Michelle Obama、Honolulu、USA、Washington, D.C.；关系 `born_in`、`politician_of`、`married_to`、`live_in`、`city_of`、`capital_of`；实体相关性分数 0.2、0.9、0.1、0.6；③ 查询依赖型消息传递；图基础 GNN 模型。  
> **文档排序与生成：** 检索到的文档；文档排序器；④ 文档排序；Q；LLM；“答：USA”；⑤ LLM 生成。文档 3：“Michelle Obama 于 2009 至 2017 年担任 United States 第一夫人，并与 Barack Obama 结婚。”；文档 4：“USA 是一个主要位于 North America 的国家。它是由 50 个州和联邦首都 Washington, D.C. 所在的联邦区组成的联邦。”  
> **阶段 1：自监督知识图谱补全预训练：** 训练知识图谱；三元组；“问：（Barack Obama，`born_in`，?）”；图基础 GNN 模型；目标实体；“答：Honolulu”。  
> **阶段 2：有监督文档检索微调：** 训练问题-文档对与 KG-index；问题；“问：Barack Obama 出生在哪里？”；图基础 GNN 模型；目标实体；支持文档；“答：Honolulu，USA”。

**图 2：GFM-RAG 的详细框架以及图基础模型的训练过程。** GFM-RAG 包含三个主要组件：A. KG-index 构建，即从文档语料库构建知识图谱索引（①）；B. 图基础模型检索器（GFM 检索器），它在大规模数据集上进行预训练，并可根据任意用户查询和 KG-index 检索文档（②③）；C. 文档排序与答案生成，即对检索到的文档进行排序并生成最终答案（④⑤）。

### 3.1 KG-index 构建

传统的基于嵌入的索引方法将文档编码为彼此独立的向量 [30, 5, 43]，因而在建模文档之间的关系时存在局限。另一方面，知识图谱（KG）能够显式捕捉数百万条事实之间的关系，可为跨越多篇文档的知识提供结构化索引 [9, 16]。KG-index 的结构特性与人类海马体记忆索引理论 [62] 高度契合：KG-index 如同人工海马体，存储不同知识记忆之间的关联，从而加强复杂推理任务对多样化知识的整合 [16]。

为构建 KG-index，给定文档集合 $\mathcal{D}$，我们首先从文档中抽取实体 $\mathcal{E}$ 和关系 $\mathcal{R}$，形成三元组 $\mathcal{T}$。随后，构建实体到文档的倒排索引 $\mathbf{M}\in\{0,1\}^{|\mathcal{E}|\times|\mathcal{D}|}$，记录每篇文档中提及的实体。这一过程可由现有的开放信息抽取（OpenIE）工具 [1, 77, 49] 完成。为更好地捕捉知识之间的联系，我们进一步执行实体解析（entity resolution）[13, 74]，在语义相似的实体之间添加额外边 $\mathcal{T}^{+}$，例如（USA，equivalent，United States of America）。因此，最终的 KG-index $\mathcal{G}$ 构建为 $\mathcal{G}=\{(e,r,e')\in\mathcal{T}\cup\mathcal{T}^{+}\}$。在实现中，我们使用一个 LLM [47] 作为 OpenIE 工具（提示词见表 22），并使用一个预训练稠密嵌入模型 [55] 进行实体解析。详细信息见 D.1 节。

### 3.2 图基础模型（GFM）检索器

GFM 检索器旨在根据任意用户查询和构建好的 KG-index 来检索相关文档。尽管 KG-index 提供了结构化的知识表示，但它仍存在不完整性与噪声；若仅依赖其结构，检索性能并非最优 [16]。近期，图神经网络（GNN）[67] 通过捕捉知识之间的复杂关系，在检索或问答任务中展现出令人瞩目的多跳推理能力 [41, 18]。然而，现有 GNN 的泛化能力有限，因为它们通常在特定图上进行训练 [40, 37]，这限制了其在未见语料库和 KG 上的应用。因此，我们仍然需要一种无需额外训练、便可直接应用于未见数据集和 KG 的图基础模型。

为解决这些问题，我们提出首个由图基础模型驱动的检索器（GFM 检索器）。它利用 GNN 的图推理能力，在统一且可迁移的空间中捕捉查询、文档与知识图谱之间的复杂关系。GFM 检索器采用查询依赖型 GNN，在图中识别有助于定位相关文档的实体。在大规模数据集上完成预训练后，GFM 检索器无需进一步训练，便可直接应用于新的语料库和 KG。

#### 3.2.1 查询依赖型 GNN

传统 GNN [14] 遵循消息传递范式，通过反复聚合邻居信息来更新实体表示。由于这一范式是图特定的，且忽略了查询相关性，因此并不适合 GFM 检索器。近期的查询依赖型 GNN [78, 11] 在捕捉查询特定信息以及泛化到未见图方面展现出良好效果；这正是 GFM 检索器所必需的能力，可表示为：

$$
\mathbf{H}_{q}^{L}=\operatorname{GNN}_{q}(q,\mathcal{G},\mathbf{H}^{0}), \tag{4}
$$

其中，$\mathbf{H}^{0}\in\mathbb{R}^{|\mathcal{E}|\times d}$ 表示初始实体特征，$\mathbf{H}_{q}^{L}$ 表示经过 $L$ 层查询依赖型消息传递后，以查询 $q$ 为条件得到的更新实体表示。

理论上已证明，查询依赖型 GNN 具备多跳逻辑推理能力 [21, 73, 52]（详见附录 A），因此我们选择它作为 GFM 检索器的骨干网络。它使 GFM 检索器能够根据用户查询动态调整消息传递过程，通过多跳推理在图上找到最相关的信息。第 4.8 节给出了这一多跳推理过程的路径解释。

**查询初始化。** 给定查询 $q$，我们首先使用句子嵌入模型将其编码为查询嵌入：

$$
\mathbf{q}=\operatorname{SentenceEmb}(q),\quad \mathbf{q}\in\mathbb{R}^{d}, \tag{5}
$$

其中，$d$ 表示查询嵌入的维度。随后，对于查询中提及的所有实体 $e_q\in\mathcal{E}_q\subseteq\mathcal{E}$，我们将其实体特征初始化为 $\mathbf{q}$，其余实体则初始化为零向量：

$$
\mathbf{H}^{0}=\begin{cases}
\mathbf{q}, & e\in\mathcal{E}_q,\\
\mathbf{0}, & \text{其他。}
\end{cases} \tag{6}
$$

**查询依赖型消息传递。** 查询依赖型消息传递会将信息从问题实体传播至 KG 中的其他实体，以捕捉这些实体与查询的相关性。消息传递过程可表示为：

**三元组层面：**

$$
\mathbf{h}_{r}^{0}=\operatorname{SentenceEmb}(r),\quad \mathbf{h}_{r}^{0}\in\mathbb{R}^{d}, \tag{7}
$$

$$
\mathbf{m}_{e}^{l+1}=\operatorname{Msg}\!\left(\mathbf{h}_{e}^{l},g^{l+1}(\mathbf{h}_{r}^{l}),\mathbf{h}_{e'}^{l}\right),\quad (e,r,e')\in\mathcal{G}. \tag{8}
$$

**实体层面：**

$$
\mathbf{h}_{e}^{l+1}=\operatorname{Update}\!\left(\mathbf{h}_{e}^{l},\operatorname{Agg}\!\left(\{\mathbf{m}_{e'}^{l+1}\mid e'\in\mathcal{N}_{r}(e),r\in\mathcal{R}\}\right)\right). \tag{9}
$$

其中，$\mathbf{h}_{e}^{l}$ 和 $\mathbf{h}_{r}^{l}$ 分别表示第 $l$ 层的实体嵌入与关系嵌入。关系嵌入 $\mathbf{h}_{r}^{0}$ 也使用与查询相同的句子嵌入模型进行初始化，以反映其语义（例如“`born_in`”），随后由层特定函数 $g^{l+1}(\cdot)$ 更新；该函数实现为两层 MLP。$\operatorname{Msg}(\cdot)$ 在 KG 的所有三元组上运行以生成消息；按照 NBFNet [78] 的架构，它由无参数的 DistMult [71] 实现。对于每个实体，我们使用求和来聚合其关系为 $r$ 的邻居 $\mathcal{N}_{r}(e)$ 所产生的消息，并以单个线性层更新实体表示。

经过 $L$ 层消息传递后，最后一个 MLP 层与 sigmoid 函数共同将实体嵌入映射为实体相对于查询的相关性分数：

$$
\mathbf{P}_{q}=\sigma\!\left(\operatorname{MLP}(\mathbf{H}_{q}^{L})\right),\quad \mathbf{P}_{q}\in\mathbb{R}^{|\mathcal{E}|\times1}. \tag{10}
$$

**泛化能力。** 由于查询嵌入、实体嵌入和关系嵌入均使用同一个句子嵌入模型初始化，且维度相同，查询依赖型 GNN 可以直接应用于不同的查询与 KG。通过在大规模数据集上训练，它能够同时考虑 KG 的语义与结构，学习查询和实体之间的复杂关系。

#### 3.2.2 训练过程

**训练目标。** GFM 检索器的训练目标是最大化查询相关实体的似然，可通过最小化二元交叉熵（BCE）损失来优化：

$$
\mathcal{L}_{\mathrm{BCE}}=-\frac{1}{|\mathcal{A}_{q}|}\sum_{e\in\mathcal{A}_{q}}\log P_{q}(e)-\frac{1}{|\mathcal{E}^{-}|}\sum_{e\in\mathcal{E}^{-}}\log\!\left(1-P_{q}(e)\right). \tag{11}
$$

其中，$\mathcal{A}_{q}$ 表示与查询 $q$ 相关的目标实体集合，$\mathcal{E}^{-}\subseteq\mathcal{E}\setminus\mathcal{A}_{q}$ 表示从 KG 中采样的负实体集合。然而，由于目标实体稀疏，BCE 损失可能遭遇梯度消失问题 [36]。为解决这一问题，我们进一步引入排序损失 [2]，以最大化正实体与负实体之间的间隔：

$$
\mathcal{L}_{\mathrm{RANK}}=-\frac{1}{|\mathcal{A}_{q}|}\sum_{e\in\mathcal{A}_{q}}\frac{P_{q}(e)}{\sum_{e'\in\mathcal{E}^{-}}P_{q}(e')}. \tag{12}
$$

最终训练目标是 BCE 损失与排序损失的加权组合：

$$
\mathcal{L}=\alpha\mathcal{L}_{\mathrm{BCE}}+(1-\alpha)\mathcal{L}_{\mathrm{RANK}}. \tag{13}
$$

**自监督知识图谱补全预训练。** 为增强 GFM 检索器的图推理能力，我们首先在大规模知识图谱（KG）补全任务上进行预训练。我们从 KG-index 中采样一组三元组，并遮蔽头实体或尾实体，生成形式为 $q=(e,r,?)$ 或 $(?,r,e')$ 的合成查询；被遮蔽的实体作为目标实体，即 $\mathcal{A}_{q}=\{e\}$ 或 $\{e'\}$。随后，按照式（13），训练 GFM 检索器结合查询与 KG 来预测被遮蔽的实体。

**有监督文档检索微调。** 完成自监督预训练后，我们在带标签的文档检索任务上对 GFM 检索器进行有监督微调。在该任务中，查询 $q$ 是自然语言问题，目标实体 $\mathcal{A}_{q}$ 则从带标签的支持文档 $\mathcal{D}_{q}$ 中提取。GFM 检索器采用与式（13）相同的训练目标，从 KG-index 中检索相关实体。

### 3.3 文档排序与答案生成

给定 GFM 检索器预测的实体相关性分数 $\mathbf{P}_{q}\in\mathbb{R}^{|\mathcal{E}|\times1}$，我们首先检索相关性分数最高的 top-$T$ 个实体 $\mathcal{E}_{q}^{T}$：

$$
\mathcal{E}_{q}^{T}=\arg\operatorname{top-}T(\mathbf{P}_{q}),\quad \mathcal{E}_{q}^{T}=\{e_{1},\ldots,e_{T}\}. \tag{14}
$$

随后，文档排序器使用这些检索到的实体来获得最终文档。为了减弱高频实体的影响，我们根据实体在文档倒排索引 $\mathbf{M}\in\{0,1\}^{|\mathcal{E}|\times|\mathcal{D}|}$ 所含文档中的出现频率的倒数，对实体进行加权；再将文档中提及实体的权重求和，计算最终的文档相关性分数：

$$
F_{e}=\begin{cases}
\dfrac{1}{\sum_{d\in\mathcal{D}}\mathbf{M}_{[e,d]}}, & e\in\mathcal{E}_{q}^{T},\\
0, & \text{其他，}
\end{cases} \tag{15}
$$

$$
\mathbf{P}_{d}=\mathbf{M}^{\top}\mathbf{F}_{e},\quad \mathbf{P}_{d}\in\mathbb{R}^{|\mathcal{D}|\times1}. \tag{16}
$$

依据文档相关性分数 $\mathbf{P}_{d}$ 检索 top-$K$ 篇文档，并以检索增强生成的方式将其送入 LLM 的上下文，以生成最终答案：

$$
\mathcal{D}^{K}=\arg\operatorname{top-}K(\mathbf{P}_{d}),\quad \mathcal{D}^{K}=\{D_{1},\ldots,D_{K}\}, \tag{17}
$$

$$
a=\operatorname{LLM}(q,\mathcal{D}^{K}). \tag{18}
$$

## 4 实验

在实验中，我们旨在回答以下研究问题：（1）GFM-RAG 在多跳检索和问答任务中的表现如何？（第 4.2 节和第 4.3 节）；（2）GFM-RAG 在多跳检索中的效率与效果如何？（第 4.4 节）；（3）作为基础模型，GFM-RAG 对未见数据集的泛化能力如何？（第 4.6 节）；（4）作为基础模型，GFM-RAG 的性能如何随训练规模扩展？（第 4.7 节）；（5）如何解释 GFM-RAG 的多跳推理过程？（第 4.8 节）。

### 4.1 实验设置

**数据集。** 我们首先在 3 个广泛使用的多跳问答数据集上评估 GFM-RAG 的有效性，包括 HotpotQA [72]、MuSiQue [63] 和 2WikiMultiHopQA（2Wiki）[20]。我们还在来自 3 个领域的 7 个 RAG 数据集上评估 GFM-RAG，包括生物医学 [25]、客户支持 [54, 44, 39, 4] 和通用知识 [45, 27]，以验证 GFM-RAG 作为基础模型的泛化能力。测试数据集的详细统计信息见附录 B。

**基线。** 我们将 GFM-RAG 与 3 类广泛使用的检索方法进行比较：（1）单步朴素方法：BM25 [53]、Contriever [22]、GTR [46]、ColBERTv2 [55]、RAPTOR [56]、Proposition [6]；（2）图增强方法：GraphRAG (MS) [9]、LightRAG [15]、HippoRAG [16]、SubgraphRAG [32]、G-retriever [18]；（3）多步方法：Adaptive-RAG [23]、FLARE [24]，以及可与任意检索方法集成以执行多步检索和推理的 IRCoT [64] 框架。各基线的详细介绍见附录 C。

**指标。** 对于检索性能，我们采用 recall@2（R@2）和 recall@5（R@5）作为评估指标。对于最终问答性能，我们遵循既有工作 [16]，采用 EM 分数和 F1 分数。

**实现细节。** GFM 检索器由 6 个查询依赖型消息传递层实现，隐藏维度设为 512。我们采用预训练的 all-mpnet-v2 [57]（表 8 给出的完整模型标识为 `sentence-transformers/all-mpnet-base-v2`）作为句子嵌入模型，并在训练期间冻结其参数。GFM 检索器共有 8M 个参数，使用 8 块 NVIDIA A100（80G）训练，批大小为 4，学习率为 5e-4；监督式检索微调阶段的损失权重为 $\alpha=0.3$。训练数据包含 60 个 KG、超过 14M 个三元组，这些三元组由训练集中抽取的 700k 篇文档构建而成。训练数据的统计信息见表 5，实现细节见附录 D。

### 4.2 检索性能

我们首先在 3 个多跳问答数据集上比较 GFM-RAG 与各基线的检索性能。如表 1 所示，GFM-RAG 在所有数据集上均取得最佳性能；在 HotpotQA、MuSiQue 和 2Wiki 的 R@2 指标上，它分别比当前最先进的 IRCoT + HippoRAG 高出 16.8%、8.3% 和 19.8%。这些结果证明了 GFM-RAG 在多跳检索中的有效性。结果还表明，图增强的 HippoRAG 优于朴素单步检索器（如 BM25、RAPTOR），凸显了图结构对于多跳检索的重要性。尽管 GraphRAG (MS) 和 LightRAG 使用了图结构，但二者在多跳问答任务上的表现并不理想，因为其检索器面向摘要生成设计，缺乏多跳推理能力。借助 LLM，多步检索流水线 IRCoT 通过迭代推理和检索提升了所有单步方法的性能。然而，即使只执行单步检索，GFM-RAG 仍大幅优于这些多步方法。这表明，GFM-RAG 能够在单步内有效完成多跳推理（详见第 4.8 节和第 E.8 节），比多步检索流水线更加高效、有效（详见第 4.4 节）。

**表 1：检索性能比较。**

| 类别 | 方法 | HotpotQA R@2 | HotpotQA R@5 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 |
|---|---|---:|---:|---:|---:|---:|---:|
| 单步 | BM25 | 55.4 | 72.2 | 32.3 | 41.2 | 51.8 | 61.9 |
| 单步 | Contriever | 57.2 | 75.5 | 34.8 | 46.6 | 46.6 | 57.5 |
| 单步 | GTR | 59.4 | 73.3 | 37.4 | 49.1 | 60.2 | 67.9 |
| 单步 | ColBERTv2 | 64.7 | 79.3 | 37.9 | 49.2 | 59.2 | 68.2 |
| 单步 | RAPTOR | 58.1 | 71.2 | 35.7 | 45.3 | 46.3 | 53.8 |
| 单步 | Proposition | 58.7 | 71.1 | 37.6 | 49.3 | 56.4 | 63.1 |
| 单步 | GraphRAG (MS) | 58.3 | 76.6 | 35.4 | 49.3 | 61.6 | 77.3 |
| 单步 | LightRAG | 38.8 | 54.7 | 24.8 | 34.7 | 45.1 | 59.1 |
| 单步 | HippoRAG (Contriever) | 59.0 | 76.2 | 41.0 | 52.1 | 71.5 | 89.5 |
| 单步 | HippoRAG (ColBERTv2) | 60.5 | 77.7 | 40.9 | 51.9 | 70.7 | 89.1 |
| 单步 | SubgraphRAG | 61.5 | 73.0 | 42.1 | 49.3 | 70.7 | 85.5 |
| 单步 | G-retriever | 53.3 | 65.5 | 38.8 | 45.1 | 60.8 | 67.8 |
| 多步 | Adaptive-RAG | 61.0 | 76.4 | 35.1 | 44.7 | 44.7 | 61.4 |
| 多步 | FLARE | 73.1 | 81.3 | 44.3 | 55.1 | 67.1 | 73.1 |
| 多步 | IRCoT + BM25 | 65.6 | 79.0 | 34.2 | 44.7 | 61.2 | 75.6 |
| 多步 | IRCoT + Contriever | 65.9 | 81.6 | 39.1 | 52.2 | 51.6 | 63.8 |
| 多步 | IRCoT + ColBERTv2 | 67.9 | 82.0 | 41.7 | 53.7 | 64.1 | 74.4 |
| 多步 | IRCoT + HippoRAG (Contriever) | 65.8 | 82.3 | 43.9 | 56.6 | 75.3 | 93.4 |
| 多步 | IRCoT + HippoRAG (ColBERTv2) | 67.0 | 83.0 | 45.3 | 57.6 | 75.8 | 93.9 |
| 单步 | GFM-RAG | **78.3** | **87.1** | **49.1** | **58.2** | **90.8** | **95.6** |

### 4.3 问答性能

随后，我们评估 GFM-RAG 的问答性能，因为该性能直接受检索质量影响。我们采用 GPT-4o-mini [47] 作为 LLM，并使用检索到的 Top-5 文档生成答案。由表 2 可见，单步 GFM-RAG 已经取得优于所有其他基线的当前最佳性能。同时，我们还将 GFM-RAG 与 IRCoT 集成以执行多步检索和推理，使 3 个数据集上的 EM 分别进一步提升 8.5%、21.2% 和 3.9%。这些结果证明，GFM-RAG 在多跳推理任务中既有效，又能与任意多步框架良好兼容。

**表 2：问答性能比较。**

| 类别 | 检索器 | HotpotQA EM | HotpotQA F1 | MuSiQue EM | MuSiQue F1 | 2Wiki EM | 2Wiki F1 |
|---|---|---:|---:|---:|---:|---:|---:|
| 单步 | 无 | 30.4 | 42.8 | 12.5 | 24.1 | 31.0 | 39.0 |
| 单步 | ColBERTv2 | 43.4 | 57.7 | 15.5 | 26.4 | 33.4 | 43.3 |
| 单步 | GraphRAG (MS) | 35.3 | 54.6 | 13.4 | 29.5 | 28.3 | 46.9 |
| 单步 | LightRAG | 36.8 | 48.3 | 18.1 | 27.5 | 45.1 | 49.5 |
| 单步 | HippoRAG (ColBERTv2) | 41.8 | 55.0 | 19.2 | 29.8 | 46.6 | 59.5 |
| 多步 | Adaptive-RAG | 45.5 | 59.6 | 13.8 | 25.6 | 48.9 | 62.8 |
| 多步 | FLARE | 48.7 | 60.6 | 16.2 | 28.4 | 46.7 | 65.4 |
| 多步 | IRCoT (ColBERTv2) | 45.5 | 58.4 | 19.1 | 30.5 | 35.4 | 45.1 |
| 多步 | IRCoT + HippoRAG (ColBERTv2) | 45.7 | 59.2 | 21.9 | 33.3 | 47.7 | 62.7 |
| 单步 | GFM-RAG | 51.6 | 66.9 | 30.2 | 40.4 | 69.8 | 77.7 |
| 多步 | IRCoT + GFM-RAG | **56.0** | **71.8** | **36.6** | **49.2** | **72.5** | **80.8** |

### 4.4 效率分析

GFM-RAG 能够在单步内高效完成多跳推理。如表 3 所示，朴素单步方法的效率最高，但性能并不理想。诚然，多步框架 IRCoT 能够提升性能，却会因反复调用 LLM 进行迭代检索和推理而产生较高的计算开销。相比之下，GFM-RAG 通过一次单步 GNN 推理完成多跳推理；它比单步方法更有效，也比多步方法更高效。

**表 3：检索效率与性能比较。**

| 方法 | HotpotQA 时间（s） | HotpotQA R@5 | MuSiQue 时间（s） | MuSiQue R@5 | 2Wiki 时间（s） | 2Wiki R@5 |
|---|---:|---:|---:|---:|---:|---:|
| ColBERTv2 | **0.035** | 79.3 | **0.030** | 49.2 | **0.029** | 68.2 |
| HippoRAG | 0.255 | 77.7 | 0.251 | 51.9 | 0.158 | 89.1 |
| LightRAG | 0.861 | 54.7 | 1.109 | 34.7 | 0.911 | 59.1 |
| GraphRAG (MS) | 2.759 | 76.6 | 3.037 | 49.3 | 1.204 | 77.3 |
| IRCoT + ColBERTv2 | 1.146 | 82.0 | 1.152 | 53.7 | 2.095 | 74.4 |
| IRCoT + HippoRAG | 3.162 | 83.0 | 3.104 | 57.6 | 3.441 | 93.9 |
| GFM-RAG | <u>0.107</u> | **87.1** | <u>0.124</u> | **58.2** | <u>0.060</u> | **95.6** |

**图 3：模型泛化能力比较。** 图例为 GFM-RAG、HippoRAG 和 LightRAG。图中标注的 GFM-RAG R@5 分别为：PubMedQA 58.5、DelucionQA 70.8、EManual 60.6、ExpertQA 62.7、TechQA 46.6、MS Marco 71.0、HAGRID 84.7。

**图 4：GFM-RAG 的神经缩放定律。** 图中拟合关系为 $z=0.24x^{0.05}+0.11y^{0.03}$，$R^2=0.95$；$z$ 轴为 MRR（刻度 0.50、0.52、0.54、0.56、0.58），$x$ 轴为数据量（3k、6k、12k、24k、45k），$y$ 轴为参数量（0.08M、0.2M、0.7M、2M、8M）。

### 4.5 消融研究

我们开展消融研究，以考察 GFM-RAG 中不同组成部分的有效性，包括：不同的句子嵌入模型（第 E.1 节）、预训练策略（第 E.2 节）、损失加权策略（第 E.3 节）、排序方法（第 E.4 节）、训练数据集（第 E.5 节）以及 KG-index 的构建（第 E.9 节）。结果表明，GFM-RAG 对句子嵌入模型的选择并不敏感，而预训练策略与损失加权策略对其性能都至关重要。

### 4.6 模型泛化能力

为证明 GFM-RAG 作为基础模型的泛化能力，我们在不进行任何领域特定微调的情况下，于 7 个 RAG 数据集上测试 GFM-RAG 的性能（R@5）。具体而言，我们首先使用各数据集中的文档构建 KG-index。随后，给定查询，利用对应的 KG-index，通过预训练的 GFM 检索器检索 Top-$K$ 文档。如图 3 所示，GFM-RAG 在所有数据集上均取得最佳性能，平均比当前最先进的 HippoRAG 高出 18.9%。这些结果表明，作为基础模型，GFM-RAG 无需任何领域特定微调即可直接应用于各种未见数据集。此外，第 E.6 节的结果还表明，在领域特定数据集上微调后，GFM-RAG 具有很强的可迁移性，性能可以进一步提升。

### 4.7 模型神经缩放定律

我们进一步研究 GFM-RAG 的神经缩放定律，以量化模型性能如何随训练数据规模和模型参数量增长。近期的基础模型研究 [29, 7] 已验证了这一规律。如图 4 所示，GFM-RAG 的性能（MRR：$z$）能够随训练数据量（$x$）和模型规模（$y$）良好扩展，并可由幂律缩放关系拟合：

$$
z\propto 0.24x^{0.05}+0.11y^{0.03}.
$$

这些结果证明了 GFM-RAG 作为基础模型的可扩展性，以及其进一步提升的潜力。神经缩放定律的详细分析见第 E.7 节。

### 4.8 路径解释

多层 GFM 赋予了 GFM-RAG 多跳推理能力。我们在表 4 中给出 GFM-RAG 多跳推理的路径解释。受 NBFNet [78] 启发，可以用预测分数对各层（跳）三元组的偏导数，量化各条路径对最终预测的重要性，定义如下：

$$
s_1,s_2,\ldots,s_L=\arg\operatorname{top-}k\frac{\partial p_e(q)}{\partial s_l}. \tag{19}
$$

通过束搜索选取 Top-$k$ 条最长路径，即可得到 Top-$k$ 条路径解释。表 4 展示了这些路径解释。在第一个示例中，GFM-RAG 成功推断出该歌曲的演唱者有一家以其名字命名的足球俱乐部，并且他曾拥有这家俱乐部。在第二个示例中，GFM-RAG 找到了与该谋杀案及主持审判的法官有关的两条路径。这些解释表明，GFM-RAG 能够在单步检索中完成多跳推理。第 E.8 节还展示了多跳预测的分布。

**表 4：GFM 多跳推理的路径解释，其中 $r^{-1}$ 表示原关系的逆关系。**

| 项目 | 内容 |
|---|---|
| 问题（示例 1） | 《Grow Some Funk of Your Own》的演唱者曾拥有哪家足球俱乐部？ |
| 答案 | Watford Football Club（沃特福德足球俱乐部） |
| 支持文档 | [“Grow Some Funk of Your Own”, “Elton John”] |
| 路径 | 1.095: (grow some funk of your own, is a song by, elton john) → (elton john, equivalent, sir elton hercules john) → (sir elton hercules john, named a stand after$^{-1}$, watford football club)<br><br>0.915: (grow some funk of your own, is a song by, elton john) → (elton john, equivalent, sir elton hercules john) → (sir elton hercules john, owned, watford football club) |
| 问题（示例 2） | 1966 年 7 月 13-14 日夜间，一名男子折磨、强奸并杀害了来自 South Chicago Community Hospital 的 8 名学生护士；在该男子的审判中作出重要贡献的法官出生于何时？ |
| 答案 | June 4, 1931（1931 年 6 月 4 日） |
| 支持文档 | [“Louis B. Garippo”, “Richard Speck”] |
| 路径 | 0.797: (south chicago community hospital, committed crimes at$^{-1}$, richard speck) → (richard speck, equivalent, trial of richard speck) → (trial of richard speck, made contributions during$^{-1}$, louis b garippo)<br><br>0.412: (south chicago community hospital, were from$^{-1}$, eight student nurses) → (eight student nurses, were from, south chicago community hospital) → (south chicago community hospital, committed crimes at$^{-1}$, richard speck) |

## 5 结论

本文提出了首个面向检索增强生成的图基础模型。借助知识图谱索引，GFM-RAG 对知识与文档之间的复杂关系进行显式建模，从而实现更加有效、高效的检索过程。GFM-RAG 由一个在大规模数据集上预训练的查询依赖型 GNN 驱动，能够在图结构上有效执行多跳推理，并在单步内找到相关知识。在 3 个基准数据集和 7 个领域特定数据集上开展的大量实验表明，GFM-RAG 在有效性、效率和泛化能力方面均显著优于当前最先进的方法。其与缩放定律的一致性也表明，该模型具有扩展到更大规模数据集的潜力。未来，我们计划开展更大规模的训练，并进一步探索 GFM-RAG 在知识图谱补全和问答等其他挑战性场景中的能力。

## 致谢

G Haffari 的部分研究得到澳大利亚研究委员会（ARC）Future Fellowship FT190100039，以及美国国防高级研究计划局（DARPA）Assured Neuro Symbolic Learning and Reasoning（ANSR）项目的资助，项目编号为 FA8750-23-2-1016。C Gong 得到中国国家自然科学基金（项目编号：62336003、12371510）资助。D Phung 得到澳大利亚研究委员会（ARC）Discovery Project DP250100262 和 DP230101176 资助。S Pan 的部分研究得到澳大利亚研究委员会（ARC）项目 FT210100097 和 DP240101547，以及 CSIRO - 美国国家科学基金会（NSF）人工智能研究合作计划的资助。

## 参考文献

*为确保引文准确性，参考文献条目保留原文，不翻译题名及出版信息。*

[1] Gabor Angeli, Melvin Jose Johnson Premkumar, and Christopher D Manning. Leveraging linguistic structure for open domain information extraction. In Proceedings of the 53rd Annual Meeting of the Association for Computational Linguistics and the 7th International Joint Conference on Natural Language Processing (Volume 1: Long Papers), pages 344–354, 2015.

[2] Aijun Bai, Rolf Jagerman, Zhen Qin, Le Yan, Pratyush Kar, Bing-Rong Lin, Xuanhui Wang, Michael Bendersky, and Marc Najork. Regression compatible listwise objectives for calibrated ranking with binary relevance. In Proceedings of the 32nd ACM International Conference on Information and Knowledge Management, pages 4502–4508, 2023.

[3] Kurt Bollacker, Colin Evans, Praveen Paritosh, Tim Sturge, and Jamie Taylor. Freebase: a collaboratively created graph database for structuring human knowledge. In Proceedings of the 2008 ACM SIGMOD international conference on Management of data, pages 1247–1250, 2008.

[4] Vittorio Castelli, Rishav Chakravarti, Saswati Dana, Anthony Ferritto, Radu Florian, Martin Franz, Dinesh Garg, Dinesh Khandelwal, Scott McCarley, Michael McCawley, Mohamed Nasr, Lin Pan, Cezar Pendus, John Pitrelli, Saurabh Pujar, Salim Roukos, Andrzej Sakrajda, Avi Sil, Rosario Uceda-Sosa, Todd Ward, and Rong Zhang. The TechQA dataset. In Dan Jurafsky, Joyce Chai, Natalie Schluter, and Joel Tetreault, editors, Proceedings of the 58th Annual Meeting of the Association for Computational Linguistics, pages 1269–1278, Online, July 2020. Association for Computational Linguistics. doi: 10.18653/v1/2020.acl-main.117. URL https://aclanthology.org/2020.acl-main.117/.

[5] Jianlv Chen, Shitao Xiao, Peitian Zhang, Kun Luo, Defu Lian, and Zheng Liu. Bge m3-embedding: Multi-lingual, multi-functionality, multi-granularity text embeddings through self-knowledge distillation, 2023.

[6] Tong Chen, Hongwei Wang, Sihao Chen, Wenhao Yu, Kaixin Ma, Xinran Zhao, Hongming Zhang, and Dong Yu. Dense X retrieval: What retrieval granularity should we use? In Yaser Al-Onaizan, Mohit Bansal, and Yun-Nung Chen, editors, Proceedings of the 2024 Conference on Empirical Methods in Natural Language Processing, pages 15159–15177, Miami, Florida, USA, November 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.emnlp-main.845. URL https://aclanthology.org/2024.emnlp-main.845/.

[7] Mostafa Dehghani, Josip Djolonga, Basil Mustafa, Piotr Padlewski, Jonathan Heek, Justin Gilmer, Andreas Peter Steiner, Mathilde Caron, Robert Geirhos, Ibrahim Alabdulmohsin, et al. Scaling vision transformers to 22 billion parameters. In International Conference on Machine Learning, pages 7480–7512. PMLR, 2023.

[8] Jialin Dong, Bahare Fatemi, Bryan Perozzi, Lin F Yang, and Anton Tsitsulin. Don’t forget to connect! improving rag with graph-based reranking. arXiv preprint arXiv:2405.18414, 2024.

[9] Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt, and Jonathan Larson. From local to global: A graph rag approach to query-focused summarization. arXiv preprint arXiv:2404.16130, 2024.

[10] Robert Friel, Masha Belyi, and Atindriyo Sanyal. Ragbench: Explainable benchmark for retrieval-augmented generation systems. arXiv preprint arXiv:2407.11005, 2024.

[11] Mikhail Galkin, Xinyu Yuan, Hesham Mostafa, Jian Tang, and Zhaocheng Zhu. Towards foundation models for knowledge graph reasoning. In The Twelfth International Conference on Learning Representations, 2024.

[12] Yunfan Gao, Yun Xiong, Xinyu Gao, Kangxiang Jia, Jinliu Pan, Yuxi Bi, Yi Dai, Jiawei Sun, and Haofen Wang. Retrieval-augmented generation for large language models: A survey. arXiv preprint arXiv:2312.10997, 2023.

[13] Dan Gillick, Sayali Kulkarni, Larry Lansing, Alessandro Presta, Jason Baldridge, Eugene Ie, and Diego Garcia-Olano. Learning dense representations for entity retrieval. In Proceedings of the 23rd Conference on Computational Natural Language Learning (CoNLL), pages 528–537, 2019.

[14] Justin Gilmer, Samuel S Schoenholz, Patrick F Riley, Oriol Vinyals, and George E Dahl. Neural message passing for quantum chemistry. In International conference on machine learning, pages 1263–1272. PMLR, 2017.

[15] Zirui Guo, Lianghao Xia, Yanhua Yu, Tu Ao, and Chao Huang. Lightrag: Simple and fast retrieval-augmented generation. 2024.

[16] Bernal Jiménez Gutiérrez, Yiheng Shu, Yu Gu, Michihiro Yasunaga, and Yu Su. Hipporag: Neurobiologically inspired long-term memory for large language models. In The Thirty-eighth Annual Conference on Neural Information Processing Systems, 2024. URL https://openreview.net/forum?id=hkujvAPVsg.

[17] Haoyu Han, Yu Wang, Harry Shomer, Kai Guo, Jiayuan Ding, Yongjia Lei, Mahantesh Halappanavar, Ryan A Rossi, Subhabrata Mukherjee, Xianfeng Tang, et al. Retrieval-augmented generation with graphs (graphrag). arXiv preprint arXiv:2501.00309, 2024.

[18] Xiaoxin He, Yijun Tian, Yifei Sun, Nitesh Chawla, Thomas Laurent, Yann LeCun, Xavier Bresson, and Bryan Hooi. G-retriever: Retrieval-augmented generation for textual graph understanding and question answering. Advances in Neural Information Processing Systems, 37:132876–132907, 2024.

[19] Joel Hestness, Sharan Narang, Newsha Ardalani, Gregory Diamos, Heewoo Jun, Hassan Kianinejad, Md Mostofa Ali Patwary, Yang Yang, and Yanqi Zhou. Deep learning scaling is predictable, empirically. arXiv preprint arXiv:1712.00409, 2017.

[20] Xanh Ho, Anh-Khoa Duong Nguyen, Saku Sugawara, and Akiko Aizawa. Constructing a multi-hop qa dataset for comprehensive evaluation of reasoning steps. In Proceedings of the 28th International Conference on Computational Linguistics, pages 6609–6625, 2020.

[21] Xingyue Huang, Miguel Romero, Ismail Ceylan, and Pablo Barceló. A theory of link prediction via relational weisfeiler-leman on knowledge graphs. Advances in Neural Information Processing Systems, 36:19714–19748, 2023.

[22] Gautier Izacard, Mathilde Caron, Lucas Hosseini, Sebastian Riedel, Piotr Bojanowski, Armand Joulin, and Edouard Grave. Unsupervised dense information retrieval with contrastive learning. Transactions on Machine Learning Research, 2022.

[23] Soyeong Jeong, Jinheon Baek, Sukmin Cho, Sung Ju Hwang, and Jong C Park. Adaptive-rag: Learning to adapt retrieval-augmented large language models through question complexity. In Proceedings of the 2024 Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies (Volume 1: Long Papers), pages 7029–7043, 2024.

[24] Zhengbao Jiang, Frank F Xu, Luyu Gao, Zhiqing Sun, Qian Liu, Jane Dwivedi-Yu, Yiming Yang, Jamie Callan, and Graham Neubig. Active retrieval augmented generation. In Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing, pages 7969–7992, 2023.

[25] Qiao Jin, Bhuwan Dhingra, Zhengping Liu, William Cohen, and Xinghua Lu. PubMedQA: A dataset for biomedical research question answering. In Kentaro Inui, Jing Jiang, Vincent Ng, and Xiaojun Wan, editors, Proceedings of the 2019 Conference on Empirical Methods in Natural Language Processing and the 9th International Joint Conference on Natural Language Processing (EMNLP-IJCNLP), pages 2567–2577, Hong Kong, China, November 2019. Association for Computational Linguistics. doi: 10.18653/v1/D19-1259. URL https://aclanthology.org/D19-1259/.

[26] Ashutosh Joshi, Sheikh Muhammad Sarwar, Samarth Varshney, Sreyashi Nag, Shrivats Agrawal, and Juhi Naik. Reaper: Reasoning based retrieval planning for complex rag systems. In Proceedings of the 33rd ACM International Conference on Information and Knowledge Management, pages 4621–4628, 2024.

[27] Ehsan Kamalloo, Aref Jafari, Xinyu Zhang, Nandan Thakur, and Jimmy Lin. Hagrid: A human-llm collaborative dataset for generative information-seeking with attribution. arXiv preprint arXiv:2307.16883, 2023.

[28] Xiaoxi Kang, Lizhen Qu, Lay-Ki Soon, Zhuang Li, and Adnan Trakic. Bridging law and data: Augmenting reasoning via a semi-structured dataset with irac methodology. arXiv preprint arXiv:2406.13217, 2024.

[29] Jared Kaplan, Sam McCandlish, Tom Henighan, Tom B Brown, Benjamin Chess, Rewon Child, Scott Gray, Alec Radford, Jeffrey Wu, and Dario Amodei. Scaling laws for neural language models. arXiv preprint arXiv:2001.08361, 2020.

[30] Vladimir Karpukhin, Barlas Oguz, Sewon Min, Patrick Lewis, Ledell Wu, Sergey Edunov, Danqi Chen, and Wen-tau Yih. Dense passage retrieval for open-domain question answering. In Proceedings of the 2020 Conference on Empirical Methods in Natural Language Processing (EMNLP), pages 6769–6781, 2020.

[31] Chankyu Lee, Rajarshi Roy, Mengyao Xu, Jonathan Raiman, Mohammad Shoeybi, Bryan Catanzaro, and Wei Ping. Nv-embed: Improved techniques for training llms as generalist embedding models. arXiv preprint arXiv:2405.17428, 2024.

[32] Mufei Li, Siqi Miao, and Pan Li. Simple is effective: The roles of graphs and large language models in knowledge-graph-based retrieval-augmented generation. In The Thirteenth International Conference on Learning Representations, 2025.

[33] Shiyang Li, Yifan Gao, Haoming Jiang, Qingyu Yin, Zheng Li, Xifeng Yan, Chao Zhang, and Bing Yin. Graph reasoning for question answering with triplet retrieval. In Findings of the Association for Computational Linguistics: ACL 2023, pages 3366–3375, 2023.

[34] Zehan Li, Xin Zhang, Yanzhao Zhang, Dingkun Long, Pengjun Xie, and Meishan Zhang. Towards general text embeddings with multi-stage contrastive learning. arXiv preprint arXiv:2308.03281, 2023.

[35] Lei Liang, Mengshu Sun, Zhengke Gui, Zhongshu Zhu, Zhouyu Jiang, Ling Zhong, Yuan Qu, Peilong Zhao, Zhongpu Bo, Jin Yang, et al. Kag: Boosting llms in professional domains via knowledge augmented generation. arXiv preprint arXiv:2409.13731, 2024.

[36] Zhutian Lin, Junwei Pan, Shangyu Zhang, Ximei Wang, Xi Xiao, Shudong Huang, Lei Xiao, and Jie Jiang. Understanding the ranking loss for recommendation with sparse user feedback. In Proceedings of the 30th ACM SIGKDD Conference on Knowledge Discovery and Data Mining, pages 5409–5418, 2024.

[37] Jiawei Liu, Cheng Yang, Zhiyuan Lu, Junze Chen, Yibo Li, Mengmei Zhang, Ting Bai, Yuan Fang, Lichao Sun, Philip S Yu, et al. Graph foundation models: Concepts, opportunities and challenges. IEEE Transactions on Pattern Analysis and Machine Intelligence, 2025.

[38] LINHAO LUO, Yuan-Fang Li, Reza Haf, and Shirui Pan. Reasoning on graphs: Faithful and interpretable large language model reasoning. In The Twelfth International Conference on Learning Representations, 2024.

[39] Chaitanya Malaviya, Subin Lee, Sihao Chen, Elizabeth Sieber, Mark Yatskar, and Dan Roth. Expertqa: Expert-curated questions and attributed answers. arXiv preprint arXiv:2309.07852, 2023.

[40] Haitao Mao, Zhikai Chen, Wenzhuo Tang, Jianan Zhao, Yao Ma, Tong Zhao, Neil Shah, Mikhail Galkin, and Jiliang Tang. Position: Graph foundation models are already here. In Forty-first International Conference on Machine Learning, 2024.

[41] Costas Mavromatis and George Karypis. Gnn-rag: Graph neural retrieval for large language model reasoning. arXiv preprint arXiv:2405.20139, 2024.

[42] Meta. Build the future of ai with meta llama 3, 2024. URL https://llama.meta.com/llama3/.

[43] Gabriel de Souza P Moreira, Radek Osmulski, Mengyao Xu, Ronay Ak, Benedikt Schifferer, and Even Oldridge. Nv-retriever: Improving text embedding models with effective hard-negative mining. arXiv preprint arXiv:2407.15831, 2024.

[44] Abhilash Nandy, Soumya Sharma, Shubham Maddhashiya, Kapil Sachdeva, Pawan Goyal, and NIloy Ganguly. Question answering over electronic devices: A new benchmark dataset and a multi-task learning based QA framework. In Marie-Francine Moens, Xuanjing Huang, Lucia Specia, and Scott Wen-tau Yih, editors, Findings of the Association for Computational Linguistics: EMNLP 2021, pages 4600–4609, Punta Cana, Dominican Republic, November 2021. Association for Computational Linguistics. doi: 10.18653/v1/2021.findings-emnlp.392. URL https://aclanthology.org/2021.findings-emnlp.392/.

[45] Tri Nguyen, Mir Rosenberg, Xia Song, Jianfeng Gao, Saurabh Tiwary, Rangan Majumder, and Li Deng. Ms marco: A human generated machine reading comprehension dataset. November 2016.

[46] Jianmo Ni, Chen Qu, Jing Lu, Zhuyun Dai, Gustavo Hernandez Abrego, Ji Ma, Vincent Zhao, Yi Luan, Keith Hall, Ming-Wei Chang, et al. Large dual encoders are generalizable retrievers. In Proceedings of the 2022 Conference on Empirical Methods in Natural Language Processing, pages 9844–9855, 2022.

[47] OpenAI. Hello gpt-4o, 2024. URL https://openai.com/index/hello-gpt-4o/.

[48] OpenAI. Learning to reason with llms, 2024. URL https://openai.com/index/learning-to-reason-with-llms/.

[49] Liu Pai, Wenyang Gao, Wenjie Dong, Lin Ai, Ziwei Gong, Songfang Huang, Li Zongsheng, Ehsan Hoque, Julia Hirschberg, and Yue Zhang. A survey on open information extraction from rule-based model to large language model. Findings of the Association for Computational Linguistics: EMNLP 2024, pages 9586–9608, 2024.

[50] Pranoy Panda, Ankush Agarwal, Chaitanya Devaguptapu, Manohar Kaul, and Prathosh Ap. Holmes: Hyper-relational knowledge graphs for multi-hop question answering using llms. In Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 13263–13282, 2024.

[51] Boci Peng, Yun Zhu, Yongchao Liu, Xiaohe Bo, Haizhou Shi, Chuntao Hong, Yan Zhang, and Siliang Tang. Graph retrieval-augmented generation: A survey. arXiv preprint arXiv:2408.08921, 2024.

[52] Haiquan Qiu, Yongqi Zhang, Yong Li, et al. Understanding expressivity of gnn in rule learning. In The Twelfth International Conference on Learning Representations, 2024.

[53] Stephen E Robertson and Steve Walker. Some simple effective approximations to the 2-poisson model for probabilistic weighted retrieval. In SIGIR’94: Proceedings of the Seventeenth Annual International ACM-SIGIR Conference on Research and Development in Information Retrieval, organised by Dublin City University, pages 232–241. Springer, 1994.

[54] Mobashir Sadat, Zhengyu Zhou, Lukas Lange, Jun Araki, Arsalan Gundroo, Bingqing Wang, Rakesh Menon, Md Parvez, and Zhe Feng. DelucionQA: Detecting hallucinations in domain-specific question answering. In Houda Bouamor, Juan Pino, and Kalika Bali, editors, Findings of the Association for Computational Linguistics: EMNLP 2023, pages 822–835, Singapore, December 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.findings-emnlp.59. URL https://aclanthology.org/2023.findings-emnlp.59/.

[55] Keshav Santhanam, Omar Khattab, Jon Saad-Falcon, Christopher Potts, and Matei Zaharia. Colbertv2: Effective and efficient retrieval via lightweight late interaction. In Proceedings of the 2022 Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies, pages 3715–3734, 2022.

[56] Parth Sarthi, Salman Abdullah, Aditi Tuli, Shubh Khanna, Anna Goldie, and Christopher D Manning. Raptor: Recursive abstractive processing for tree-organized retrieval. In The Twelfth International Conference on Learning Representations, 2024.

[57] SBERT. Sentence-transformers all-mpnet-base-v2, 2021. URL https://huggingface.co/sentence-transformers/all-mpnet-base-v2.

[58] Weihang Su, Yichen Tang, Qingyao Ai, Zhijing Wu, and Yiqun Liu. DRAGIN: Dynamic retrieval augmented generation based on the real-time information needs of large language models. In Lun-Wei Ku, Andre Martins, and Vivek Srikumar, editors, Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 12991–13013, Bangkok, Thailand, August 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.702. URL https://aclanthology.org/2024.acl-long.702/.

[59] Jiashuo Sun, Chengjin Xu, Lumingyuan Tang, Saizhuo Wang, Chen Lin, Yeyun Gong, Lionel Ni, Heung-Yeung Shum, and Jian Guo. Think-on-graph: Deep and responsible reasoning of large language model on knowledge graph. In The Twelfth International Conference on Learning Representations, 2024.

[60] Jiabin Tang, Yuhao Yang, Wei Wei, Lei Shi, Lixin Su, Suqi Cheng, Dawei Yin, and Chao Huang. Graphgpt: Graph instruction tuning for large language models. In Proceedings of the 47th International ACM SIGIR Conference on Research and Development in Information Retrieval, pages 491–500, 2024.

[61] Jiabin Tang, Yuhao Yang, Wei Wei, Lei Shi, Long Xia, Dawei Yin, and Chao Huang. Higpt: Heterogeneous graph language model. In Proceedings of the 30th ACM SIGKDD Conference on Knowledge Discovery and Data Mining, pages 2842–2853, 2024.

[62] Timothy J Teyler and Pascal DiScenna. The hippocampal memory indexing theory. Behavioral neuroscience, 100(2):147, 1986.

[63] Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, and Ashish Sabharwal. Musique: Multihop questions via single-hop question composition. Transactions of the Association for Computational Linguistics, 10:539–554, 2022.

[64] Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, and Ashish Sabharwal. Interleaving retrieval with chain-of-thought reasoning for knowledge-intensive multi-step questions. In Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pages 10014–10037, 2023.

[65] Denny Vrandečić and Markus Krötzsch. Wikidata: a free collaborative knowledgebase. Communications of the ACM, 57(10):78–85, 2014.

[66] Zehong Wang, Zheyuan Zhang, Nitesh V Chawla, Chuxu Zhang, and Yanfang Ye. GFT: Graph foundation model with transferable tree vocabulary. In The Thirty-eighth Annual Conference on Neural Information Processing Systems, 2024. URL https://openreview.net/forum?id=0MXzbAv8xy.

[67] Zonghan Wu, Shirui Pan, Fengwen Chen, Guodong Long, Chengqi Zhang, and S Yu Philip. A comprehensive survey on graph neural networks. IEEE transactions on neural networks and learning systems, 32(1):4–24, 2020.

[68] Lianghao Xia, Ben Kao, and Chao Huang. Opengraph: Towards open graph foundation models. arXiv preprint arXiv:2403.01121, 2024.

[69] Shitao Xiao, Zheng Liu, Peitian Zhang, and Niklas Muennighoff. C-pack: Packaged resources to advance general chinese embedding, 2023.

[70] An Yang, Baosong Yang, Binyuan Hui, Bo Zheng, Bowen Yu, Chang Zhou, Chengpeng Li, Chengyuan Li, Dayiheng Liu, Fei Huang, Guanting Dong, Haoran Wei, Huan Lin, Jialong Tang, Jialin Wang, Jian Yang, Jianhong Tu, Jianwei Zhang, Jianxin Ma, Jin Xu, Jingren Zhou, Jinze Bai, Jinzheng He, Junyang Lin, Kai Dang, Keming Lu, Keqin Chen, Kexin Yang, Mei Li, Mingfeng Xue, Na Ni, Pei Zhang, Peng Wang, Ru Peng, Rui Men, Ruize Gao, Runji Lin, Shijie Wang, Shuai Bai, Sinan Tan, Tianhang Zhu, Tianhao Li, Tianyu Liu, Wenbin Ge, Xiaodong Deng, Xiaohuan Zhou, Xingzhang Ren, Xinyu Zhang, Xipin Wei, Xuancheng Ren, Yang Fan, Yang Yao, Yichang Zhang, Yu Wan, Yunfei Chu, Yuqiong Liu, Zeyu Cui, Zhenru Zhang, and Zhihao Fan. Qwen2 technical report. arXiv preprint arXiv:2407.10671, 2024.

[71] Bishan Yang, Scott Wen-tau Yih, Xiaodong He, Jianfeng Gao, and Li Deng. Embedding entities and relations for learning and inference in knowledge bases. In Proceedings of the International Conference on Learning Representations (ICLR) 2015, 2015.

[72] Zhilin Yang, Peng Qi, Saizheng Zhang, Yoshua Bengio, William Cohen, Ruslan Salakhutdinov, and Christopher D Manning. Hotpotqa: A dataset for diverse, explainable multi-hop question answering. In Proceedings of the 2018 Conference on Empirical Methods in Natural Language Processing, pages 2369–2380, 2018.

[73] Michihiro Yasunaga, Hongyu Ren, Antoine Bosselut, Percy Liang, and Jure Leskovec. Qa-gnn: Reasoning with language models and knowledge graphs for question answering. In Proceedings of the 2021 Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies, pages 535–546, 2021.

[74] Alexandros Zeakis, George Papadakis, Dimitrios Skoutas, and Manolis Koubarakis. Pre-trained embeddings for entity resolution: an experimental analysis. Proceedings of the VLDB Endowment, 16(9):2225–2238, 2023.

[75] Xinyu Zhang, Nandan Thakur, Odunayo Ogundepo, Ehsan Kamalloo, David Alfonso-Hermelo, Xiaoguang Li, Qun Liu, Mehdi Rezagholizadeh, and Jimmy Lin. Making a miracl: Multilingual information retrieval across a continuum of languages. arXiv preprint arXiv:2210.09984, 2022.

[76] Lingfeng Zhong, Jia Wu, Qian Li, Hao Peng, and Xindong Wu. A comprehensive survey on automatic knowledge graph construction. ACM Computing Surveys, 56(4):1–62, 2023.

[77] Shaowen Zhou, Bowen Yu, Aixin Sun, Cheng Long, Jingyang Li, and Jian Sun. A survey on neural open information extraction: Current status and future directions. In Lud De Raedt, editor, Proceedings of the Thirty-First International Joint Conference on Artificial Intelligence, IJCAI-22, pages 5694–5701. International Joint Conferences on Artificial Intelligence Organization, 7 2022. doi: 10.24963/ijcai.2022/793. URL https://doi.org/10.24963/ijcai.2022/793. Survey Track.

[78] Zhaocheng Zhu, Zuobai Zhang, Louis-Pascal Xhonneux, and Jian Tang. Neural bellman-ford networks: A general graph neural network framework for link prediction. Advances in Neural Information Processing Systems, 34:29476–29490, 2021.

# 附录

## 附录目录

- A 用于多跳推理与检索的查询依赖型 GNN（原文第 17 页）
- B 数据集（原文第 17 页）
  - B.1 多跳问答数据集（原文第 17 页）
  - B.2 领域特定 RAG 数据集（原文第 18 页）
- C 基线（原文第 19 页）
- D 实现与训练细节（原文第 20 页）
  - D.1 训练数据构建（原文第 20 页）
  - D.2 模型设置（原文第 20 页）
  - D.3 训练设置（原文第 20 页）
- E 补充实验（原文第 21 页）
  - E.1 不同句子嵌入的有效性（原文第 21 页）
  - E.2 不同训练策略的有效性（原文第 22 页）
  - E.3 损失权重的有效性（原文第 22 页）
  - E.4 排序方法的有效性（原文第 23 页）
  - E.5 训练数据集消融研究（原文第 23 页）
  - E.6 模型可迁移性（原文第 23 页）
  - E.7 模型神经缩放的细节（原文第 24 页）
  - E.8 多跳预测分布可视化（原文第 25 页）
  - E.9 LLM 对 KG-index 构建的成本与影响（原文第 26 页）
- F 提示词（原文第 27 页）
- G 局限性（原文第 27 页）

## A 用于多跳推理与检索的查询依赖型 GNN

本节详细解释为何查询依赖型 GNN 可用于多跳推理与检索。近期研究 [21, 52] 已从理论上证明，查询依赖型 GNN 能够有效捕捉知识图谱上的多跳逻辑关联，从而回答如下查询：

$$
\exists y:\ \operatorname{politician\_of}(\text{Barack Obama},y)\leftarrow \operatorname{work\_in}(\text{Barack Obama},z_1)\land \operatorname{city\_of}(z_1,y). \tag{20}
$$

右侧表示可执行的逻辑关联，用于回答左侧的查询，即 `politician_of(Barack Obama, y)`。

该查询在语义上等价于自然语言问题：“Barack Obama 是哪个国家的政治人物？”我们将输入问题视为“软查询”（即自然语言形式的查询），并应用查询依赖型 GNN（GFM）来弥合自然语言与逻辑查询之间的差距。GFM 尝试理解问题的语义，并学习在知识图谱上执行复杂逻辑推理（如多跳推理），以完成检索 [73]。模型学到的推理逻辑关联见第 4.8 节。

**表 5：训练所用问题-文档对及知识图谱的统计信息。**

| 数据集 | #问题-文档对 | #文档 | #KG | #实体 | #关系 | #三元组 |
|---|---:|---:|---:|---:|---:|---:|
| HotpotQA | 20,000 | 204,822 | 20 | 1,930,362 | 967,218 | 6,393,342 |
| MuSiQue | 20,000 | 410,380 | 20 | 1,544,966 | 900,338 | 4,848,715 |
| 2Wiki | 20,000 | 122,108 | 20 | 916,907 | 372,554 | 2,883,006 |
| 合计 | **60,000** | **737,310** | **60** | **4,392,235** | **2,240,110** | **14,125,063** |

## B 数据集

### B.1 多跳问答数据集

实验使用了 3 个多跳问答数据集：HotpotQA [72]、MuSiQue [63] 和 2WikiMultiHopQA（2Wiki）[20]。下面对这些数据集作简要介绍。

**表 6：测试所用数据集及所构建 KG-index 的统计信息。**

| 数据集 | 领域 | #测试样本 | #文档 | #实体 | #关系 | #三元组 |
|---|---|---:|---:|---:|---:|---:|
| HotpotQA | 多跳 | 1,000 | 9,221 | 87,768 | 45,112 | 279,112 |
| MuSiQue | 多跳 | 1,000 | 11,656 | 100,853 | 55,944 | 319,618 |
| 2Wiki | 多跳 | 1,000 | 6,119 | 48,779 | 20,748 | 160,950 |
| PubMedQA | 生物医学 | 2,450 | 5,932 | 42,389 | 20,952 | 149,782 |
| DelucionQA | 客户支持 | 184 | 235 | 2,669 | 2,298 | 6,183 |
| TechQA | 客户支持 | 314 | 769 | 10,221 | 4,606 | 57,613 |
| ExpertQA | 客户支持 | 203 | 808 | 11,079 | 6,810 | 16,541 |
| EManual | 客户支持 | 132 | 102 | 695 | 586 | 1,329 |
| MS Marco | 通用知识 | 423 | 3,481 | 24,740 | 17,042 | 63,995 |
| HAGRID | 通用知识 | 1,318 | 1,975 | 23,484 | 18,653 | 48,969 |

- HotpotQA [72] 是一个多跳问答数据集，需要跨多篇文档进行推理才能回答问题。该数据集包含 97k 个问题-答案对；每个问题最多关联 2 篇支持文档和若干干扰文档。问题被设计为需要综合支持文档中的多条信息才能作答。

- MuSiQue [63] 是一个具有挑战性的多跳问答数据集，包含 25k 个需要 2-4 跳推理的问题。要回答这些跨越多篇文档的问题，需要进行连贯的多步推理。

- 2WikiMultiHopQA（2Wiki）[20] 是一个多跳问答数据集，需要跨多篇 Wikipedia 文章进行推理才能回答问题。该数据集包含 192k 个问题，这些问题被设计为可利用 2 篇或 4 篇文章中的信息作答。

实验中，我们遵循官方数据划分获取训练样本，并沿用既有方法 [64, 16]，从每个验证集中使用相同的 1,000 个样本，以避免数据泄漏。我们将候选段落合并为用于构建 KG-index 的文档语料库。训练数据和测试数据的统计信息分别见表 5 和表 6。

### B.2 领域特定 RAG 数据集

为测试 GFM-RAG 的泛化能力，我们在 7 个领域特定 RAG 数据集 [10] 上进行评估，包括：（1）生物医学：PubMedQA [25]；（2）客户支持：DelucionQA [54]、TechQA [4]、ExpertQA [39]、EManual [44]；（3）通用知识：MS Marco [45]、HAGRID [27]。下面对这些数据集作简要介绍。

- PubMedQA [25] 收集了 PubMed 研究摘要及相应问题，每个问题配有 4 个摘要片段。

- DelucionQA [54] 是一个领域特定 RAG 数据集，以 Jeep 2023 款 Gladiator 车型手册作为知识来源；每个问题关联 4 篇上下文文档，其中仅有 1 个相关段落。

- TechQA [4] 收集了用户在 IBMDeveloper 和 DeveloperWorks 论坛上发布的真实问题，并为每个问题配有 10 篇相关技术支持文档。

- ExpertQA [39] 收集了科学、艺术和法律等不同领域专家编写的问题。该数据集还包含由专家筛选、与各问题相关的段落。

- EManual [44] 是一个问答数据集，由消费电子设备手册以及人工标注者据此编写的真实问题构成；每个问题最多关联 3 篇上下文文档。

- MS Marco [45] 是一个开放域问答数据集，来源于 Bing 搜索引擎的用户查询日志。每个问题关联 10 个通过 Bing 网页搜索检索得到的上下文段落。

- HAGRID [27] 是一个多语言信息检索数据集，其问题和段落来自 MIRACL [75]。

实验中，我们使用由 RAGBench [10] 构建的测试集，并将所有候选段落合并为用于构建 KG-index 的文档语料库。测试数据集的统计信息详见表 6。

## C 基线

实验中，我们将 GFM-RAG 与 3 类广泛使用的检索方法进行比较：（1）单步朴素方法：BM25 [53]、Contriever [22]、GTR [46]、ColBERTv2 [55]、RAPTOR [56]、Proposition [6]；（2）图增强方法：GraphRAG (MS) [9]、LightRAG [15]、HippoRAG [16]；（3）多步方法：Adaptive-RAG [23]、FLARE [24] 和 IRCoT [64]。下面详细介绍这些基线。

> **译注：** 正文第 4.1 节的基线列表还包括 SubgraphRAG 与 G-retriever，但原文附录 C 未为这两种方法提供单独介绍。

**单步朴素方法**因其出色的效率与泛化能力，被广泛用于实际应用。

- BM25 [53] 是一种基于概率模型的经典信息检索方法，它根据查询词在各文档中的出现频率对一组文档进行排序。

- Contriever [22] 在大规模语料库上通过对比学习训练稠密检索器，以检索与给定查询相关的文档。

- GTR [46] 构建了一个经规模扩展、基于 T5 的稠密检索器，能够泛化到不同的数据集和领域。

- ColBERTv2 [55] 是一种当前最先进的稠密检索器，将强力的残差压缩机制与去噪监督策略相结合，从而提升检索质量。

- RAPTOR [56] 是一种由 LLM 增强的检索器，它对文本块进行递归嵌入、聚类和摘要，构建包含不同摘要层级的树结构，从而实现准确检索。

- Proposition [6] 利用 LLM 生成能够捕捉文档关键信息的自然语言命题，以提升稠密检索器的性能。

**图增强方法**在图结构之上构建检索器，以执行有效的检索和推理。

- GraphRAG (MS) [9] 是 Microsoft 最初提出的一种图增强检索方法。它从文档语料库构建图结构，利用层次化社区检测将文档聚类为多个社区，并为每个社区生成摘要。检索器同时检索这些摘要与原始文档，供 LLM 生成答案。

- LightRAG [15] 是一种创新的图增强 RAG 方法，将图结构融入文本索引和检索，从而高效检索实体及其关系。它采用双层检索系统，同时收集低层与高层知识，供 LLM 生成答案。

- HippoRAG [16] 是一种当前最先进、无需训练的图增强检索器。它使用 Personalized PageRank 算法评估实体与查询的相关性，并在基于文档的知识图谱上执行多跳检索；该方法可以直接应用于各种数据集。

**多步方法**通过迭代地检索文档并基于文档进行推理来执行多跳推理，并可与任意检索方法集成。

- Adaptive-RAG [23] 提出一种自适应多步检索方法，可根据查询的复杂度动态选择最合适的检索策略。

- FLARE [24] 提出一种多步检索方法，能够主动决定何时以及如何检索文档；它还会预测未来内容，以指导后续步骤中的检索。

- IRCoT [64] 是一套强大的多步检索流水线，将检索与 LLM 的思维链（CoT）推理相结合。它利用 CoT 指导检索，再利用检索到的文档改进 CoT。IRCoT 可与任意检索器兼容，以执行多步检索和推理。

**表 7：GFM-RAG 的详细实现与训练设置。**

| 模块 | 设置 | GFM-RAG |
|---|---|---|
| KG-index 构建 | OpenIE | GPT-4o-mini |
| KG-index 构建 | 实体解析 | ColBERTv2 |
| KG-index 构建 | $\tau$ | 0.8 |
| GFM 模型 | 层数 | 6 |
| GFM 模型 | 隐藏维度 | 512 |
| GFM 模型 | 消息函数 | DistMult |
| GFM 模型 | 聚合 | 求和 |
| GFM 模型 | $g^l(\cdot)$ | 2 层 MLP |
| GFM 模型 | 句子嵌入模型 | all-mpnet-v2 |
| GFM 模型 | 文档排序器实体数 $T$ | 20 |
| KGC 预训练 | $\alpha$ | 1 |
| KGC 预训练 | 优化器 | AdamW |
| KGC 预训练 | 学习率 | 5e-4 |
| KGC 预训练 | 批大小 | 4 |
| KGC 预训练 | 训练步数 | 30,000 |
| KGC 预训练 | 负样本数 | 128 |
| 监督式检索微调 | $\alpha$ | 0.3 |
| 监督式检索微调 | 优化器 | AdamW |
| 监督式检索微调 | 学习率 | 5e-4 |
| 监督式检索微调 | 批大小 | 4 |
| 监督式检索微调 | 训练轮数 | 5 |
| 监督式检索微调 | 负样本 | $\mathcal{E}\setminus\mathcal{A}_q$ |

## D 实现与训练细节

### D.1 训练数据构建

我们从 HotpotQA、MuSiQue 和 2Wiki 的训练集中抽取 60,000 个样本，用于构建 KG-index 并开展大规模训练。具体而言，我们将候选段落合并为文档语料库。在构建 KG-index 时，我们使用 GPT-4o-mini [47]，按照 HippoRAG [16] 中描述的 OpenIE 提示词，从文档语料库中抽取实体、关系和三元组。随后，我们使用 ColBERTv2 [55] 进行实体解析，其做法是按下式计算实体之间的相似度：

$$
s(e_i,e_j)=\operatorname{Emb.}(e_i)^\top\operatorname{Emb.}(e_j), \tag{21}
$$

当 $s(e_i,e_j)>\tau$ 且 $e_i\ne e_j$ 时，生成一个新的三元组 $(e_i,\text{equivalent},e_j)$。实验中，我们将阈值 $\tau$ 设为 0.8。为控制所构建 KG-index 的规模，我们将样本划分为若干组，每组约包含 1k 个问题和 10k 篇文档。最终，我们得到 60 个不同的 KG-index，以及与之关联的“问题-文档”对，用于模型训练。

### D.2 模型设置

在 GFM-RAG 中，GFM 被实现为一个 6 层、查询依赖型 GNN，其隐藏维度为 512，采用 DistMult 消息函数与求和聚合。关系更新函数 $g^l(\cdot)$ 由一个 2 层 MLP 实现。我们使用 all-mpnet-v2 作为句子嵌入模型，其维度为 768。GFM 的可训练参数总量为 8M。在检索阶段，我们为文档排序器选取排名前 $T=20$ 的实体。

### D.3 训练设置

在知识图谱补全预训练中，我们从知识图谱中随机采样三元组 $(e,r,t)$，并遮蔽头实体或尾实体，以自监督方式构造合成查询 $q=(e,r,?)$ 和答案 $a=\{e\}$。例如，给定三元组 `(Barack Obama, born_in, Honolulu)`，可以构造查询 `(Barack Obama, born_in, ?)`；该查询被编码为句子嵌入，并输入 GFM，以在图上预测目标实体 Honolulu。

> **译注：** 原文此处的符号存在笔误。若遮蔽尾实体，应该是 $q=(e,r,?)$、$a=\{t\}$；若遮蔽头实体，则应是 $q=(?,r,t)$、$a=\{e\}$。后面的 Honolulu 示例与前一种情形一致。

**表 8：GFM-RAG 所用不同句子嵌入模型的比较。**

| 句子嵌入模型 | HotpotQA R@2 | HotpotQA R@5 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 |
|---|---:|---:|---:|---:|---:|---:|
| sentence-transformers/all-mpnet-base-v2 | 70.2 | 82.1 | 46.0 | 55.1 | 81.1 | 85.6 |
| BAAI/bge-large-en | 68.1 | 81.1 | 45.9 | 55.9 | 80.7 | 86.3 |
| Alibaba-NLP/gte-Qwen2-1.5B-instruct | 69.9 | 81.5 | 46.0 | 55.0 | 79.8 | 86.2 |
| Alibaba-NLP/gte-Qwen2-7B-instruct | 68.5 | 81.5 | 45.5 | 55.1 | 80.8 | 85.6 |
| nvidia/NV-Embed-v2 | 69.2 | 81.4 | 46.3 | 54.9 | 80.3 | 85.5 |

**表 9：GFM-RAG 与预训练及微调后句子嵌入模型的比较。**

| 方法 | HotpotQA R@2 | HotpotQA R@5 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 |
|---|---:|---:|---:|---:|---:|---:|
| GFM-RAG | 78.3 | 87.1 | 49.1 | 58.2 | 90.8 | 95.6 |
| all-mpnet-v2（预训练） | 59.4 | 73.3 | 33.2 | 46.3 | 48.5 | 59.4 |
| all-mpnet-v2（微调） | 67.0 | 82.3 | 41.7 | 55.0 | 65.1 | 76.7 |

在监督式文档检索微调中，我们从多跳问答数据集中获取自然语言问题及其支持文档。对于每个问题，我们将其支持文档中的实体确定为目标。例如，给定问题“Barack Obama 出生在哪里？”，我们可以从其支持文档（如图 2 中的文档 2）中抽取 `[Honolulu, USA]` 这样的两个实体。训练 GFM 的目标是最大化这两个目标实体的似然。

在自监督知识图谱补全预训练阶段，GFM 在 60 个已构建 KG-index 的混合数据上训练 30,000 步。随后，我们在带标注的“问题-文档”对上进行 5 个 epoch 的监督式文档检索微调。两项损失之间的权重 $\alpha$ 设为 0.3。我们使用 AdamW 优化器，学习率为 5e-4，两个训练阶段的批大小均设为 4。每个批次只包含一个 KG-index 及与其关联的训练样本；训练期间，我们从不同 KG-index 中随机采样。模型使用 8 块 NVIDIA A100（80G）训练，其中预训练耗时 14 小时，监督微调耗时 5 小时。详细设置汇总于表 7。

> **译注：** 原文此处笼统写 $\alpha=0.3$，而表 7 将两个阶段区分为：KGC 预训练阶段 $\alpha=1$，监督式检索微调阶段 $\alpha=0.3$。

## E 补充实验

### E.1 不同句子嵌入的有效性

本节首先研究不同句子嵌入在 GFM 中的有效性。我们比较 all-mpnet-v2 [57]、bge-large-en [69]、gte-Qwen2-1.5B-instruct、gte-Qwen2-7B-instruct [34] 以及 NV-Embed-v2 [31]。我们从 Hugging Face[^3] 下载官方预训练模型。各模型的详细信息见表 8。从结果可以看出，不同句子嵌入之间的性能差异相对较小，其中 all-mpnet-v2 在 3 项指标上取得最佳性能。这表明 GFM-RAG 对句子嵌入模型的选择并不敏感。实验中，考虑到效率，我们使用 all-mpnet-v2 作为默认句子嵌入模型。不过，它的上下文长度相对较小（512），因而限制了输入文本的长度。我们将探索上下文长度更大的句子嵌入模型（例如上下文为 32k 的 NV-Embed-v2）留作未来工作。

随后，我们扩展消融研究，将 GFM-RAG 与不使用 GNN、仅使用预训练 all-mpnet-v2 嵌入的变体，以及仅使用在多跳问答数据上微调所得嵌入的变体分别进行比较。结果见表 9。可以看到，GNN 在检索中发挥着关键作用。句子嵌入模型 all-mpnet-v2 在大规模文本数据上进行过预训练，因而可能已经见过这些问答数据。然而，它并未针对多跳问答任务专门训练，因此在捕捉问题与支持文档之间的关系时表现欠佳。经过多跳问答数据监督微调的 all-mpnet-v2 优于预训练版本，但仍不及 GFM-RAG。这表明，GNN 能够有效捕捉知识之间的关系并执行多跳推理，而仅使用句子嵌入模型无法做到这一点。

[^3]: https://huggingface.co/

**表 10：GFM-RAG 中 KGC 预训练与监督式检索微调的有效性。**

| 方法 | HotpotQA R@2 | HotpotQA R@5 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 |
|---|---:|---:|---:|---:|---:|---:|
| GFM-RAG | 78.3 | 87.1 | 49.1 | 58.2 | 89.1 | 92.8 |
| GFM-RAG（无检索微调） | 21.0 | 32.8 | 18.3 | 25.9 | 44.6 | 53.4 |
| GFM-RAG（无 KGC 预训练） | 77.8 | 86.5 | 48.3 | 58.3 | 88.3 | 92.5 |

> **译注：** 表 10 中完整 GFM-RAG 的 2Wiki 结果为 89.1/92.8，而正文表 1 及附录中的其他相关表格报告为 90.8/95.6；原文未说明两者是否采用不同设置，译文按表中数值保留。

**表 11：不同训练策略的知识图谱补全结果。**

| 方法 | MRR | Hits@1 | Hits@3 | Hits@10 |
|---|---:|---:|---:|---:|
| GFM-RAG | 0.193 | 0.138 | 0.221 | 0.293 |
| GFM-RAG（无检索微调） | 0.304 | 0.234 | 0.323 | 0.451 |
| GFM-RAG（无 KGC 预训练） | 0.029 | 0.007 | 0.022 | 0.067 |

### E.2 不同训练策略的有效性

本节首先研究 GFM-RAG 所采用两项训练任务的有效性。我们分别比较仅进行知识图谱补全预训练（GFM-RAG w/o Fine-tune）和仅进行监督式文档检索微调（GFM-RAG w/o Pre-train）时的性能。结果见表 10。结果表明，移除监督式文档检索微调会显著降低 GFM-RAG 的性能。这凸显了监督微调的重要性：它使模型能够理解用户查询，并更好地捕捉问题与知识之间的相关性，从而改善检索效果。

尽管预训练对最终性能的影响相对较小，但遵循 ULTRA [11] 等既有研究，其首要目的在于学习通用的图推理能力。这将增强 GFM 的泛化性与鲁棒性，并可能有利于其在知识图谱补全等其他任务上的表现。为进一步验证这一点，我们开展消融研究，比较采用不同训练策略的 GFM-RAG 在知识图谱补全任务上的表现。我们报告 HotpotQA 数据集测试集所对应 KG-index 上的知识图谱补全（KGC）性能，结果见表 11。

从知识图谱补全结果可以看到，仅经过预训练的 GFM-RAG（GFM-RAG w/o Fine-tune）取得最佳性能，说明预训练能够有效学习通用图推理能力。仅经过监督微调的 GFM-RAG（GFM-RAG w/o Pre-train）性能显著低于经过预训练的 GFM-RAG。这表明，监督微调仅学习特定的下游任务，因而会限制 GFM-RAG 作为基础模型的泛化能力。同时经过预训练与监督微调的 GFM，在知识图谱补全任务上取得第二佳性能，在多跳问答任务上取得最佳性能。这说明，两种训练策略对于 GFM-RAG 学习通用图推理能力并赋能特定下游任务都不可或缺。

### E.3 损失权重的有效性

本节考察训练 GFM-RAG 时分配给 BCE 损失与排序损失的权重是否有效。我们改变两项损失之间的权重 $\alpha$，比较模型性能：

$$
\mathcal{L}=\alpha\mathcal{L}_{\mathrm{BCE}}+(1-\alpha)\mathcal{L}_{\mathrm{RANK}}.
$$

结果见表 12。实验发现，仅使用 BCE 损失或仅使用排序损失都会导致次优性能（$\alpha=0$ 或 $1$）。当 $\alpha$ 设为 0.3 时，模型性能最佳；这与既有研究 [36] 一致，即当训练数据中的正样本较少时，最好为 BCE 损失分配较小权重。

**表 12：两项损失的权重 $\alpha$ 对有效性（MRR）的影响。**

| $\alpha$ | HotpotQA | MuSiQue | 2Wiki |
|---:|---:|---:|---:|
| 0 | 0.5189 | 0.3252 | 0.4425 |
| 1 | 0.5096 | 0.3214 | 0.4282 |
| 0.7 | 0.5202 | 0.3249 | 0.4348 |
| 0.3 | 0.5243 | 0.3260 | 0.4490 |

### E.4 排序方法的有效性

本节研究 GFM-RAG 中基于倒排索引的不同排序方法是否有效。我们比较以下 4 种排序方法：

1. **IDF + Top-T Pred**：本文提出的方法（式 (14) 至 (16)），使用逆文档频率（IDF）加权分数，将 GFM 预测的前 $T$ 个实体映射到文档。
2. **IDF + All Pred**：使用 GFM 预测的所有实体，并按 IDF 加权（不使用式 (14)）。
3. **Top-T Pred**：仅使用预测的前 $T$ 个实体，不应用 IDF 加权（不使用式 (15)）。
4. **All Pred**：使用所有实体预测，并直接映射为文档分数（不使用式 (14) 和 (15)）。

结果见表 13。结果表明，本文提出的 IDF + Top-T Pred 表现最佳。这说明倒排索引是 GFM-RAG 的关键组成部分：它连接了知识图谱上的结构化推理与 LLM 所需的非结构化文档，因此必须谨慎设计。

我们也认识到还存在其他潜在方案。作为一个很有前景的未来方向，我们计划探索能够对结构化知识与非结构化知识进行联合推理、且不依赖显式倒排索引的端到端模型。

**表 13：不同排序方法的比较。**

| 排序方法 | HotpotQA R@2 | HotpotQA R@5 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 |
|---|---:|---:|---:|---:|---:|---:|
| IDF + Top-T Pred（GFM-RAG） | 78.3 | 87.1 | 49.1 | 58.2 | 90.8 | 95.6 |
| IDF + All Pred（不使用式 (14)） | 68.1 | 71.4 | 35.8 | 41.2 | 86.0 | 87.5 |
| Top-T Pred（不使用式 (15)） | 71.6 | 78.6 | 46.3 | 52.5 | 74.7 | 78.1 |
| All Pred（不使用式 (14) 和 (15)） | 77.6 | 82.9 | 41.1 | 46.9 | 88.6 | 90.4 |

### E.5 训练数据集消融研究

我们进一步开展消融研究，分别在每个数据集上训练 GFM-RAG，并报告模型在全部 3 个基准上的性能。结果见表 14。结果表明，GFM-RAG 不仅在用于训练的数据集上表现良好，也能很好地泛化到其他数据集。更重要的是，在多领域数据集上训练的模型在所有数据集上都展现出有竞争力的性能；这验证了模型能够通过在多样化知识图谱上训练，学习可跨领域泛化的推理能力，从而有效泛化至不同领域并从多样化训练中获益。

**表 14：分别在各数据集上训练 GFM-RAG 的消融研究。最佳结果以粗体标出，第二佳结果以下划线标出。**

| 训练数据集 | HotpotQA R@2 | HotpotQA R@5 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 |
|---|---:|---:|---:|---:|---:|---:|
| HotpotQA | **79.3** | **87.8** | 46.9 | 57.2 | 86.6 | 92.4 |
| MuSiQue | 68.8 | 81.8 | <u>47.6</u> | <u>57.5</u> | 84.4 | 89.6 |
| 2Wiki | 72.2 | 77.9 | 46.6 | 55.5 | <u>89.3</u> | <u>93.2</u> |
| 全部 | <u>78.3</u> | <u>87.1</u> | **49.1** | **58.2** | **90.8** | **95.6** |

### E.6 模型可迁移性

本节通过在每个领域数据集的训练划分上进行领域特定微调，评估 GFM-RAG 的可迁移性。如表 15 所示，GFM-RAG 具有良好的零样本泛化性能，而领域特定微调可进一步提升其表现。这凸显了 GFM-RAG 在适配领域特定数据集时的可迁移性。

**表 15：模型性能（R@5）及可迁移性比较。**

| 模型 | DelucionQA | EManual | ExpertQA | TechQA | MS Marco | HAGRID |
|---|---:|---:|---:|---:|---:|---:|
| HippoRAG（零样本） | 59.0 | 50.0 | 55.1 | 39.5 | 51.1 | 75.5 |
| LightRAG（零样本） | 46.1 | 46.2 | 59.4 | 36.8 | 48.3 | 75.9 |
| GFM-RAG（零样本） | 70.8 | 60.6 | 62.7 | 46.6 | 71.0 | 84.7 |
| GFM-RAG（领域特定微调） | **82.7** | **75.9** | **60.8** | **49.5** | **77.5** | **86.6** |

> **译注：** 表 15 显示领域特定微调在 6 个已报告数据集中有 5 个得到提升；ExpertQA 是例外，其 R@5 从 62.7 降至 60.8。

### E.7 模型神经缩放的细节

本节给出神经缩放实验的更多细节。我们评估模型性能随参数规模及训练数据规模变化的情况。在 GFM-RAG 中，模型参数规模主要受 GFM 隐藏维度的影响。因此，我们将维度从 32 改变至 512，使模型参数规模处于 0.08M 至 8M 之间。详细设置见表 16。我们在规模从 3k 至 45k 个样本的不同训练数据上测试不同大小的模型。图 5 分别给出了性能随模型参数规模及训练数据规模变化的拟合趋势线。从趋势线可以看出，GFM-RAG 的性能会随模型参数规模和训练数据规模增大而提升。同时，模型参数规模越大，就需要越大的训练数据规模才能达到最佳性能。这表明，同时扩大模型规模与训练数据规模，可以进一步提升 GFM-RAG 的性能。

为进一步研究架构设计，我们在隐藏维度固定为 512 的情况下，将 GNN 层数从 1 改变至 8，并在所有数据集上评估模型性能。结果见表 17。我们观察到，随着 GNN 层数加深，性能总体有所提升；我们认为原因既包括模型规模的增大，也包括模型捕捉更复杂多跳关联的能力增强。这一趋势符合基础模型中观察到的神经缩放定律，即参数量越大，通常泛化能力越强。

有趣的是，我们发现某些情况下，模型性能在约 4 层时达到峰值。如附录 A 和第 4.8 节所述，GFM-RAG 旨在通过多跳消息传递捕捉知识图谱中的逻辑关联。然而，由于我们的数据集所需的最大推理跳数为 4，超过 4 层后，额外层带来的收益有限；这很可能是因为缺少更高跳数的训练信号。这一发现支持了我们的假设：GFM-RAG 能够有效学习与查询相关的多跳推理路径；若没有需要更复杂推理的数据集，更深的架构可能无法改善性能。总而言之，这些结果证明了所提出的基于 GNN 的架构具有有效性与可解释性，并确认模型容量与逻辑表达能力共同促成了 GFM-RAG 的强劲表现。我们认识到其他架构设计的潜力，并计划在未来加以探索，同时也希望启发研究社区开展同类研究。

**表 16：用于缩放定律分析的隐藏维度、对应模型规模及训练批大小。**

| 隐藏维度 | 参数规模 | 批大小（A100，80G） |
|---:|---:|---:|
| 32 | 78,977 | 40 |
| 64 | 215,297 | 20 |
| 128 | 659,969 | 20 |
| 256 | 2,237,441 | 8 |
| 512 | 8,144,897 | 4 |

**图 5：GFM-RAG 的模型缩放定律与数据缩放定律示意图。**

- **模型缩放（Model Scaling）**：图例中的训练数据规模为 3k、6k、12k、24k、30k、36k 和 45k；横轴为“参数量（对数刻度）”，刻度为 0.08M、0.2M、0.7M、2M 和 8M；纵轴为 MRR，刻度为 0.49、0.50、0.51、0.52、0.53、0.54、0.55、0.56、0.57、0.58 和 0.59。
- **数据缩放（Data Scaling）**：图例中的模型规模为 0.08M、0.2M、0.7M、2M 和 8M；横轴为“数据量（对数刻度）”，刻度为 3k、5k、10k、20k、30k 和 50k；纵轴为 MRR，刻度为 0.49、0.50、0.51、0.52、0.53、0.54、0.55、0.56、0.57、0.58 和 0.59。

**表 17：用于缩放定律分析的不同层数、对应模型规模及性能。**

| 隐藏维度 = 512，层数 | 平均 R@2 | 平均 R@5 | HotpotQA R@2 | HotpotQA R@5 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 层（3M） | 53.9 | 66.7 | 59.3 | 74.2 | 40.7 | 50.2 | 61.8 | 75.7 |
| 2 层（4M） | 69.9 | 78.6 | 73.6 | 85.4 | 47.6 | 57.0 | 88.6 | 93.3 |
| 4 层（6M） | 72.2 | **80.1** | 78.4 | 87.8 | 49.3 | **60.1** | 88.8 | 92.5 |
| 6 层（8M） | 71.9 | 79.6 | 78.0 | 87.0 | 48.4 | 58.7 | 89.3 | 93.1 |
| 8 层（10M） | **73.0** | 79.9 | **79.7** | **87.8** | **49.7** | 59.1 | **89.5** | **92.8** |

> **译注：** 粗体沿用原表。原表将 8 层模型的 2Wiki R@5（92.8）加粗，但该列数值高于它的还有 2 层模型（93.3）和 6 层模型（93.1）。

### E.8 多跳预测分布可视化

本节将 GFM-RAG 多跳推理过程中跳数的分布可视化。我们计算 HotpotQA、MuSiQue 和 2Wiki 测试集中每个问题的真实推理路径所需跳数，然后比较真实推理路径、GFM-RAG 预测推理路径与 HippoRAG 预测推理路径的跳数分布。结果见图 6。可以看到，GFM-RAG 的分布与真实分布高度一致，说明 GFM-RAG 能够在单个步骤内有效执行多跳推理。与此同时，HippoRAG 的分布与真实分布差异较大，在 2Wiki 数据集上尤其如此。这表明 HippoRAG 可能无法有效捕捉复杂关系，进而难以在图上执行多跳推理。

**表 18：KG-index 构建成本。**

| LLM | 每 10k 篇文档的价格 | 总价格 |
|---|---:|---:|
| GPT-4o-mini | \$2.93 | \$216 |

**图 6：GFM-RAG 与 HippoRAG 相对于真实结果的预测跳数统计。**

- 图例：真实结果（GT）、GFM-RAG、HippoRAG。
- 三个子图分别对应 HotpotQA、MuSiQue 和 2Wiki；横轴为跳数，标示刻度为 2、4、6、8；纵轴为百分比（%），刻度为 0、10、20、30、40、50。
- HotpotQA：GFM-RAG 相对 GT 的 MAE 为 2.03，HippoRAG 相对 GT 的 MAE 为 4.71。
- MuSiQue：GFM-RAG 相对 GT 的 MAE 为 4.90，HippoRAG 相对 GT 的 MAE 为 8.13。
- 2Wiki：GFM-RAG 相对 GT 的 MAE 为 4.56，HippoRAG 相对 GT 的 MAE 为 16.44。

**表 19：索引构建的 token 成本比较。**

| 方法 | 每 10k 篇文档的 token 数 |
|---|---:|
| LightRAG | 55M |
| GraphRAG | 76M |
| GFM-RAG | **48M** |

### E.9 LLM 对 KG-index 构建的成本与影响

本节首先分析 KG-index 的构建成本。实验中，我们使用 GPT-4o-mini[^4] 进行 OpenIE 抽取，并为 737,310 篇文档构建 KG-index。成本见表 18 和表 19。具体而言，我们发现，每 10k 篇文档构建 KG-index 约需 48M 个 token；使用 GPT-4o-mini 时，成本约为 \$2.6。LightRAG 和 GraphRAG 分别消耗 57M 和 76M 个 token。与其他方法相比，GFM-RAG 更具成本效益，因为它不需要生成社区层级摘要。此外，我们还在表 20 中比较了 GFM-RAG 的图索引构建时间。结果表明，GFM-RAG 得益于检索期间快速的索引过程，因为它不需要构建传统向量数据库来存储文档与实体。

> **译注：** 原文此处与表 18、表 19 存在数值不一致：表 18 报告每 10k 篇文档 \$2.93，此处写约 \$2.6；表 19 报告 LightRAG 为 55M token，此处写 57M。译文均按原文保留。

诚然，使用 LLM 构建 KG-index 会产生计算成本。不过，KG 构建已经得到广泛研究，并且存在许多不依赖 LLM 的替代方法 [76]。我们的实现提供了一个便捷接口，可以与任何 KG 构建工具集成。未来，我们将探索使用其他 KG 构建方法。

我们进一步分析用于构建 KG-index 的 LLM 对 GFM-RAG 性能的影响。我们使用不同 LLM 构建 KG-index，包括 GPT-4o-mini 和 GPT-3.5-turbo[^5]，随后在所构建的 KG-index 上重新评估 GFM-RAG 与 HippoRAG 的性能。结果见表 21。从结果来看，两种方法在 GPT-4o-mini 所抽取 KG 上的性能，都高于其在 GPT-3.5-turbo 所抽取 KG 上的性能。这支持了如下观点：在构建高质量 KG-index 方面，GPT-4o-mini 总体优于 GPT-3.5-turbo；而高质量 KG-index 对于图增强检索至关重要。不过，在两种 KG-index 上，GFM-RAG 的性能都显著高于 HippoRAG。这说明 GFM-RAG 对 KG-index 质量更具鲁棒性，体现了 GFM 在图推理与检索方面的有效性。

[^4]: https://platform.openai.com/docs/models/o4-mini
[^5]: https://platform.openai.com/docs/models/gpt-3-5-turbo

> **译注：** 原文将 GPT-4o-mini 的脚注链接写为 `.../models/o4-mini`，链接路径与正文模型名不一致；此处保留原链接。

**表 20：图索引构建时间比较。**

| 方法 | 索引构建时间（秒） |
|---|---:|
| LightRAG | 1430.32 |
| GraphRAG（MS） | 1796.43 |
| GFM-RAG | **93.55** |

**表 21：使用不同 LLM 构建 KG-index 时的模型性能比较。**

| 方法 | HotpotQA R@2 | HotpotQA R@5 | MuSiQue R@2 | MuSiQue R@5 | 2Wiki R@2 | 2Wiki R@5 |
|---|---:|---:|---:|---:|---:|---:|
| GFM-RAG（gpt-4o-mini） | 78.3 | 87.1 | 49.1 | 58.2 | 90.8 | 95.6 |
| HippoRAG（gpt-4o-mini） | 62.2 | 79.3 | 41.7 | 53.6 | 72.1 | 89.5 |
| GFM-RAG（gpt-3.5-turbo） | 75.6 | 84.7 | 46.1 | 55.8 | 85.2 | 90.4 |
| HippoRAG（gpt-3.5-turbo） | 60.5 | 77.7 | 40.9 | 51.9 | 70.7 | 89.1 |

## F 提示词

在实验中，我们沿用 HippoRAG [16] 所使用的提示词，从文档语料库中抽取三元组，具体内容见表 22。

## G 局限性

GFM-RAG 的局限性如下：

1. KG-index 的构建可能成本高昂且耗时，尤其是在使用 LLM 进行 OpenIE 抽取时。未来，我们将探索使用高效的 KG 构建方法，并优化构建流程。
2. 与拥有数十亿参数的大语言模型等其他基础模型相比，GFM-RAG 的模型规模相对较小（8M）。尽管直接比较基于 GNN 的模型与基于 Transformer 的 LLM 并不公平，但未来我们将探索扩大 GFM-RAG 的规模，以提升其性能与泛化能力。
3. GFM-RAG 目前仅在多跳问答任务和 KG 补全任务上进行了评估。未来，我们将探索 GFM-RAG 在知识图谱问答、知识图谱推理等其他任务上的能力，以验证其作为基础模型的有效性。

## 影响声明

本文所述工作的目标是推动机器学习领域的发展。我们的工作可能产生许多社会影响，但我们认为其中没有任何一项必须在此特别强调。

## OpenIE 提示词

### 指令

你的任务是根据给定段落和命名实体列表构建一个 RDF（资源描述框架）图。请返回一个由三元组组成的 JSON 列表，其中每个三元组表示 RDF 图中的一种关系。请注意以下要求：

- 每个三元组应至少包含该段落命名实体列表中的 1 个实体，最好包含 2 个。
- 为保持表述清晰，请明确地将代词解析为其所指的具体名称。

将段落转换为一个 JSON 字典，其中包含命名实体列表和三元组列表。

### 单样本演示

段落：

```
Radio City
Radio City 是印度第一家私营 FM 广播电台，于 2001年7月3日开播。它播放印地语、英语及地区语言歌曲。2008年5月，Radio City 通过推出音乐门户网站 PlanetRadiocity.com 进军新媒体；该网站提供音乐相关新闻、视频、歌曲及其他音乐相关功能。
```

```json
{
  "named_entities": [
    "Radio City",
    "印度",
    "2001年7月3日",
    "印地语",
    "英语",
    "2008年5月",
    "PlanetRadiocity.com"
  ]
}
```

```json
{
  "triples": [
    ["Radio City", "位于", "印度"],
    ["Radio City", "是", "私营 FM 广播电台"],
    ["Radio City", "开始运营于", "2001年7月3日"],
    ["Radio City", "播放的歌曲语言为", "印地语"],
    ["Radio City", "播放的歌曲语言为", "英语"],
    ["Radio City", "进军", "新媒体"],
    ["Radio City", "推出", "PlanetRadiocity.com"],
    ["PlanetRadiocity.com", "推出于", "2008年5月"],
    ["PlanetRadiocity.com", "是", "音乐门户网站"],
    ["PlanetRadiocity.com", "提供", "新闻"],
    ["PlanetRadiocity.com", "提供", "视频"],
    ["PlanetRadiocity.com", "提供", "歌曲"]
  ]
}
```

### 输入

将段落转换为一个 JSON 字典，其中包含命名实体列表和三元组列表。段落：

```
INPUT PASSAGE
```

**表 22：用于 OpenIE 抽取的提示词。**

# NeurIPS 论文检查表

## 1. 主要声明

**问题：** 摘要与引言中的主要声明是否准确反映了论文的贡献与范围？

**回答：** [Yes（是）]

**理由：** 摘要与引言中的主要声明准确反映了论文的贡献与范围。

**指南：**

- 回答 `NA` 表示摘要与引言未包含论文提出的声明。
- 摘要和/或引言应清楚陈述论文提出的声明，包括论文贡献以及重要假设与局限性。对本问题回答 `No` 或 `NA`，不会给审稿人留下良好印象。
- 所提出的声明应与理论及实验结果相符，并反映这些结果预期可以在多大程度上泛化至其他场景。
- 可以将愿景性目标作为研究动机，但前提是明确说明论文尚未实现这些目标。

## 2. 局限性

**问题：** 论文是否讨论了作者所开展工作的局限性？

**回答：** [Yes（是）]

**理由：** 我们已在附录 G 中讨论本工作的局限性。

**指南：**

- 回答 `NA` 表示论文没有局限性；回答 `No` 则表示论文存在局限性，但并未在文中加以讨论。
- 鼓励作者在论文中单独设置“局限性”一节。
- 论文应指出所有强假设，并说明结果在这些假设遭到违反时的鲁棒性如何（例如独立性假设、无噪声设置、模型设定正确、仅在局部成立的渐近近似）。作者应反思这些假设在实践中可能如何被违反，以及由此会产生哪些影响。
- 作者应反思所提出声明的适用范围，例如，该方法是否只在少数数据集上测试，或是否只运行了少数几次。一般而言，实证结果往往依赖隐含假设，而这些假设应当明确阐述。
- 作者应反思影响所提方法性能的因素。例如，图像分辨率较低或拍摄光线不足时，人脸识别算法可能表现不佳；又如，语音转文本系统可能无法可靠地为在线课程生成隐藏字幕，因为它无法处理技术术语。
- 作者应讨论所提算法的计算效率，以及算法如何随数据集规模扩展。
- 如适用，作者应讨论其方法在处理隐私与公平性问题方面可能存在的局限性。
- 作者或许担心，完全坦诚地说明局限性会被审稿人用作拒稿理由；但更糟糕的结果可能是，审稿人发现了论文未承认的局限性。作者应运用最佳判断，并认识到，个人为透明度采取的行动对于形成维护研究社区诚信的规范至关重要。审稿人将被明确要求，不得因作者如实说明局限性而予以惩罚。

## 3. 理论假设与证明

**问题：** 对于每一项理论结果，论文是否给出了完整的假设集合以及完整且正确的证明？

**回答：** [NA（不适用）]

**理由：** 本文不包含理论结果。

**指南：**

- 回答 `NA` 表示论文不包含理论结果。
- 论文中的所有定理、公式和证明都应编号并进行交叉引用。
- 任何定理的陈述都应清楚列出或引用其全部假设。
- 证明可以出现在论文正文或补充材料中；但若证明置于补充材料，鼓励作者在正文中提供简短的证明思路，以帮助读者建立直觉。
- 反过来，论文核心内容中给出的任何非形式化证明，都应由附录或补充材料中的形式化证明加以补充。
- 证明所依赖的定理与引理应得到恰当引用。

## 4. 实验结果的可复现性

**问题：** 在论文主要实验结果会影响其主要声明和/或结论的范围内，论文是否完整披露了复现这些结果所需的全部信息（无论是否提供代码与数据）？

**回答：** [Yes（是）]

**理由：** 我们在附录 D 中详细说明了数据构建流程、模型设置和训练流程，以确保结果可复现。

**指南：**

- 回答 `NA` 表示论文不包含实验。
- 如果论文包含实验，对本问题回答 `No` 不会给审稿人留下良好印象：无论是否提供代码与数据，保证论文可复现都很重要。
- 如果论文贡献是数据集和/或模型，作者应说明为确保结果可复现或可验证而采取的步骤。
- 可复现性可以根据贡献类型通过多种方式实现。例如，如果贡献是一种新架构，完整描述该架构可能已经足够；如果贡献是某个具体模型及其实证评估，则可能需要让他人能够使用同一数据集复现该模型，或提供模型访问方式。一般而言，发布代码与数据通常是实现可复现性的一种好方法；但也可以通过提供复现实验的详细说明、提供托管模型的访问权限（例如大语言模型）、发布模型检查点，或采用其他适合所开展研究的方式来实现。
- 尽管 NeurIPS 不要求发布代码，但会议要求所有投稿都提供某种合理的复现途径；具体途径可能取决于贡献性质。例如：
  - (a) 如果主要贡献是一种新算法，论文应清楚说明如何复现该算法。
  - (b) 如果主要贡献是一种新的模型架构，论文应清晰、完整地描述该架构。
  - (c) 如果贡献是一个新模型（例如大语言模型），则应提供一种访问该模型以复现结果的方式，或者提供一种复现该模型的方式（例如使用开源数据集，或提供数据集构建说明）。
  - (d) 我们认识到，某些情况下实现可复现性可能很困难；此时，欢迎作者说明其提供的具体复现方式。对于闭源模型，模型访问可能在某种程度上受到限制（例如仅向注册用户开放），但其他研究人员仍应拥有某种复现或验证结果的途径。

## 5. 数据与代码的开放获取

**问题：** 论文是否提供数据与代码的开放访问权限，并按照补充材料中的说明，提供了足以忠实复现主要实验结果的指导？

**回答：** [Yes（是）]

**理由：** 我们已将代码上传至论文中的匿名链接。

**指南：**

- 回答 `NA` 表示论文不包含需要代码的实验。
- 更多详情请参阅 [NeurIPS 代码与数据提交指南](https://nips.cc/public/guides/CodeSubmissionPolicy)。
- 尽管我们鼓励发布代码与数据，但也理解这可能无法做到，因此回答 `No` 是可以接受的。论文不能仅仅因为未包含代码而被拒稿，除非代码是论文贡献的核心（例如，论文提出了新的开源基准）。
- 说明中应包含复现结果所需运行的准确命令与环境。更多详情请参阅 [NeurIPS 代码与数据提交指南](https://nips.cc/public/guides/CodeSubmissionPolicy)。
- 作者应提供数据访问与准备说明，包括如何访问原始数据、预处理数据、中间数据和生成数据等。
- 作者应提供脚本，以复现新提出方法及各基线的全部实验结果。如果只有部分实验可以复现，则应说明脚本省略了哪些实验，以及省略的原因。
- 投稿时，为保持匿名性，作者应发布匿名化版本（如适用）。
- 建议在补充材料（附于论文之后）中提供尽可能多的信息，但也允许包含指向数据与代码的 URL。

## 6. 实验设置/细节

**问题：** 论文是否给出了理解实验结果所需的全部训练与测试细节（例如数据划分、超参数、超参数的选择方式、优化器类型等）？

**回答：** [Yes（是）]

**理由：** 我们已在附录 D 中详细说明实验设置。

**指南：**

- 回答 `NA` 表示论文不包含实验。
- 论文正文应以足以理解并合理解读结果的详细程度说明实验设置。
- 完整细节可以随代码提供，也可以置于附录或补充材料中。

## 7. 实验的统计显著性

**问题：** 论文是否报告了定义恰当且正确的误差条，或其他有关实验统计显著性的适当信息？

**回答：** [NA（不适用）]

**理由：** 实验采用固定随机种子进行，未报告误差条。

**指南：**

- 回答 `NA` 表示论文不包含实验。
- 如果至少对于支撑论文主要声明的实验，结果附有误差条、置信区间或统计显著性检验，则作者应回答 `Yes`。
- 应清楚说明误差条所捕捉的变异因素（例如训练/测试划分、初始化、某一参数的随机抽取，或在给定实验条件下的完整运行过程）。
- 应说明误差条的计算方法（闭式公式、调用库函数、bootstrap 等）。
- 应给出所采用的假设（例如误差服从正态分布）。
- 应明确误差条表示标准差还是均值的标准误差。
- 可以报告 1-sigma 误差条，但应明确说明。若未验证误差正态性假设，与其声称具有 96% 置信区间，作者最好报告 2-sigma 误差条。
- 对于非对称分布，作者应谨慎，不要在表格或图中展示可能导致结果超出有效范围的对称误差条（例如出现负错误率）。
- 如果在表格或图中报告了误差条，作者应在正文中解释其计算方式，并引用相应的图或表。

## 8. 实验计算资源

**问题：** 对于每项实验，论文是否提供了足够信息，说明复现实验所需的计算机资源（计算工作节点类型、内存、执行时间）？

**回答：** [Yes（是）]

**理由：** 实验所用资源已在附录 D.3 中详细说明。

**指南：**

- 回答 `NA` 表示论文不包含实验。
- 论文应说明计算工作节点的类型，如 CPU 或 GPU、内部集群或云服务提供商，并包括相关内存与存储信息。
- 论文应给出每次独立实验运行所需的计算量，并估计总计算量。
- 论文应披露整个研究项目是否消耗了比文中所报告实验更多的计算资源（例如，未纳入论文的初步实验或失败实验）。

## 9. 伦理准则

**问题：** 论文所开展的研究是否在各个方面都符合 [NeurIPS 伦理准则](https://neurips.cc/public/EthicsGuidelines)？

**回答：** [Yes（是）]

**理由：** 论文所开展的研究符合 NeurIPS 伦理准则。

**指南：**

- 回答 `NA` 表示作者尚未审阅 NeurIPS 伦理准则。
- 如果作者回答 `No`，则应解释必须偏离伦理准则的特殊情形。
- 作者应确保保持匿名性（例如，因其所在司法管辖区的法律或法规而需进行特殊考虑时）。

## 10. 更广泛的影响

**问题：** 论文是否同时讨论了所开展工作可能产生的正面社会影响与负面社会影响？

**回答：** [NA（不适用）]

**理由：** 所提出的方法聚焦于问题的技术层面，不涉及社会影响。

**指南：**

- 回答 `NA` 表示所开展的工作不存在社会影响。
- 如果作者回答 `NA` 或 `No`，则应解释为什么其工作没有社会影响，或为什么论文没有讨论社会影响。
- 负面社会影响的示例包括：潜在的恶意或非预期用途（例如虚假信息、生成虚假档案、监控）；公平性问题（例如部署某项技术后，可能作出对特定群体产生不公平影响的决策）；隐私问题；以及安全问题。
- 会议预计许多论文将属于基础研究，与特定应用无关，更不涉及部署。不过，如果工作存在直接通向任何负面应用的路径，作者应当指出。例如，可以合理指出，生成模型质量的提升可能被用于制作传播虚假信息的深度伪造内容。另一方面，没有必要指出，用于优化神经网络的通用算法可能让人们更快训练生成深度伪造内容的模型。
- 作者应考虑以下几类潜在危害：技术按预期使用且运行正确时可能产生的危害；技术按预期使用但给出错误结果时可能产生的危害；以及因有意或无意滥用技术而产生的危害。
- 如果存在负面社会影响，作者还可以讨论可能的缓解策略（例如限制模型发布、在提供攻击方法的同时提供防御方法、设置滥用监控机制、设置用于监控系统如何随时间从反馈中学习的机制，以及提高机器学习的效率与可及性）。

## 11. 保障措施

**问题：** 对于具有较高滥用风险的数据或模型（例如预训练语言模型、图像生成器或抓取所得数据集），论文是否说明了为负责任地发布这些资产而采取的保障措施？

**回答：** [NA（不适用）]

**理由：** 本文使用的现有数据集与预训练模型均已发布，并且已经配备保障措施。

**指南：**

- 回答 `NA` 表示论文不存在此类风险。
- 对于滥用或双重用途风险较高的模型，应在发布时配备必要保障措施，以便对模型使用加以控制；例如，要求用户遵守使用指南或访问限制，或实现安全过滤器。
- 从互联网抓取的数据集可能带来安全风险。作者应说明如何避免发布不安全图像。
- 我们认识到，提供有效保障措施具有挑战性，而且许多论文并不需要此类措施；但我们仍鼓励作者将其纳入考虑，并尽最大诚意付出努力。

## 12. 现有资产的许可证

**问题：** 论文所使用资产（例如代码、数据、模型）的创建者或原始所有者是否得到恰当致谢？论文是否明确提及并妥善遵守其许可证与使用条款？

**回答：** [Yes（是）]

**理由：** 我们已恰当引用研究中使用的全部代码、数据和模型，并遵守原作者设定的许可协议与使用条款。

**指南：**

- 回答 `NA` 表示论文没有使用现有资产。
- 作者应引用最初发布该代码包或数据集的论文。
- 作者应说明所用资产的版本，并尽可能提供 URL。
- 每项资产都应包含许可证名称（例如 CC-BY 4.0）。
- 对于从特定来源（例如网站）抓取的数据，应提供该来源的版权信息与服务条款。
- 如果发布资产，资产包中应提供许可证、版权信息及使用条款。对于常用数据集，[paperswithcode.com/datasets](https://paperswithcode.com/datasets) 已整理了部分数据集的许可证，其许可指南可帮助确定数据集的许可证。
- 对于经过重新打包的现有数据集，应同时提供原始许可证和派生资产的许可证（如果许可证发生变化）。
- 如果无法在线获取这些信息，鼓励作者联系资产创建者。

## 13. 新资产

**问题：** 论文引入的新资产是否得到充分记录？相关文档是否与资产一同提供？

**回答：** [NA（不适用）]

**理由：** 本文没有引入新资产。

**指南：**

- 回答 `NA` 表示论文没有发布新资产。
- 研究人员应在投稿时通过结构化模板说明数据集/代码/模型的细节，包括训练、许可证、局限性等。
- 论文应讨论是否以及如何获得资产所涉人员的同意。
- 投稿时，请记得对资产进行匿名化（如适用）。可以创建匿名 URL，也可以附上匿名化 zip 文件。

## 14. 众包与人类受试者研究

**问题：** 对于众包实验和人类受试者研究，论文是否包含向参与者提供的完整说明文本及截图（如适用），以及报酬细节（如有）？

**回答：** [NA（不适用）]

**理由：** 本文不涉及众包，也不涉及人类受试者研究。

**指南：**

- 回答 `NA` 表示论文不涉及众包或人类受试者研究。
- 可以将这些信息放在补充材料中；但如果论文的主要贡献涉及人类受试者，则应尽可能在论文正文中提供详细信息。
- 根据 NeurIPS 伦理准则，参与数据收集、整理或其他劳动的工作者，应至少获得数据收集者所在国家规定的最低工资。

## 15. 人类受试者研究的机构审查委员会（IRB）批准或同等批准

**问题：** 论文是否说明了研究参与者可能面临的风险、是否已向受试者披露这些风险，以及是否获得机构审查委员会（IRB）的批准（或依据所在国家或机构要求取得的同等批准/审查）？

**回答：** [NA（不适用）]

**理由：** 本文不涉及众包，也不涉及人类受试者研究。

**指南：**

- 回答 `NA` 表示论文不涉及众包或人类受试者研究。
- 根据研究所在国家的规定，任何人类受试者研究都可能需要获得 IRB 批准（或同等批准）。如果已获得 IRB 批准，应在论文中明确说明。
- 我们认识到，不同机构和地区的相关程序可能存在显著差异；我们期望作者遵守 NeurIPS 伦理准则及其所在机构的指南。
- 初次投稿时，不要包含任何会破坏匿名性的信息（如适用），例如执行审查的机构名称。

## 16. LLM 使用声明

**问题：** 如果 LLM 是本研究核心方法中的重要、原创或非标准组成部分，论文是否说明了 LLM 的使用情况？请注意，如果 LLM 仅用于写作、编辑或格式整理，不影响研究的核心方法、科学严谨性或原创性，则无需声明。

**回答：** [Yes（是）]

**理由：** 附录 D 和 D.1 已说明并讨论 LLM 的使用情况。

**指南：**

- 回答 `NA` 表示本研究的核心方法开发并未将 LLM 作为任何重要、原创或非标准组成部分。
- 关于哪些内容应当或不应当说明，请参阅我们的 [LLM 政策](https://neurips.cc/Conferences/2025/LLM)。
