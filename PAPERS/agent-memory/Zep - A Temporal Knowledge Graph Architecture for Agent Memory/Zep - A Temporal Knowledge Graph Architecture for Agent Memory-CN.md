# Zep：一种面向智能体记忆的时序知识图谱架构

**原文题目：** *Zep: A Temporal Knowledge Graph Architecture for Agent Memory*  
**作者：** Preston Rasmussen、Pavlo Paliychuk、Travis Beauvais、Jack Ryan、Daniel Chalef（Zep AI）  
**原文版本：** arXiv:2501.13956v1 [cs.CL]，2025 年 1 月 20 日  
**译注：** 本文按原文结构完整翻译。模型名、系统名、代码标识符与提示词标签保留原文；参考文献题名不译，以避免检索歧义。原文第 2.2.1、2.2.2 节开头及附录 6.1.1 第 6 条存在缺文，已在相应位置说明。

## 摘要

我们提出 Zep——一种面向 AI 智能体的新型记忆层服务；在深度记忆检索（Deep Memory Retrieval，DMR）基准上，它超越了当前最先进的系统 MemGPT。此外，在比 DMR 更全面、更具挑战性且更能反映真实企业应用场景的评测中，Zep 也表现出色。现有面向大语言模型（LLM）智能体的检索增强生成（RAG）框架局限于静态文档检索，而企业应用则需要动态整合来自多种来源的知识，包括持续进行的对话与业务数据。Zep 通过其核心组件 Graphiti 解决了这一根本局限。Graphiti 是一个具备时序感知能力的知识图谱引擎，能够动态综合非结构化对话数据与结构化业务数据，同时保留历史关系。在 MemGPT 团队作为主要评测指标建立的 DMR 基准上，Zep 展现出更优的性能（94.8% 对 93.4%）。除 DMR 外，Zep 的能力还通过难度更高的 LongMemEval 基准得到进一步验证；该基准借助复杂的时序推理任务，更贴近企业应用场景。在这项评测中，与基线实现相比，Zep 将准确率最多提升 18.5%，同时将响应延迟降低 90%。在跨会话信息综合和长期上下文维持等对企业至关重要的任务上，这些优势尤其显著，表明 Zep 能够有效部署于真实世界应用。

## 1 引言

近年来，基于 Transformer 的大语言模型（LLM）对产业界与研究界产生的影响备受关注 [1]。LLM 的一项主要应用是开发基于聊天的智能体。然而，这些智能体的能力受到 LLM 上下文窗口、有效利用上下文的能力以及预训练所得知识的限制。因此，需要提供额外上下文，以补充分布外（out-of-domain，OOD）知识并减少幻觉。

检索增强生成（Retrieval-Augmented Generation，RAG）已成为 LLM 应用中的一个重要研究方向。RAG 利用过去五十年发展起来的信息检索（Information Retrieval，IR）技术 [2]，向 LLM 提供所需的领域知识。

当前使用 RAG 的方法主要面向宽泛的领域知识和基本静态的语料库——也就是说，加入语料库之后，文档内容很少发生变化。若要让智能体广泛进入我们的日常生活，并自主解决从琐碎到高度复杂的各类问题，它们就需要访问一个持续演化的大型数据语料库；其中不仅要包含用户与智能体交互产生的数据，还要包含相关的业务数据和现实世界数据。我们认为，赋予智能体这种广泛且动态的“记忆”，是实现上述愿景不可或缺的基础模块；而当前的 RAG 方法并不适合这一未来。由于完整的对话历史、业务数据集及其他特定领域内容无法有效装入 LLM 的上下文窗口，智能体记忆需要采用新的方法。为 LLM 驱动的智能体增加记忆并非新概念——MemGPT [3] 已对此进行过探索。

近来，知识图谱（Knowledge Graph，KG）被用于增强 RAG 架构，以解决传统 IR 技术的诸多不足 [4]。本文介绍 Zep [5]：一种由 Graphiti [6] 驱动的记忆层服务。Graphiti 是一个动态、具备时序感知能力的知识图谱引擎。Zep 能够摄取并综合非结构化消息数据和结构化业务数据。Graphiti 知识图谱引擎以无损方式用新信息动态更新知识图谱，维护事实和关系的时间线，包括它们各自的有效期。借助这种方法，知识图谱能够表示一个复杂且不断演化的世界。

鉴于 Zep 是生产系统，我们尤其重视其记忆检索机制的准确性、延迟与可扩展性。我们采用两个现有基准来评估这些机制的有效性：MemGPT [3] 提出的深度记忆检索任务（DMR），以及 LongMemEval 基准 [7]。

## 2 知识图谱构建

在 Zep 中，记忆由具备时序感知能力的动态知识图谱 $G=(N,E,\phi)$ 驱动，其中 $N$ 表示节点，$E$ 表示边，$\phi:E\rightarrow N\times N$ 表示形式化关联函数。该图包含三个具有层级关系的子图层次：情景子图、语义实体子图和社区子图。

- **情景子图 $G_e$：** 情景节点（episode）$n_i\in N_e$ 以消息、文本或 JSON 的形式保存原始输入数据。情景充当无损数据存储，语义实体与关系均从中抽取。情景边 $e_i\in E_e\subseteq\phi^*(N_e\times N_s)$ 将情景连接至其所指涉的语义实体。
- **语义实体子图 $G_s$：** 语义实体子图构建在情景子图之上。实体节点 $n_i\in N_s$ 表示从情景中抽取、并与图中既有实体完成消歧解析的实体。实体边（语义边）$e_i\in E_s\subseteq\phi^*(N_s\times N_s)$ 表示从情景中抽取出的实体间关系。
- **社区子图 $G_c$：** 社区子图构成 Zep 知识图谱的最高层。社区节点 $n_i\in N_c$ 表示由强连接实体构成的簇。社区保存这些实体簇的高层摘要，并以更全面、互联程度更高的方式表示 $G_s$ 的结构。社区边 $e_i\in E_c\subseteq\phi^*(N_c\times N_s)$ 将社区与其成员实体连接起来。

同时存储原始情景数据与派生的语义实体信息，呼应了人类记忆的心理学模型。这些模型区分情景记忆与语义记忆：前者表示彼此有别的事件，后者则捕捉概念及其含义之间的关联 [8]。采用这种方法，使用 Zep 的 LLM 智能体能够形成更复杂、更细致的记忆结构，也更符合我们对人类记忆系统的理解。知识图谱是表示这类记忆结构的有效载体；我们将情景子图与语义子图区分开来的实现，借鉴了 AriGraph [9] 中的类似方法。

我们利用社区节点表示高层结构和领域概念，这建立在 GraphRAG [4] 的工作之上，使系统能够更全面地从全局理解相关领域。最终形成的层级组织——从情景到事实、实体，再到社区——拓展了现有的层级式 RAG 策略 [10][11]。

### 2.1 情景

Zep 的图构建过程始于摄取被称为“情景”（Episode）的原始数据单元。情景可分为三种核心类型：消息、文本或 JSON。尽管图构建时每种类型都需要特定处理，但本文聚焦于消息类型，因为我们的实验以对话记忆为中心。在本文语境中，一条消息由相对较短的文本（一个 LLM 上下文窗口可容纳若干条消息）以及说出该话语的相应参与者组成。

每条消息都包含参考时间戳 $t_{ref}$，指明消息的发送时间。借助这项时间信息，Zep 能够准确识别并抽取消息内容中提及的相对日期或不完整日期（例如“下周四”“两周后”或“去年夏天”）。Zep 实现了双时态模型：时间线 $T$ 表示事件的时间顺序，时间线 $T'$ 表示 Zep 摄取数据的事务顺序。$T'$ 时间线承担传统的数据库审计功能，而 $T$ 时间线则提供一个额外维度，用于建模对话数据和记忆的动态性质。与以往基于图的 RAG 方案相比，这种双时态方法是基于 LLM 构建知识图谱的一项新进展，也是 Zep 多项独特能力的基础。

情景边 $E_e$ 将情景连接至从中抽取出的实体节点。情景及其派生语义边维护双向索引，用以追踪边与其源情景之间的关系。该设计支持正向和反向遍历，进一步强化了 Graphiti 情景子图的无损特性：一方面，可以将语义产物追溯至源数据，以便引用或逐字引述；另一方面，情景可以快速检索相关实体和事实。虽然本文实验并未直接考察这些连接，但未来工作将对此展开研究。

### 2.2 语义实体与事实

#### 2.2.1 实体

实体抽取是处理情景的初始阶段。摄取数据时，系统同时处理当前消息内容和最近 $n$ 条消息，为命名实体识别提供上下文。在本文及 Zep 的通用实现中，$n=4$，即提供两个完整的对话轮次用于上下文评估。由于我们聚焦于消息处理，系统会自动将说话者抽取为实体。完成初步实体抽取后，我们采用受 Reflexion [12] 启发的反思技术，以尽量减少幻觉并提高抽取覆盖率。系统还会从情景中抽取实体摘要，以便后续执行实体消歧与检索操作。

抽取之后，系统将每个实体名称嵌入到 1,024 维向量空间。借助该嵌入，系统可以在既有图实体节点中通过余弦相似度搜索检索相似节点。系统还会针对既有实体的名称与摘要单独执行全文搜索，以识别其他候选节点。随后，这些候选节点连同情景上下文一起，通过实体消歧提示词交由 LLM 处理。如果系统识别出重复实体，就会生成更新后的名称与摘要。

完成实体抽取与消歧之后，系统使用预定义的 Cypher 查询将数据并入知识图谱。我们没有采用由 LLM 生成数据库查询的方式，是为了保证模式格式一致，并降低产生幻觉的可能性。

附录给出了部分图构建提示词。

> **译注：** 原 PDF 和 arXiv HTML 均将本小节首词排为“ntity”，缺少首字母 `E`；这里依据小节主题与语法还原为 “Entity”。

#### 2.2.2 事实

原文在此处残缺，仅存：“……针对包含其关键谓词的每项事实。”此句在原 PDF 与 arXiv HTML 中均以 “or each fact containing its key predicate.” 起始，显然缺少句首乃至更前面的上下文；作者当前公开实现也不足以可靠复原，因此本译文不作臆补。需要特别指出的是，同一事实可以在不同实体之间被多次抽取，这使 Graphiti 能够通过一种超边实现来建模复杂的多实体事实。

抽取完成后，系统为事实生成嵌入，为将其整合进图做好准备。系统通过类似实体消歧的流程对边进行去重。对相关边执行混合搜索时，搜索范围被限制为与拟新增边连接相同实体对的既有边。这一约束不仅可防止把不同实体之间的相似边错误组合起来，还通过将搜索空间限制为与特定实体对相关的边子集，显著降低了去重过程的计算复杂度。

#### 2.2.3 时间信息抽取与边失效

与其他知识图谱引擎相比，Graphiti 的一项关键差异化特征，是它能够通过时间信息抽取与边失效流程管理动态信息更新。

系统使用 $t_{ref}$ 从情景上下文中抽取与事实有关的时间信息。由此，它既可以准确抽取和表示绝对时间戳（例如“Alan Turing 出生于 1912 年 6 月 23 日”），也可以处理相对时间戳（例如“我在两周前开始了新工作”）。与双时态建模方法一致，系统追踪四个时间戳：$t'_{created}$ 和 $t'_{expired}\in T'$ 监测事实在系统中何时创建或失效；$t_{valid}$ 和 $t_{invalid}\in T$ 则追踪事实成立的时间范围。这些时间数据点与其他事实信息一同存储在边上。

引入新边可能使数据库中的既有边失效。系统使用 LLM，将新边与语义相关的既有边进行比较，以识别潜在矛盾。识别出时间范围重叠的矛盾时，系统把受影响边的 $t_{invalid}$ 设置为导致其失效之边的 $t_{valid}$，从而令这些边失效。按照事务时间线 $T'$，Graphiti 在判断边是否失效时始终优先采用新信息。

这套完整方法使 Graphiti 能随对话演进动态加入数据，同时保留关系的当前状态以及关系随时间演变的历史记录。

### 2.3 社区

建立情景子图与语义子图后，系统通过社区检测来构建社区子图。我们的社区检测方法以 GraphRAG [4] 所述技术为基础，但使用标签传播算法 [13]，而不是 Leiden 算法 [14]。选择标签传播，是因为它便于直接进行动态扩展：随着新数据进入图中，系统能够在更长时间内维持准确的社区表示，推迟完整刷新社区的时点。

这种动态扩展实现了标签传播中单次递归步骤的逻辑。系统向图中加入新实体节点 $n_i\in N_s$ 时，会考察其邻居节点所属的社区。系统把新节点分配给其邻居中成员数最多的社区，随后相应更新社区摘要和图。虽然动态更新可以随着数据流入而高效扩展社区，但所得社区会逐渐偏离完整运行一次标签传播所生成的社区。因此，仍需定期刷新社区。不过，这种动态更新策略提供了一种实用的启发式方法，能够显著降低延迟和 LLM 推理成本。

沿用文献 [4] 的做法，我们的社区节点包含通过迭代式 MapReduce 摘要流程从成员节点导出的摘要。但我们的检索方法与 GraphRAG 的 MapReduce 方法 [4] 有很大不同。为支持我们的检索方法，我们根据社区摘要生成社区名称，其中包含关键术语和相关主题。随后对这些名称进行嵌入并存储，以支持余弦相似度搜索。

## 3 记忆检索

Zep 的记忆检索系统提供了强大、复杂且高度可配置的功能。从高层来看，Zep 图搜索 API 实现函数 $f:S\rightarrow S$：它接收文本字符串查询 $\alpha\in S$ 作为输入，并返回文本字符串上下文 $\beta\in S$ 作为输出。输出 $\beta$ 包含来自节点和边的格式化数据，LLM 智能体需要这些数据才能为查询 $\alpha$ 生成准确回答。过程 $f(\alpha)\rightarrow\beta$ 包含三个不同步骤：

- **搜索（$\varphi$）：** 首先识别可能包含相关信息的候选节点与候选边。Zep 虽然采用多种不同搜索方法，但总体搜索函数可表示为 $\varphi:S\rightarrow E_s^n\times N_s^n\times N_c^n$。因此，$\varphi$ 将查询转换为一个三元组，其中包含语义边列表、实体节点列表和社区节点列表——这三类图对象均保存相关文本信息。
- **重排序器（$\rho$）：** 第二步重新排列搜索结果。重排序函数或模型接收一个搜索结果列表，并输出重新排序后的版本：$\rho:\varphi(\alpha),\ldots\rightarrow E_s^n\times N_s^n\times N_c^n$。
- **构造器（$\chi$）：** 最后，构造器把相关节点和边转换为文本上下文：$\chi:E_s^n\times N_s^n\times N_c^n\rightarrow S$。对于每条 $e_i\in E_s$，$\chi$ 返回 `fact`、$t_{valid}$ 和 $t_{invalid}$ 字段；对于每个 $n_i\in N_s$，返回 `name` 和 `summary` 字段；对于每个 $n_i\in N_c$，返回 `summary` 字段。

有了上述定义，$f$ 可表示为这三个组件的复合：

$$f(\alpha)=\chi(\rho(\varphi(\alpha)))=\beta$$

上下文字符串模板示例：

```text
FACTS 和 ENTITIES 表示与当前对话相关的上下文。
以下是相关性最高的事实及其有效日期范围。如果事实涉及某个事件，则该事件发生在此时间段内。
格式：FACT（日期范围：从 - 到）
<FACTS>
{facts}
</FACTS>
以下是相关性最高的实体
ENTITY_NAME: 实体摘要
<ENTITIES>
{entities}
</ENTITIES>
```

### 3.1 搜索

Zep 实现了三种搜索函数：余弦语义相似度搜索（$\varphi_{cos}$）、Okapi BM25 全文搜索（$\varphi_{bm25}$）和广度优先搜索（$\varphi_{bfs}$）。前两种函数使用 Neo4j 对 Lucene 的实现 [15][16]。各搜索函数在识别相关文档方面各具能力；三者结合，可在重排序之前全面覆盖候选结果。三类对象对应的搜索字段有所不同：对于 $E_s$，搜索 `fact` 字段；对于 $N_s$，搜索实体名称；对于 $N_c$，则搜索社区名称——社区名称由该社区所涵盖的相关关键词和短语构成。尽管彼此独立开发，我们的社区搜索方法与 LightRAG [17] 的高层关键词搜索方法相似。将 LightRAG 的方法与 Graphiti 等基于图的系统混合，是一个很有前景的未来研究方向。

余弦相似度和全文搜索方法已在 RAG 领域得到广泛确立 [18]；相比之下，基于知识图谱的广度优先搜索在 RAG 领域受到的关注较少，AriGraph [9] 和 Distill-SynthKG [19] 等基于图的 RAG 系统属于少数显著例外。在 Graphiti 中，广度优先搜索通过识别 $n$ 跳范围内的其他节点和边来扩充初始搜索结果。此外，$\varphi_{bfs}$ 可以接收节点作为搜索参数，从而提高对搜索函数的控制力。将近期情景作为广度优先搜索的种子时，这项功能尤其有价值，因为系统可以把近期提到的实体和关系纳入检索上下文。

三种搜索方法分别针对不同的相似性维度：全文搜索识别词语相似性，余弦相似度捕捉语义相似性，广度优先搜索则揭示上下文相似性——图中距离越近的节点和边，往往出现在越相似的对话上下文中。这种多维度候选结果识别方法，最大限度提高了找到最佳上下文的可能性。

### 3.2 重排序器

初始搜索方法旨在取得高召回率，而重排序器则通过优先排列最相关的结果来提高精确率。Zep 支持倒数排序融合（Reciprocal Rank Fusion，RRF）[20]、最大边际相关性（Maximal Marginal Relevance，MMR）[21] 等现有重排序方法。此外，Zep 还实现了一种基于图的“情景提及”重排序器，它根据对话中实体或事实被提及的频率确定结果优先级，从而使经常被提及的信息更容易访问。系统还包含节点距离重排序器，根据结果与指定中心节点的图距离重新排序，以提供局部聚焦于知识图谱特定区域的上下文。系统最复杂的重排序能力使用交叉编码器：这类 LLM 利用交叉注意力，结合查询评估节点和边并生成相关性分数，但这种方法的计算成本也最高。

## 4 实验

本节分析基于两个 LLM 记忆基准开展的实验。第一项评测采用文献 [3] 提出的深度记忆检索（DMR）任务；该任务使用了“Beyond Goldfish Memory: Long-Term Open-Domain Conversation”[22] 所提出 Multi-Session Chat 数据集中的 500 段对话子集。第二项评测采用“LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory”[7] 提出的 LongMemEval 基准。具体而言，我们使用 LongMemEvals 数据集；其提供的对话上下文平均约为 115,000 个 token。

在两项实验中，我们均通过 Zep API 将对话历史整合进 Zep 知识图谱。随后使用第 3 节所述技术，检索相关性最高的 20 条边（事实）和实体节点（实体摘要）。系统将这些数据重新格式化为上下文字符串，与 Zep 记忆 API 提供的功能一致。

这些实验虽然展示了 Graphiti 的关键检索能力，但只覆盖系统完整搜索功能的一个子集。聚焦这一范围，既便于与现有基准作清晰比较，也将其他知识图谱能力留待未来工作探索。

### 4.1 模型选择

我们的实验实现使用 BAAI 的 BGE-m3 模型执行重排序与嵌入任务 [23][24]。在图构建和回答生成方面，使用 `gpt-4o-mini-2024-07-18` 构建图，并分别使用 `gpt-4o-mini-2024-07-18` 与 `gpt-4o-2024-11-20` 作为聊天智能体，根据所提供的上下文生成回答。

为确保能够与 MemGPT 的 DMR 结果直接比较，我们还使用 `gpt-4-turbo-2024-04-09` 进行了 DMR 评测。

实验笔记本将通过我们的 GitHub 仓库公开，相关实验提示词收录于附录。

### 4.2 深度记忆检索（DMR）

文献 [3] 提出的深度记忆检索评测包含 500 段多会话对话；每段对话由 5 个聊天会话组成，每个会话最多包含 12 条消息。每段对话均包含一个用于记忆评测的问题/答案对。MemGPT 框架 [3] 使用 `gpt-4-turbo` 达到 93.4% 的准确率，是当时该性能指标的领先者；相较之下，递归摘要基线仅为 35.3%。

为建立比较基线，我们实现了两种常见的 LLM 记忆方法：完整对话上下文与会话摘要。使用 `gpt-4-turbo` 时，完整对话基线取得 94.4% 的准确率，略高于 MemGPT 报告的结果；会话摘要基线则为 78.6%。使用 `gpt-4o-mini` 时，两种方法的性能均有所提高：完整对话为 98.0%，会话摘要为 88.0%。由于 MemGPT 已发表工作提供的方法细节不足，我们无法使用 `gpt-4o-mini` 复现其结果。

随后，我们将对话摄取进 Zep，并用其搜索函数检索相关性最高的 10 个节点与 10 条边，以此评测 Zep 的性能。LLM 裁判将智能体回答与给定标准答案进行比较。使用 `gpt-4-turbo` 时，Zep 准确率为 94.8%；使用 `gpt-4o-mini` 时为 98.2%，相较 MemGPT 及各自对应的完整对话基线均有小幅提升。不过，必须结合具体背景理解这些结果：每段对话只有 60 条消息，可以轻松容纳于当前 LLM 的上下文窗口之中。

**表 1　深度记忆检索**

| 记忆方法 | 模型 | 得分 |
|---|---:|---:|
| 递归摘要† | gpt-4-turbo | 35.3% |
| 对话摘要 | gpt-4-turbo | 78.6% |
| MemGPT† | gpt-4-turbo | 93.4% |
| 完整对话 | gpt-4-turbo | 94.4% |
| Zep | gpt-4-turbo | 94.8% |
| 对话摘要 | gpt-4o-mini | 88.0% |
| 完整对话 | gpt-4o-mini | 98.0% |
| Zep | gpt-4o-mini | 98.2% |

† 文献 [3] 报告的结果。

DMR 评测的局限并不止于规模过小。我们的分析揭示了该基准设计上的重大缺陷。评测仅包含单轮事实检索问题，无法考察对记忆的复杂理解。许多问题措辞含糊，例如提到“放松时最喜欢的饮料”或“古怪爱好”，但对话中并未明确将相关内容表述为这类概念。最关键的是，该数据集无法充分代表 LLM 智能体在真实世界中的企业应用场景。使用现代 LLM 的简单完整上下文方法也能取得很高性能，这进一步凸显了该基准不适合评估记忆系统。

文献 [7] 的发现进一步说明了这一不足：随着对话长度增加，LLM 在 LongMemEval 基准上的性能会迅速下降。LongMemEval 数据集 [7] 提供更长、更连贯且更能反映企业场景的对话，并包含更多样的评测问题，从而解决了上述诸多不足。

### 4.3 LongMemEval（LME）

我们使用 LongMemEvals 数据集评测 Zep。该数据集提供了可代表 LLM 智能体真实业务应用的对话和问题。LongMemEvals 给现有 LLM 和商业记忆解决方案带来显著挑战 [7]；其中对话平均长度约为 115,000 个 token。这个长度虽然可观，但仍处于当前前沿模型的上下文窗口范围内，因而使我们能够为评估 Zep 性能建立有意义的基线。

数据集包含六种不同的问题类型：单会话-用户、单会话-助手、单会话-偏好、多会话、知识更新和时间推理。这些类别在数据集中的分布并不均匀；详细分布信息请参阅文献 [7]。

所有实验均于 2024 年 12 月至 2025 年 1 月间进行。测试使用一台消费级笔记本电脑，地点为美国马萨诸塞州波士顿的一处住宅，通过网络连接部署在 AWS `us-west-2` 区域的 Zep 服务。这种分布式架构在评估 Zep 性能时引入了额外网络延迟，而基线评测并不存在这部分延迟。

在答案评估方面，我们使用 GPT-4o，并采用文献 [7] 针对不同问题提供的提示词；这些提示词的评估结果已被证明与人工评估者高度相关。

#### 4.3.1 LongMemEval 与 MemGPT

为在 Zep 与当前最先进的 MemGPT 系统 [3] 之间建立比较基准，我们尝试使用 LongMemEval 数据集评估 MemGPT。鉴于当前 MemGPT 框架不支持直接摄取既有消息历史，我们采取一种变通方案，将对话消息加入归档历史。然而，采用该方法后，我们未能使系统成功回答问题。我们期待其他研究团队对这一基准开展评测，因为比较性性能数据将有益于整个 LLM 记忆系统领域的发展。

#### 4.3.2 LongMemEval 结果

对两个模型变体而言，Zep 在准确率与延迟方面相较基线均有显著改进。使用 `gpt-4o-mini` 时，Zep 的准确率相对基线提升 15.2%；使用 `gpt-4o` 时则提升 18.5%。提示词规模缩小，与基线实现相比还显著降低了延迟成本。

**表 2　LongMemEvals**

| 记忆方法 | 模型 | 得分 | 延迟 | 延迟四分位距 | 平均上下文 token 数 |
|---|---|---:|---:|---:|---:|
| 完整上下文 | gpt-4o-mini | 55.4% | 31.3 秒 | 8.76 秒 | 115k |
| Zep | gpt-4o-mini | 63.8% | 3.20 秒 | 1.31 秒 | 1.6k |
| 完整上下文 | gpt-4o | 60.2% | 28.9 秒 | 6.01 秒 | 115k |
| Zep | gpt-4o | 71.2% | 2.58 秒 | 0.684 秒 | 1.6k |

按问题类型分析可见，采用 Zep 的 `gpt-4o-mini` 在六个类别中的四个类别上有所提升，其中复杂问题类型的提升最大：单会话-偏好、多会话和时间推理。使用 `gpt-4o` 时，Zep 在知识更新类别上也进一步体现出性能改善，凸显了它配合能力更强的模型时的有效性。不过，要让能力较弱的模型更好地理解 Zep 的时间数据，可能还需进一步开发。

**表 3　LongMemEvals 按问题类型细分**

| 问题类型 | 模型 | 完整上下文 | Zep | 变化幅度 |
|---|---|---:|---:|---:|
| 单会话-偏好 | gpt-4o-mini | 30.0% | 53.3% | ↑77.7% |
| 单会话-助手 | gpt-4o-mini | 81.8% | 75.0% | ↓9.06% |
| 时间推理 | gpt-4o-mini | 36.5% | 54.1% | ↑48.2% |
| 多会话 | gpt-4o-mini | 40.6% | 47.4% | ↑16.7% |
| 知识更新 | gpt-4o-mini | 76.9% | 74.4% | ↓3.36% |
| 单会话-用户 | gpt-4o-mini | 81.4% | 92.9% | ↑14.1% |
| 单会话-偏好 | gpt-4o | 20.0% | 56.7% | ↑184% |
| 单会话-助手 | gpt-4o | 94.6% | 80.4% | ↓17.7% |
| 时间推理 | gpt-4o | 45.1% | 62.4% | ↑38.4% |
| 多会话 | gpt-4o | 44.3% | 57.9% | ↑30.7% |
| 知识更新 | gpt-4o | 78.2% | 83.3% | ↑6.52% |
| 单会话-用户 | gpt-4o | 81.4% | 92.9% | ↑14.1% |

这些结果表明，Zep 能够在不同模型规模上提升性能；与能力更强的模型搭配时，在复杂、细致的问题类型上提升最为显著。延迟改善尤其值得注意：Zep 在保持更高准确率的同时，将响应时间缩短约 90%。

单会话-助手问题的性能下降——`gpt-4o` 下降 17.7%，`gpt-4o-mini` 下降 9.06%——是 Zep 整体稳定提升趋势中的一个显著例外，表明还需开展进一步研究与工程工作。

## 5 结论

本文提出 Zep，一种基于图的 LLM 记忆方法，将语义记忆与情景记忆同实体摘要、社区摘要结合起来。评测表明，Zep 在现有记忆基准上达到最先进性能，同时降低了 token 成本，并以显著更低的延迟运行。

Graphiti 与 Zep 所取得的结果虽然令人瞩目，但很可能只是基于图的记忆系统初期进展。多个研究方向可以在这些框架上继续发展，包括把其他 GraphRAG 方法整合进 Zep 范式，以及对本文工作进行新的拓展。

研究已经证明，在 GraphRAG 范式中，针对 LLM 实体抽取与边抽取进行微调的模型具有价值，既能提高准确率，也能降低成本和延迟 [19][25]。针对 Graphiti 提示词进行类似微调的模型，可能会改善知识抽取，尤其是在复杂对话中。此外，当前关于 LLM 生成知识图谱的研究大多未采用形式化本体 [9][4][17][19][26]，但领域特定本体具有巨大潜力。图本体是 LLM 出现之前知识图谱工作的基础，值得在 Graphiti 框架内进一步探索。

我们在寻找合适的记忆基准时发现，可选方案十分有限：现有基准往往缺乏稳健性与复杂性，常常退化为简单的“大海捞针”式事实检索问题 [3]。该领域需要更多记忆基准，尤其是能够反映客户体验任务等业务应用的基准，从而有效评估并区分不同记忆方法。值得注意的是，现有基准都无法充分评测 Zep 对对话历史与结构化业务数据进行处理和综合的能力。Zep 虽然聚焦于 LLM 记忆，但其传统 RAG 能力也应使用文献 [17]、[27] 和 [28] 中的成熟基准进行评估。

当前关于 LLM 记忆与 RAG 系统的文献，并未充分讨论生产系统在成本和延迟方面的可扩展性。借鉴 LightRAG 作者重视这些指标的做法，我们纳入了检索机制的延迟基准，希望开始弥补这一缺口。

## 6 附录

### 6.1 图构建提示词

#### 6.1.1 实体抽取

```text
<PREVIOUS MESSAGES>
{previous_messages}
</PREVIOUS MESSAGES>
<CURRENT MESSAGE>
{current_message}
</CURRENT MESSAGE>

给定上述对话，从 CURRENT MESSAGE 中抽取被明确或隐含提及的实体节点：

指南：
1. 始终将说话者/参与者抽取为第一个节点。说话者是每行对话中冒号之前的部分。
2. 抽取 CURRENT MESSAGE 中提及的其他重要实体、概念或参与者。
3. 不要为关系或动作创建节点。
4. 不要为日期、时间或年份等时间信息创建节点（这些信息稍后将加入边中）。
5. 节点名称应尽可能明确，并使用完整名称。
6. 不要抽取仅在……中提及的实体 [原文至此截断]
```

> **译注：** 第 6 条在原 PDF 与 arXiv HTML 中都截断于 “DO NOT extract entities mentioned only”。作者当前公开的 Graphiti 实现含有意思相近但并不相同的新版提示词，不能据此精确补写论文原句。

#### 6.1.2 实体消歧

```text
<PREVIOUS MESSAGES>
{previous_messages}
</PREVIOUS MESSAGES>
<CURRENT MESSAGE>
{current_message}
</CURRENT MESSAGE>
<EXISTING NODES>
{existing_nodes}
</EXISTING NODES>

给定上述 EXISTING NODES、MESSAGE 和 PREVIOUS MESSAGES，判断从对话中抽取的 NEW NODE 是否与 EXISTING NODES 中的某一实体重复。

<NEW NODE>
{new_node}
</NEW NODE>

任务：
1. 如果 New Node 与 Existing Nodes 中的任一节点表示同一实体，则在响应中返回 'is_duplicate: true'；否则返回 'is_duplicate: false'。
2. 如果 is_duplicate 为 true，还要在响应中返回既有节点的 uuid。
3. 如果 is_duplicate 为 true，为节点返回一个最完整的全名。

指南：
1. 同时使用节点的名称和摘要判断实体是否重复；重复节点可能具有不同名称。
```

#### 6.1.3 事实抽取

```text
<PREVIOUS MESSAGES>
{previous_messages}
</PREVIOUS MESSAGES>
<CURRENT MESSAGE>
{current_message}
</CURRENT MESSAGE>
<ENTITIES>
{entities}
</ENTITIES>

给定上述 MESSAGES 和 ENTITIES，从 CURRENT MESSAGE 中抽取与所列 ENTITIES 有关的全部事实。

指南：
1. 仅抽取所提供实体之间的事实。
2. 每项事实都应表示两个不同节点之间的明确关系。
3. relation_type 应当是对事实的简洁、全大写描述（例如 LOVES、IS_FRIENDS_WITH、WORKS_FOR）。
4. 提供包含全部相关信息的、更为详细的事实描述。
5. 在相关时考虑关系的时间属性。
```

#### 6.1.4 事实消歧

```text
给定以下上下文，判断 New Edge 是否表示 Existing Edges 列表中的任一条边。

<EXISTING EDGES>
{existing_edges}
</EXISTING EDGES>
<NEW EDGE>
{new_edge}
</NEW EDGE>

任务：
1. 如果 New Edge 与 Existing Edges 中的任一条边表示相同的事实信息，则在响应中返回 'is_duplicate: true'；否则返回 'is_duplicate: false'。
2. 如果 is_duplicate 为 true，还要在响应中返回既有边的 uuid。

指南：
1. 两项事实不必完全相同才算重复；只要表达相同信息即可。
```

#### 6.1.5 时间信息抽取

```text
<PREVIOUS MESSAGES>
{previous_messages}
</PREVIOUS MESSAGES>
<CURRENT MESSAGE>
{current_message}
</CURRENT MESSAGE>
<REFERENCE TIMESTAMP>
{reference_timestamp}
</REFERENCE TIMESTAMP>
<FACT>
{fact}
</FACT>

重要：仅当时间信息属于给定事实的一部分时才抽取它，否则忽略所提及的时间。
若仅提及相对时间（例如 10 年前、2 分钟前），请根据给定参考时间戳尽最大努力确定具体日期。
如果关系不具有跨时间段的性质，但仍能确定日期，则只设置 valid_at。

定义：
- valid_at：边所描述的事实关系开始成立或建立的日期与时间。
- invalid_at：边所描述的事实关系不再成立或终止的日期与时间。

任务：
分析对话，判断是否存在属于边事实一部分的日期。仅当日期明确关系到该关系本身的形成或改变时，才设置日期。

指南：
1. 日期时间采用 ISO 8601 格式（YYYY-MM-DDTHH:MM:SS.SSSSSSZ）。
2. 确定 valid_at 和 invalid_at 日期时，将参考时间戳作为当前时间。
3. 如果事实使用现在时，则将 Reference Timestamp 用作 valid_at 日期。
4. 如果没有发现能够确定或改变该关系的时间信息，将字段留为 null。
5. 不要从相关事件推断日期。只使用直接说明该关系建立或改变的日期。
6. 对与关系直接相关的相对时间表述，根据参考时间戳计算实际日期时间。
7. 如果只提及日期而没有具体时间，则使用该日的 00:00:00（午夜）。
8. 如果只提及年份，则使用该年 1 月 1 日 00:00:00。
9. 始终包含时区偏移量（若未提及具体时区，则使用 Z 表示 UTC）。
```

## 参考文献

[1] Ashish Vaswani, Noam Shazeer, Niki Parmar, Jakob Uszkoreit, Llion Jones, Aidan N. Gomez, Lukasz Kaiser, and Illia Polosukhin. *Attention is all you need*, 2023.

[2] K. Sparck Jones. A statistical interpretation of term specificity and its application in retrieval. *Journal of Documentation*, 28(1):11–21, 1972.

[3] Charles Packer, Sarah Wooders, Kevin Lin, Vivian Fang, Shishir G. Patil, Ion Stoica, and Joseph E. Gonzalez. *MemGPT: Towards LLMs as operating systems*, 2024.

[4] Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt, and Jonathan Larson. *From local to global: A graph RAG approach to query-focused summarization*, 2024.

[5] Zep. *Zep: Long-term memory for AI agents*. https://www.getzep.com, 2024. 面向 AI 应用的商业记忆层。

[6] Zep. *Graphiti: Temporal knowledge graphs for agentic applications*. https://github.com/getzep/graphiti, 2024. Graphiti 构建动态、具备时序感知能力的知识图谱，用以表示实体间随时间变化的复杂演化关系。

[7] Di Wu, Hongwei Wang, Wenhao Yu, Yuwei Zhang, Kai-Wei Chang, and Dong Yu. *LongMemEval: Benchmarking chat assistants on long-term interactive memory*, 2024.

[8] Wong Gonzalez and Daniela. *The relationship between semantic and episodic memory: Exploring the effect of semantic neighbourhood density on episodic memory*. PhD thesis, University of Winsor, 2018.

[9] Petr Anokhin, Nikita Semenov, Artyom Sorokin, Dmitry Evseev, Mikhail Burtsev, and Evgeny Burnaev. *AriGraph: Learning knowledge graph world models with episodic memory for LLM agents*, 2024.

[10] Xinyue Chen, Pengyu Gao, Jiangjiang Song, and Xiaoyang Tan. *HiQA: A hierarchical contextual augmentation RAG for multi-documents QA*, 2024.

[11] Krish Goel and Mahek Chandak. *HiRO: Hierarchical information retrieval optimization*, 2024.

[12] Noah Shinn, Federico Cassano, Edward Berman, Ashwin Gopinath, Karthik Narasimhan, and Shunyu Yao. *Reflexion: Language agents with verbal reinforcement learning*, 2023.

[13] Xiaojin Zhu and Zoubin Ghahramani. *Learning from labeled and unlabeled data with label propagation*, 2002.

[14] V. A. Traag, L. Waltman, and N. J. van Eck. From Louvain to Leiden: guaranteeing well-connected communities. *Scientific Reports* 9, 5233, 2019.

[15] Neo4j. *Neo4j - the world’s leading graph database*, 2012.

[16] Apache Software Foundation. *Apache Lucene - scoring*, 2011. 最后访问：2011 年 10 月 20 日。

[17] Zirui Guo, Lianghao Xia, Yanhua Yu, Tu Ao, and Chao Huang. *LightRAG: Simple and fast retrieval-augmented generation*, 2024.

[18] Jimmy Lin, Ronak Pradeep, Tommaso Teofili, and Jasper Xian. *Vector search with OpenAI embeddings: Lucene is all you need*, 2023.

[19] Prafulla Kumar Choubey, Xin Su, Man Luo, Xiangyu Peng, Caiming Xiong, Tiep Le, Shachar Rosenman, Vasudev Lal, Phil Mui, Ricky Ho, Phillip Howard, and Chien-Sheng Wu. *Distill-SynthKG: Distilling knowledge graph synthesis workflow for improved coverage and efficiency*, 2024.

[20] Gordon V. Cormack, Charles L. A. Clarke, and Stefan Buettcher. Reciprocal rank fusion outperforms Condorcet and individual rank learning methods. In *Proceedings of the 32nd International ACM SIGIR Conference on Research and Development in Information Retrieval*, SIGIR ’09, pages 758–759. ACM, 2009.

[21] Jaime Carbonell and Jade Goldstein. The use of MMR, diversity-based reranking for reordering documents and producing summaries. In *Proceedings of the 21st Annual International ACM SIGIR Conference on Research and Development in Information Retrieval*, SIGIR ’98, pages 335–336, New York, NY, USA, 1998. Association for Computing Machinery.

[22] Jing Xu, Arthur Szlam, and Jason Weston. *Beyond goldfish memory: Long-term open-domain conversation*, 2021.

[23] Chaofan Li, Zheng Liu, Shitao Xiao, and Yingxia Shao. *Making large language models a better foundation for dense retrieval*, 2023.

[24] Jianlv Chen, Shitao Xiao, Peitian Zhang, Kun Luo, Defu Lian, and Zheng Liu. *BGE M3-embedding: Multi-lingual, multi-functionality, multi-granularity text embeddings through self-knowledge distillation*, 2024.

[25] Shreyas Pimpalgaonkar, Nolan Tremelling, and Owen Colegrove. *Triplex: A SOTA LLM for knowledge graph construction*, 2024.

[26] Shilong Li, Yancheng He, Hangyu Guo, Xingyuan Bu, Ge Bai, Jie Liu, Jiaheng Liu, Xingwei Qu, Yangguang Li, Wanli Ouyang, Wenbo Su, and Bo Zheng. *GraphReader: Building graph-based agent to enhance long-context abilities of large language models*, 2024.

[27] Pranab Islam, Anand Kannappan, Douwe Kiela, Rebecca Qian, Nino Scherrer, and Bertie Vidgen. *FinanceBench: A new benchmark for financial question answering*, 2023.

[28] Nandan Thakur, Nils Reimers, Andreas Rücklé, Abhishek Srivastava, and Iryna Gurevych. *BEIR: A heterogenous benchmark for zero-shot evaluation of information retrieval models*, 2021.

## Sources

- `papers/agent-memory/Zep - A Temporal Knowledge Graph Architecture for Agent Memory/Zep - A Temporal Knowledge Graph Architecture for Agent Memory.pdf`
- arXiv HTML（用于核对原文件中的缺文）：https://arxiv.org/html/2501.13956v1
- Graphiti 作者仓库提示词目录（仅用于确认残句不能由当前实现精确补回）：https://github.com/getzep/graphiti/tree/main/graphiti_core/prompts
