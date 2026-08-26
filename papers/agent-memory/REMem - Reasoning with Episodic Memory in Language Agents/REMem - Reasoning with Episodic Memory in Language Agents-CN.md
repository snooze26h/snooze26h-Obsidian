# REMem：语言智能体中的情景记忆推理

Yiheng Shu¹，Saisri Padmaja Jonnalagedda²，Xiang Gao²，Bernal Jiménez Gutiérrez¹  
Weijian Qi¹，Kamalika Das²，Huan Sun¹，Yu Su¹

¹ 俄亥俄州立大学　² Intuit AI Research  
{shu.251, su.809}@osu.edu，{saisri_jonnalagedda, xiang_gao}@intuit.com

> 译注：本文以会议论文形式发表于 ICLR 2026。REMem 是 Reasoning with Episodic Memory（利用情景记忆进行推理）的缩写。方法名、模型名、数据集名及工具/参数名保留英文；术语首次出现时酌情附英文。公式、算法、表格、图注、脚注及作者-年份引文均与原文对应。参考文献保留原始题名与关键书目信息，以便准确检索。原文中的明显编号或措辞异常不作静默修正，必要处以译注标明。

## 摘要

人类擅长沿时空情境记住具体经历，并跨越这些事件进行推理；这正是情景记忆（episodic memory）能力。相比之下，语言智能体中的记忆仍以语义记忆为主，现有智能体尚不能有效回忆交互历史并对其进行推理。针对这一差距，我们识别并形式化了情景回忆与情景推理的核心挑战，同时观察到：现有工作常常忽视情景性，缺少显式事件建模，或过度强调简单检索而非复杂推理。我们提出 REMem——一个分为两个阶段、用于构建情景记忆并利用其进行推理的框架：（1）**索引**：REMem 将经历转换为混合记忆图，灵活连接具有时间感知能力的要旨与事实；（2）**智能体式推断**：REMem 使用配备精心设计工具的智能体式检索器，在记忆图上进行迭代检索。对四个情景记忆基准的全面评估表明，REMem 显著优于 Mem0、HippoRAG 2 等当前最先进的记忆系统，在情景回忆和情景推理任务上分别取得 3.4% 和 13.4% 的绝对提升。此外，面对无法回答的问题，REMem 也表现出更稳健的拒答行为。[^1]

[^1]: 代码和数据见 <https://github.com/intuit-ai-research/REMem>。

## 1 引言

有意且精确地重新体验过去事件的能力，是人类智能的标志性特征之一。这种“心理时间旅行”（mental time travel）能力，也就是情景记忆，使我们能够沿时空轴访问特定事件，明确其顺序、持续时间，乃至因果关系。语义记忆存储关于世界的概念与知识；与之不同，情景记忆塑造每个人独特的经历与偏好，是个体在其环境中持续学习的基石。

尽管情景记忆对人类认知至关重要，而且语言智能体记忆系统日益受到关注，实现人类水平的情景记忆仍遥不可及，主要原因是当前方法受语义记忆范式主导。参数记忆在预训练或微调期间嵌入模型权重，缺乏适应能力，也没有扎根于具体经历的上下文。模型编辑方法（Yao et al., 2023b）可以更新已存事实，却仍局限于修改静态语义知识。使用嵌入模型的检索增强生成（RAG）系统（Zhang et al., 2025; Lee et al., 2025a）能够动态访问知识，但依然以去情境化方式运作，与时空上下文相割裂。最后，更先进的非参数系统使用大语言模型（LLM）构建摘要或语义图（Gutierrez et al., 2024; Gutiérrez et al., 2025; Edge et al., 2024），虽较 RAG 有所改进，却仍优先考虑结构化世界知识，而非亲身经历且特定于交互的经验。

近来，少数研究开始更直接地面向情景设置。它们通常有两种做法：一是在（时序）知识图谱中以实体关系表示情景记忆，却丢失连贯的事件上下文（Rasmussen et al., 2025）；二是选择性推断看似重要的信息并将其作为摘要提供，但缺少显式事件建模（Chhikara et al., 2025; Tan et al., 2025）。关键在于，它们未能在交互历史中整合时间、地点和参与者等情境维度。此外，当记忆需要支持跨多个关联事件的推理时，现有方法高度依赖基于相似度的检索，对复杂事件间关系的推断能力有限。基于这些观察，我们认为，全面的事件表示结合灵活的检索与推理，值得进一步探索。

**图 1：情景记忆评估概览。** 话语被锚定到时间线（上部）。我们评估两种层层递进的能力，并展示各自的平均得分（下部）：（1）**情景回忆**：回忆过去经历中的时间及其他情境要素；在 LoCoMo 和 REALTALK 上以 LLM 评审得分衡量。（2）**情景推理**：以回忆为基础，跨时间线进行多步推理，例如事件到事件关系、计数和序数查询；在 Complex-TR 上以 LLM 评审得分衡量，在 Test of Time 上以精确匹配（EM）得分衡量。

> **图 1 中的示例。** Melanie：“上周我刚带家人去山里露营。我们一家人在一起度过了一段非常美好的时光！”（2023 年 6 月 27 日）；“我昨天刚报名参加陶艺课。对我来说，它就像一种疗愈。”（2023 年 7 月 3 日）；“我昨天带孩子们去了公园。他们探索、玩耍，玩得很开心。”（2023 年 8 月 28 日）。情景回忆问题：“Melanie 什么时候去了公园？”答：“8 月 27 日”；“Melanie 喜欢全家一起露营吗？”答：“是。”情景推理问题：“从 Melanie 报名陶艺课到带孩子去公园，过去了多少天？”答：“56 天。”

为推动语言智能体中的情景记忆与推理，我们首先识别出图 1 所示的两项关键且递进的挑战：（1）**情景回忆**：依据经历重建事件及其情境维度，例如时间、地点、参与者和情绪；换言之，就是将情境要素绑定到特定事件的能力。（2）**情景推理**：以情景回忆为基础进行多步推理，例如处理事件间关系、序数约束和最高级。

随后，我们为语言智能体提出新的情景记忆框架 REMem（Reasoning with Episodic Memory）。我们将情景记忆形式化为具备时间感知能力的事件表示，并提出一种混合记忆图，用于存储要旨（带有解析后时间戳、简洁且便于人类阅读的事件摘要）和事实（具有时间作用域的三元组）。不同于选择性抽取有用信息的现有工作，我们明确指示 LLM 主要沿时间组织记忆，并将其与参与者、地点和情绪等情境维度相连接。我们还开发了一套智能体式推断流程，配备精心设计的检索与图探索工具。由此，记忆管理不再只是匹配孤立文本片段，而能完成复杂的逻辑组合，包括时间范围筛选、邻居探索和序数约束。

我们开展了迄今最全面的情景记忆评估之一，覆盖四个对话理解与时序阅读理解基准。REMem 相较当前最先进方法取得稳定增益，在情景回忆和情景推理任务上分别获得 3.4% 和 13.4% 的绝对提升。REMem 还展现出无可比拟的推理能力：它是 Test of Time 基准（Fatemi et al., 2025）上唯一精确匹配得分超过 90% 的方法，并能更稳健地拒答无法回答的问题。这些强劲结果表明，REMem 是让语言智能体获得有效情景记忆的重要一步。

## 2 相关工作

### 2.1 LLM 的非参数记忆

大量工作通过非参数记忆增强 LLM。ChatGPT（OpenAI, 2025a）将过往聊天与用户可控的已保存记忆相结合，以实现个性化。强嵌入模型（Lee et al., 2025a; Zhang et al., 2025）能够提供有竞争力的检索性能，但其扁平向量空间并未显式编码情景结构或时间结构。结构增强方法构建图或记忆层，以改善多跳与跨会话检索（Edge et al., 2024）：HippoRAG 1&2（Gutierrez et al., 2024; Gutiérrez et al., 2025）组织知识，以支持联想检索和持续更新；Graphiti/Zep（Rasmussen et al., 2025）维护时序知识图谱及上下文组装流水线；Mem0（Chhikara et al., 2025）从对话中抽取记忆并整合为图结构，同时采用知识更新机制。

与图分层正交，MemGPT（Packer et al., 2023）实现了操作系统式的层级分页和虚拟上下文管理。A-Mem（Xu et al., 2025）在笔记之间执行类似卡片盒笔记法（Zettelkasten）的智能体式动态链接。MemoryBank（Zhong et al., 2024）引入基于时间衰减的整合与遗忘。Reflective Memory Management（Tan et al., 2025）进一步优化随时间推移应存储和检索哪些内容。总体而言，大多数现有系统依赖基于 LLM 的摘要或图构建来抽取关键信息，却忽略了时空上下文与情境要素对支持情景记忆的关键作用。这些系统很少为情景记忆与推理提出显式设计。相比之下，我们的方法显式表示锚定到时间线的事件要旨与时间事实，将其与核心情境维度相连接，并与工具增强推理相结合。

### 2.2 情景记忆与推理

近期基准越来越强调语言智能体的长时程交互：LoCoMo（Maharana et al., 2024）与 REALTALK（Lee et al., 2025b）分别在合成场景和真实场景中评估多会话对话记忆。LongMemEval（Wu et al., 2025）的对话理解任务要求信息抽取或多会话推理，但大多数任务都可视为单次检索问题，智能体只需找回正确片段（Hu et al., 2025b）。TimeQA（Chen et al., 2021）、MenatQA（Wei et al., 2023）、Complex-TR（Tan et al., 2024）和 Test of Time（Fatemi et al., 2025）等时序阅读理解基准，将时序推理解构为多种技能：事件排序、日期解析、计数、持续时间估计和时间线构建。

从方法上看，TISER（Bazaga et al., 2025）等时间线自反思方法只通过改进提示来提升时序推理；工具使用型智能体（Ge et al., 2025; Hu et al., 2025a）则可与非参数记忆进行程序化交互。我们的工作从两方面补充这些研究：（1）在混合图中，将情景编码为具备时间感知能力的要旨与事实；（2）在推断阶段，利用工具增强的检索与图探索完成情景回忆和推理。

## 3 方法

我们的 REMem 框架通过两个阶段，使语言智能体能够利用情景记忆进行推理：索引和智能体式推断。索引阶段构建存储各个情景之要旨与事实的记忆图；随后，智能体式推断阶段灵活利用精心设计的工具，对用户查询进行推理并从记忆图中检索。

**图 2：REMem 概览。** 索引阶段抽取事件要旨和具有时间作用域的事实（三元组），把话语转化为具备时间感知能力的记忆，并将其组织为混合图。智能体式推断阶段在该图上反复调用精心设计的工具，找出与推理最相关的要旨和事实。

> **图 2 中的示例。** 原始消息（2024 年 1 月 20 日 15:57）：Alice：“说到上周，让我想想。我上周日修好了篱笆，然后在 1 月 17 日从 Peter 那里买了 3 头牛。”要旨：“Alice 上周日（2024 年 1 月 14 日）修好了篱笆”；“Alice 于 1 月 17 日（2024 年 1 月 17 日）从 Peter 那里买了 3 头牛”。事实图包含短语节点 Alice、修篱笆、3 头牛，关系为“完成任务”“购买”，时间分别为 2024 年 1 月 14 日和 17 日。查询：“Alice 在买牛之前是哪天修好篱笆的？”智能体先调用 <code>semantic_retrieve("Alice buys cow")</code>，观察到“于 2024 年 1 月 17 日购买”；再调用带时间条件的 <code>semantic_retrieve("Alice fixes fence", start_time="17 Jan 2024", start_op="lt")</code>，观察到“Alice 于 2024 年 1 月 14 日完成修篱笆”；最后调用 <code>output_answer("14 Jan 2024")</code>，回答“2024 年 1 月 14 日”。

**表 1：精心设计的工具及其签名。** 检索工具与图探索工具都输出两组结果：要旨列表和事实列表。提示与演示见附录 D。

| 类型 | 名称 | 参数 |
|---|---|---|
| 检索 | <code>semantic_retrieve</code> | <code>query</code>、<code>start_time</code>、<code>end_time</code>、<code>start_operator</code>、<code>end_operator</code> |
| 检索 | <code>lexical_retrieve</code> | <code>query</code>、<code>start_time</code>、<code>end_time</code>、<code>start_operator</code>、<code>end_operator</code> |
| 图探索 | <code>find_gist_contexts</code> | <code>gist_id</code>、<code>start_time</code>、<code>end_time</code>、<code>start_operator</code>、<code>end_operator</code> |
| 图探索 | <code>find_entity_contexts</code> | <code>subject</code>、<code>object</code>、<code>predicate</code>、<code>start_time</code>、<code>end_time</code>、<code>start_operator</code>、<code>end_operator</code>、<code>limit</code>、<code>ordering</code>、<code>offset</code>、<code>aggregation</code> |
| 流程控制 | <code>output_answer</code> | <code>answer</code> |

### 3.1 索引

认知科学告诉我们，人类做决策时更多依赖要旨记忆而非逐字记忆（Reyna & Brainerd, 1995）；我们的语篇记忆围绕情境模型形成，而情境模型会整合超越表层细节的时间、空间、因果及其他情境维度（Johnson-Laird, 1983; Zwaan & Radvansky, 1998）。此外，近期 RAG 研究表明，结合概念层信息与上下文层信息的混合记忆结构，比单独使用任一层级更有效（Edge et al., 2024; Gutiérrez et al., 2025）。受这些见解启发，我们构建记忆图（图 2）时遵循全面性与灵活性原则，共同编码包含多种情境维度的要旨，以及带有时间上下文的事实。具体执行以下步骤。

1. **要旨抽取。** 针对每条事件陈述或每个聊天会话，我们生成一条或多条自然语言要旨陈述。在适用时，每条要旨都以该情景的时间戳（参照时间）为前缀，并将所有相对时间表达解析为绝对日期。由此，每个情景（例如一个聊天会话）得到一个要旨列表；其中每条要旨都是简洁句子，以单一原子事件描述捕捉该情景的关键细节，包括参与者、动作、对象、地点、意图和数量。
2. **事实抽取。** 我们进一步从每个情景的文本和抽取出的要旨列表中提取结构化事实。这些事实表示为（主语、谓词、宾语）三元组；每个字段都是无模式短语，主要捕捉“谁对谁做了什么”。我们还抽取时间上下文（日期/时间），并可为相应三元组附加 Wikidata 式限定符（Vrandecic & Krötzsch, 2014）——<code>point_in_time</code>、<code>start_time</code> 和 <code>end_time</code>——从而把每项事实锚定到时间线。随着时间推移，我们会保留已加入的要旨与事实，即使它们可能相互矛盾；这样便维持了一份可回溯查看的历史记录。
3. **图构建。** 利用上述输出，我们构建整合要旨节点与短语节点的记忆图。要旨节点充当上下文层的情景表示，每个节点都连接到从同一文本块抽取的短语节点。在概念层，每项事实的主语短语节点与宾语短语节点由边直接连接，以编码这些短语之间抽取出的关系。由此形成结合概念层与上下文层的混合记忆图。为进一步增强连通性，我们沿用 HippoRAG 2（Gutiérrez et al., 2025）的方法，在嵌入相似度超过阈值的要旨节点之间添加同义边。这一机制将语义相关的要旨聚为一组（例如，对相似事件的不同措辞），并以更高层的语义连接丰富图结构。

### 3.2 智能体式推断

常见 RAG 方法只是对相关文本执行一次匹配；与之不同，我们的推断阶段包含迭代检索，以处理复杂逻辑，还包含“心理时间旅行”，按时间条件筛选记忆条目。我们采用 ReAct 式智能体（Yao et al., 2023a; Gu et al., 2024），并为其配备三类精心设计的工具来访问混合图：（1）检索工具；（2）图探索工具；（3）流程控制工具。检索工具或图探索工具返回的结果同时包含要旨与事实，以提供全面视图。具体而言，如表 1 所示，智能体在混合记忆图上遵循三阶段协议。完整工具规格见附录 D。

1. **检索。** 智能体首先调用 <code>semantic_retrieve</code>（使用嵌入模型）或 <code>lexical_retrieve</code>（使用 BM25），从图中获取经过截断的种子节点及其上下文，例如候选实体 ID、时间窗口和粗粒度主题线索。智能体会把复杂问题分解为更简单的子查询，以指导下一步。
2. **图探索。** 智能体以第一阶段检索出的种子节点为基础，针对性调用 <code>find_gist_contexts</code>，获取情景层叙事和时间已锚定的证据；当查询明确针对已知图模式下的实体时，则使用 <code>find_entity_contexts</code>。<code>find_entity_contexts</code> 不仅支持指定主语、谓词或宾语，还能筛选满足特定时间条件的要旨或事实。
3. **流程控制。** 收集到充分证据并达到所需置信度后，智能体调用 <code>output_answer</code> 生成最终回答。该过程利用完整交互历史，将已探索的要旨与事实一并纳入最终推理。

## 4 实验设置

### 4.1 数据集

对于情景回忆任务，我们使用对话问答（QA）基准：LoCoMo（Maharana et al., 2024）是一个合成对话基准，REALTALK（Lee et al., 2025b）则采集自真实人类对话。两个基准都包含涉及时间方面和非时间情境方面的问题，我们使用其中的全部样本。

对于情景推理任务，我们使用阅读理解基准 Complex-TR（Tan et al., 2024）和 Test of Time（Fatemi et al., 2025）。我们从 Complex-TR 中随机抽取 1,000 个查询；对于 Test of Time，则使用其语义部分的全部样本，因为该部分明确要求使用记忆。

**表 2：抽样数据集的统计信息。**

| 数据集 | LoCoMo | REALTALK | Complex-TR | Test of Time |
|---|---:|---:|---:|---:|
| 查询数 | 1,986 | 728 | 1,000 | 2,800 |
| 文本块/消息数 | 5,882 | 8,944 | 1,095 | 124,919 |
| 图数量 | 10 | 10 | 1 | 560 |

### 4.2 对比方法

我们从以下类别中选择当前最先进的方法进行比较：

1. **RAG 设置下来自 MTEB 基准的强嵌入模型**（Muennighoff et al., 2023），包括 Qwen/Qwen3-Embedding-8B（Zhang et al., 2025）和 nvidia/NV-Embed-v2（Lee et al., 2025a）。
2. **结构增强型记忆方法**，包括 Mem0（Chhikara et al., 2025）、Graphiti（Rasmussen et al., 2025）和 HippoRAG 2（Gutiérrez et al., 2025）；其中前两种方法原本就在情景记忆任务上进行评估。我们使用它们的开源实现复现实验，而非使用专有版本。
3. **用于时序推理的提示方法 TISER**（Bazaga et al., 2025）：给定查询与检索出的上下文，我们在最终生成时使用其提示。该方法与任何记忆系统都相互正交。
4. **其他参考方法。** Oracle Message 接收预言机检索结果，只执行生成。Full-Context 使用整个语料库和查询进行生成。由于上下文窗口有限，Full-Context 不应被视为一种记忆方法；因此，我们只把它作为参考，而不将其视为对比记忆方法。

### 4.3 指标

对于 Test of Time（Fatemi et al., 2025），我们使用其唯一指标——精确匹配（EM）得分。对于其余基准，我们采用基于 token 的 F1、BLEU-1（Papineni et al., 2002）以及 LLM 评审得分作为 QA 任务指标；下文将 LLM 评审得分记为 LLM-J。F1 的计算沿用 HippoRAG 2（Gutiérrez et al., 2025），BLEU-1 使用 HuggingFace Evaluate（HuggingFace, 2025）的实现。我们采用与 Mem0（Chhikara et al., 2025）相同的 LLM 评估提示，其中同时考虑时间上下文和对话上下文。

**表 3：情景回忆任务上的性能（%）。** 每列最高值与第二高值在原文中分别以粗体和下划线标示。数值为均值，括号内为 95% bootstrap 置信区间的“上界增量/下界减量”；后续表格亦然。

| 方法 | LoCoMo F1 | LoCoMo BLEU-1 | LoCoMo LLM-J | REALTALK F1 | REALTALK BLEU-1 | REALTALK LLM-J |
|---|---:|---:|---:|---:|---:|---:|
| Oracle Message | 48.0 | 38.3 | 81.0 | - | - | - |
| Full-Context | 37.8 | 28.6 | 76.7 | 25.3 | 18.6 | 65.1 |
| Qwen3-Embed-8B | 35.3 (+1.7/-1.6) | 28.9 (+1.8/-1.5) | 64.2 (+2.4/-2.0) | 20.2 (+1.8/-1.6) | 14.9 (+1.6/-1.4) | 52.5 (+3.4/-3.6) |
| NV-Embed-v2 (7B) | 39.6 (+1.7/-1.4) | 31.0 (+1.7/-1.4) | 73.0 (+2.0/-1.8) | 23.8 (+1.9/-1.8) | 17.7 (+1.5/-1.5) | 59.5 (+3.3/-3.6) |
| Mem0 | 25.1 (+1.7/-1.5) | 18.0 (+1.4/-1.1) | 49.7 (+2.3/-2.2) | 9.8 (+1.5/-1.3) | 7.2 (+1.1/-1.0) | 14.3 (+2.7/-2.3) |
| Graphiti | 33.7 (+1.8/-1.8) | 28.9 (+1.9/-1.7) | 52.5 (+2.3/-2.3) | 15.1 (+1.8/-1.5) | 11.5 (+1.4/-1.2) | 35.3 (+3.7/-3.3) |
| HippoRAG 2 | 39.0 (+1.6/-1.6) | 30.8 (+1.5/-1.5) | 74.0 (+1.7/-2.1) | 21.9 (+1.6/-1.6) | 16.2 (+1.4/-1.3) | 55.8 (+3.4/-3.6) |
| REMem-I | 42.4 (+1.6/-1.6) | 32.7 (+1.5/-1.6) | 76.2 (+2.0/-1.9) | 25.6 (+1.8/-1.7) | 18.1 (+1.6/-1.4) | 63.7 (+3.6/-3.5) |
| REMem-S | 41.3 (+1.6/-1.5) | 31.5 (+1.6/-1.4) | 77.5 (+1.9/-1.6) | 26.2 (+1.8/-1.6) | 19.2 (+1.5/-1.3) | 65.3 (+3.6/-3.1) |

**表 4：情景推理任务上的性能（%）。**

| 方法 | Complex-TR F1 | Complex-TR BLEU-1 | Complex-TR LLM-J | Test of Time EM |
|---|---:|---:|---:|---:|
| Full-Context | 74.2 | 68.0 | 81.6 | 79.7 |
| Qwen3-Embed-8B | 77.1 (+2.3/-2.1) | 71.4 (+2.5/-2.4) | 80.9 (+2.5/-2.5) | 70.3 (+1.8/-1.7) |
| NV-Embed-v2 (7B) | 77.5 (+2.2/-2.1) | 71.9 (+2.3/-2.3) | 80.4 (+2.6/-2.5) | 68.9 (+1.7/-1.7) |
| NV-Embed-v2 w/ TISER | 88.1 (+1.7/-1.5) | 83.6 (+2.2/-1.8) | 88.3 (+1.9/-1.9) | 68.9 (+1.8/-1.8) |
| Mem0 | 43.1 (+2.8/-2.7) | 35.1 (+2.5/-2.4) | 41.0 (+3.0/-3.0) | - |
| Graphiti | 76.6 (+2.2/-2.3) | 71.4 (+2.4/-2.5) | 78.8 (+2.6/-2.6) | - |
| HippoRAG 2 | 78.2 (+2.3/-1.8) | 72.7 (+2.4/-2.4) | 81.5 (+3.4/-1.3) | 66.9 (+1.7/-1.7) |
| REMem-I | 83.3 (+1.8/-1.8) | 77.6 (+2.2/-2.1) | 89.6 (+2.0/-2.0) | 93.1 (+0.9/-1.1) |
| REMem-I w/ TISER | 90.6 (+1.2/-1.4) | 86.0 (+1.7/-1.7) | 92.0 (+1.6/-1.7) | 90.6 (+1.0/-1.2) |
| REMem-S | 78.5 (+2.0/-2.1) | 72.7 (+2.4/-2.4) | 82.6 (+2.3/-2.4) | 72.5 (+1.8/-1.8) |

> 表 3-4 中的方法分组依次为参考方法、大型嵌入模型、结构增强型记忆方法和本文方法；此处用行顺序代替原表的跨列分组标题。

### 4.4 实现细节

我们使用 GPT-4.1-mini-2025-04-14（OpenAI, 2025b）作为默认 LLM，使用 nvidia/NV-Embed-v2（Lee et al., 2025a）作为默认嵌入模型；无论是 REMem 还是对比方法，无论是抽取任务还是 QA 任务，均采用这一设置。对于使用嵌入模型的基线，我们检索 top-10 原始段落（对话中的消息）。对于 Mem0 和 Graphiti，我们检索其 top-10 已处理文本块。REMem 在每一步采用相同的检索范围，在 top-10 要旨和事实上操作。HippoRAG 2 返回的 top-3 会话用于最终生成。

我们研究了本方法的两种设置：

- **REMem-I（Iterative，迭代式）**：依照第 3.2 节的协议，在多步检索和推理过程中自主选择工具。
- **REMem-S（Single，单步式）**：只执行一次基于嵌入的检索，随后生成回答。

我们使用一个小型验证集，在 2 至 5 之间为各数据集选择 REMem-I 的最大智能体式推断步数；随后，情景回忆任务设为 3 步，情景推理任务设为 5 步。沿用 HippoRAG 2 的设置，同义边相似度阈值设为 0.8。

## 5 结果

### 5.1 情景回忆

情景回忆任务的结果见表 3。总体上，采自真实人类话语的 REALTALK 比合成的 LoCoMo 更具挑战。作为参考，Full-Context 表现良好，但推断成本相当可观：每个 LoCoMo 查询的平均输入 token 数为 26k；相比之下，REMem-I 和 REMem-S 在推断阶段分别消耗 9k 和 0.9k token。

使用大型嵌入模型的 RAG 方法构成强基线，NV-Embed-v2 尤其如此。结构增强型记忆方法总体表现欠佳，在 REALTALK 上尤为明显；该数据集中的人类话语更即兴、噪声更多且信息密度更低。Mem0 抽取许多陈述，但随后由其自身决策将其中大部分从记忆中丢弃，结果只能记住少量细节。Graphiti 构建以实体为中心的时序知识图谱，因而丢失了与事件各种情境维度相关的连贯上下文信息。HippoRAG 2 没有对时间维度或事件进行任何建模；由于它使用带嵌入检索的段落节点，其总体性能只与嵌入基线相当。

相比之下，REMem-I 和 REMem-S 显著优于现有方法，甚至接近预言机性能。LoCoMo 与 REALTALK 的更详细结果分别见附录 C.1 和 C.2；结果表明，REMem-I 与 REMem-S 在不同指标上各有优势。值得注意的是，LoCoMo 中跨会话问题仅占基准的 14.2%，而且大多数问题的支持消息来自单个聊天会话；这解释了为什么单步变体 REMem-S 在某些场景中表现更好。

### 5.2 情景推理

情景推理任务的结果见表 4。使用大型嵌入模型的 RAG 仍然是强基线。TISER 通过提示引导语言智能体执行时序推理；与我们直接明了的答案生成提示（图 6）相比，它在 Complex-TR 上展现出更强的推理能力。然而，它仍是一种固定提示，难以覆盖全部情景推理挑战，主要擅长先于/后于或最先/最后一类时间顺序问题。

结构增强型记忆方法主要在不同粒度（例如实体或摘要）上利用嵌入进行检索。然而，这种简单语义匹配不足以捕捉复杂推理所需的完整上下文信息，也无法支持必要的逻辑操作。REMem 在复杂推理任务上具有绝对优势。特别是，REMem-I 相较 REMem-S 优势明显（LLM-J +7.0，EM +20.6），并成为唯一 EM 得分超过 90% 的方法；这得益于多步检索和灵活的工具使用。此外，在这些富有挑战的情景推理任务上，REMem-I 相较 Full-Context 取得的提升（LLM-J +8.0，EM +13.4）显著大于它在情景回忆任务上的提升。Complex-TR 与 Test of Time 的更详细结果分别见附录 C.3 和 C.4。

## 6 讨论

### 6.1 消融研究

我们在 LoCoMo 和 Complex-TR 上开展了消融实验（表 5）。要旨和事实都很重要，但作用不同。移除要旨会造成最大幅度的性能下降，LoCoMo 上尤其如此：LLM-J 从 76.2 降至 48.9。这支持了我们的设计选择，即由要旨承载主要情境要素。移除事实导致的下降较小但始终存在，在 Complex-TR 上尤其明显：LLM-J 从 89.6 降至 87.2。这表明事实为多跳推理提供辅助，具体而言，它们提供了跨会话连接概念的具体锚点（附录 C.6）。

图结构与检索工具同样重要。消除同义边会降低两个数据集上的 F1 和 BLEU-1，说明对同义关系建模能够提高词汇层面的稳健性和召回率；与此同时，LLM-J 几乎不变，表明核心推理路径在很大程度上仍得到保留。最后，移除语义检索工具或词法检索工具都会损害性能。没有 <code>semantic_retrieve</code> 时，Complex-TR 上的 LLM-J 从 89.6 降至 88.1，在 LoCoMo 上也有所下降，说明语义检索对于找到概念相关的记忆至关重要。没有 <code>lexical_retrieve</code> 时，F1 和 BLEU-1 都会下降，Complex-TR 上的 LLM-J 也降至 87.5，说明词法检索可通过提高表层形式覆盖率来补充语义检索。

**表 5：LoCoMo 和 Complex-TR 上关于图结构与检索工具使用的消融研究。**

| 方法 | LoCoMo F1 | LoCoMo BLEU-1 | LoCoMo LLM-J | Complex-TR F1 | Complex-TR BLEU-1 | Complex-TR LLM-J |
|---|---:|---:|---:|---:|---:|---:|
| REMem-S | 41.3 | 31.5 | 77.5 | 78.5 | 72.7 | 82.6 |
| REMem-I | 42.4 | 32.7 | 76.2 | 83.3 | 77.6 | 89.6 |
| 不使用要旨 | 31.7 | 28.7 | 48.9 | 80.3 | 75.9 | 80.9 |
| 不使用事实 | 42.0 | 32.6 | 74.1 | 80.5 | 74.5 | 87.2 |
| 不使用同义边 | 37.6 | 28.7 | 76.4 | 81.6 | 75.6 | 89.2 |
| 不使用 <code>semantic_retrieve</code> | 41.7 | 33.3 | 72.8 | 82.4 | 76.4 | 88.1 |
| 不使用 <code>lexical_retrieve</code> | 40.6 | 31.2 | 76.8 | 81.7 | 75.8 | 87.5 |

### 6.2 拒答性能

在真实应用中，用户总会提出系统缺少足够上下文的问题。这些对抗性问题或无法回答的问题仍是有效的评估样本，因此我们将其纳入完整 LoCoMo 基准。LoCoMo 将这类问题的答案标注为“no information available”（没有可用信息）。因此，如果一种方法无法给出答案，或依照指示显式输出这一短语，我们便将其视为拒答。

表 6 展示的是拒答行为指标，而非 QA 指标。REMem 将最高精确率 73.3% 与有竞争力的召回率 56.8% 相结合，取得最高 F1（63.96%）。Graphiti 虽然召回率最高（83.6%），却给出 954 次拒答且精确率很低（38.9%）；相比之下，REMem 的精确率提高 34.4 个百分点，F1 提高 10.9 个百分点，而拒答次数约为前者的三分之一（344 对 954），说明不必要拒绝显著减少。与过度宽松、只拒答 90 次的 Mem0 相比，REMem 能正确标记更多无法回答的问题。总体而言，REMem 在对抗性问题的拒答行为上取得了更好的平衡。

**表 6：LoCoMo 上的拒答性能。** 在 1,986 个查询中，有 446 个无法回答。精确率是预测为无法回答的样本中确实无法回答者（即正确拒答为“no information available”）所占比例；召回率是真正无法回答的样本中被正确预测为无法回答者所占比例；F1 由这两个数值计算。

| 方法 | 拒答数 | 精确率（%） | 召回率（%） | F1（%） |
|---|---:|---:|---:|---:|
| Graphiti | 954 | 38.9 | 83.6 | 53.1 |
| Mem0 | 90 | 40.0 | 8.1 | 13.5 |
| REMem | 344 | 73.3 | 56.8 | 64.0 |

### 6.3 人工评估

为证明在情景记忆、尤其是推理任务中使用 LLM 作为评审的有效性，我们从 LoCoMo 中随机选择 100 个样本进行人工评估（表 7）。LLM 评审与人工评估均采用二元得分。LLM 评审在 93% 的样本上与人工评分一致，只有 7 个分歧。人工判定其中 5 个被 LLM 接受的回答有误，原因是列表不完整或存在时序推理错误；反过来，人工判定 2 个被 LLM 拒绝的回答正确，因为 LLM 未能识别有效的同义改述。尽管仍有一定局限，这些发现说明，与传统指标相比，使用 LLM 作为评审所产生的评估与人工判断最为一致。

**表 7：自动指标与人工评估的比较。** Mean 表示 100 个所选 LoCoMo 样本上的均值；相关系数均以人工评估为参照。

| 指标 | Mean | 与人工评估的 Pearson $r$ | 与人工评估的 Spearman $\rho$ |
|---|---:|---:|---:|
| 人工评估 | 0.710 | - | - |
| F1 | 0.410 | 0.551 | 0.603 |
| BLEU-1 | 0.284 | 0.417 | 0.531 |
| LLM 评审 | 0.740 | 0.827 | 0.827 |

### 6.4 错误分析

我们对 REMem 在 LoCoMo 和 Complex-TR 上的错误进行分析。在抽样的 100 个 LoCoMo 错误中，最常见的是选择或指代落地错误（46%）：REMem 找到了正确或相似的槽位，却赋予错误值，或在细节上错误理解了指称对象。例如，当被问及“Nate 最喜欢的电子游戏是什么？”时，REMem 回答“Catan”，但这只是 Nate 的一项兴趣，他真正最喜欢的是“Xenoblade Chronicles”。

时间或数值推理错误占 19%，包括相对日期或持续时间的计算错误。例如，对于“Nate 什么时候举办游戏聚会？”这一问题，模型回答“2022 年 6 月 18 至 19 日”，而金标准参考答案是“2022 年 6 月 3 日之后的那个周末”。另有 18% 的错误是：尽管证据已经检索到，模型仍然弃答，错误声称没有可用信息，而金标准答案实际就在证据中。

在抽样的 100 个 Complex-TR 错误中，最常见的失败模式是时间窗口错配（42%）：正确实体已被检索到，却没有与指定时间范围对齐。例如，当问题为“Ott-Heinrich Keller 在 1941 年 8 月至 1945 年 3 月期间曾为哪些雇主工作？”时，金标准答案同时包含“the Naval Academy at Mürwik”和“the University of Münster”，而我们的预测只列出前者。

约 21% 的错误源于多实体列表不完整或不一致，即检索结果遗漏某些项目或包含无关项目。约 18% 是偏移方向错误，例如混淆“之前”与“之后”，或跳到了更靠后的推理跳。对于“Karyn A. Temple 在 RIAA 之后曾为哪家雇主工作？”这一问题，金标准答案是“the U.S. Department of Justice”，模型却回答“the Copyright Office”。另有较小一部分错误（约 5%）是在金标准事实存在的情况下仍返回“no information available”。

### 6.5 对比分析：REMem 与 RAG

我们给出几个定性示例，将 REMem 与使用 NV-Embed-v2 的 RAG 系统进行比较。对于需要在品牌类别之间消歧、并协调带时间戳事件的问题，REMem 优于 NV-Embed-v2；而在直接明了的区间计算上，嵌入基线表现出色。

在表 8 的 Q1 中，REMem 抽取规范化要旨并据此推理，找出一条带有明确日期的摘要——“[2023 年 12 月 19 日] John 与一家知名户外装备公司达成了一项很棒的合作”——因此与金标准标签完全匹配。NV-Embed-v2 则选中了语义相似但错误的品牌垂类（饮料）。对于 Q2，REMem 正确串联事件并比较时间戳（Susie 大约于 2021 年 8 月被收养，而 Seraphim 是去年被收养的）；嵌入方法却固着于更早的一次提及，忽略了较新的收养事件。

相反，对于比较型问题 Q3，NV-Embed-v2 清楚检索出两个日期，并算出三个月的时间间隔；REMem 却在较长上下文中错误对齐事件，导致一个月的误差。总体而言，这些示例说明，具备时间感知能力的要旨抽取和智能体式推断通常能更稳健地应对干扰项与时间歧义；当答案只需对准确检索出的事实进行简单计算时，普通 RAG 流水线也可以很可靠。REMem 与 TISER 在 Complex-TR 上的另一项对比分析见附录 F.1。

**表 8：NV-Embed-v2 与 REMem 在 LoCoMo 上的比较。** T 或 F 表示 LLM 的判断（正确或错误）。

| 问题 | 金标准答案 | NV-Embed-v2 | REMem |
|---|---|---|---|
| Q1：John 在 12 月获得了哪种合作？ | 与一家户外装备公司的合作 | 与一家饮料公司的代言合作（F） | 与一家户外装备公司的代言合作（T） |
| Q2：Jolene 收养 Susie 和 Seraphim 中的哪只宠物更晚？ | Seraphim | Susie（F） | Seraphim（T） |
| Q3：Andrew 收养 Toby 和 Buddy 两件事之间相隔多少个月？ | 三个月 | 3 个月（T） | 1 个月（F） |

## 7 结论

对于语言智能体而言，情景回忆和情景推理等挑战远未解决。我们提出了 REMem——一个具备时间感知能力的情景记忆框架。所提出的混合记忆图以灵活的时间感知能力统一概念层信息和上下文层信息，智能体式检索器则实现了检索与推理的整合。在四个基准的情景回忆与情景推理任务上，REMem 始终表现更好；面对无法回答的查询，它拥有更好的拒答行为和更高的 token 效率。REMem 是迈向更可靠长时程语言智能体的一项可期进展。未来工作应考虑在更复杂环境中运行的语言智能体之长期记忆。与离线批量索引相比，以流式形式构建记忆也带来了工程挑战。

## 致谢

作者感谢俄亥俄州立大学 NLP 小组和 Intuit AI Research 的同事所提供的建设性讨论。本材料基于美国国家科学基金会资助编号 2443149 所支持的工作。本文表达的任何观点、发现、结论或建议均属于作者本人，不一定反映美国国家科学基金会的立场。本工作还得到 Alfred P. Sloan Foundation Fellowship 的支持。

## 可复现性声明

我们提供了复现本方法与实验所需的实现细节。本文使用的所有数据集均公开可用；数据集抽样见第 4.1 节，指标计算见第 4.3 节，REMem 及对比方法的实现细节见第 4.4 节。Mem0 和 Graphiti 的更多细节见附录 E。

## 参考文献

> 以下条目保留原文书目信息与题名，不翻译论文标题，以避免影响检索。

- Adrián Bazaga, Rexhina Blloshmi, Bill Byrne, and Adrià de Gispert. Learning to reason over time: Timeline self-reflection for improved temporal reasoning in language models. *CoRR*, abs/2504.05258, 2025. doi: 10.48550/ARXIV.2504.05258.
- Wenhu Chen, Xinyi Wang, and William Yang Wang. A dataset for answering time-sensitive questions. In *Proceedings of the Neural Information Processing Systems Track on Datasets and Benchmarks 1*, 2021.
- Prateek Chhikara, Dev Khant, Saket Aryan, Taranjeet Singh, and Deshraj Yadav. Mem0: Building production-ready AI agents with scalable long-term memory. arXiv:2504.19413, 2025.
- Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt, and Jonathan Larson. From local to global: A graph RAG approach to query-focused summarization. *CoRR*, abs/2404.16130, 2024.
- Bahare Fatemi, Mehran Kazemi, Anton Tsitsulin, Karishma Malkan, Jinyeong Yim, John Palowitch, Sungyong Seo, Jonathan Halcrow, and Bryan Perozzi. Test of Time: A benchmark for evaluating LLMs on temporal reasoning. In *ICLR 2025*, 2025.
- Yubin Ge, Salvatore Romeo, Jason Cai, Raphael Shu, Yassine Benajiba, Monica Sunkara, and Yi Zhang. TReMu: Towards neuro-symbolic temporal reasoning for LLM-agents with memory in multi-session dialogues. In *Findings of ACL 2025*, pp. 18974-18988, 2025.
- Yu Gu, Yiheng Shu, Hao Yu, Xiao Liu, Yuxiao Dong, Jie Tang, Jayanth Srinivasa, Hugo Latapie, and Yu Su. Middleware for LLMs: Tools are instrumental for language agents in complex environments. In *EMNLP 2024*, pp. 7646-7663, 2024.
- Bernal Jimenez Gutierrez, Yiheng Shu, Yu Gu, Michihiro Yasunaga, and Yu Su. HippoRAG: Neurobiologically inspired long-term memory for large language models. In *NeurIPS 2024*, 2024.
- Bernal Jiménez Gutiérrez, Yiheng Shu, Weijian Qi, Sizhe Zhou, and Yu Su. From RAG to memory: Non-parametric continual learning for large language models. *CoRR*, abs/2502.14802, 2025.
- Qianyi Hu, Xinhui Tu, Cong Guo, and Shunping Zhang. Time-aware ReAct agent for temporal knowledge graph question answering. In *Findings of NAACL 2025*, pp. 6013-6024, 2025a.
- Yuanzhe Hu, Yu Wang, and Julian J. McAuley. Evaluating memory in LLM agents via incremental multi-turn interactions. *CoRR*, abs/2507.05257, 2025b.
- HuggingFace. Evaluate. <https://huggingface.co/docs/evaluate>, 2025. Accessed: 2025-08-12.
- Philip Nicholas Johnson-Laird. *Mental models: Towards a cognitive science of language, inference, and consciousness*. Harvard University Press, 1983.
- Chankyu Lee, Rajarshi Roy, Mengyao Xu, Jonathan Raiman, Mohammad Shoeybi, Bryan Catanzaro, and Wei Ping. NV-Embed: Improved techniques for training LLMs as generalist embedding models. In *ICLR 2025*, 2025a.
- Dong-Ho Lee, Adyasha Maharana, Jay Pujara, Xiang Ren, and Francesco Barbieri. REALTALK: A 21-day real-world dataset for long-term conversation. *CoRR*, abs/2502.13270, 2025b.
- Adyasha Maharana, Dong-Ho Lee, Sergey Tulyakov, Mohit Bansal, Francesco Barbieri, and Yuwei Fang. Evaluating very long-term conversational memory of LLM agents. In *ACL 2024*, pp. 13851-13870, 2024.
- Niklas Muennighoff, Nouamane Tazi, Loïc Magne, and Nils Reimers. MTEB: Massive text embedding benchmark. In *EACL 2023*, pp. 2006-2029, 2023.
- OpenAI. What is memory? OpenAI Help Center, 2025a.
- OpenAI. GPT-4.1-mini. <https://platform.openai.com/docs/models/gpt-4.1-mini>, 2025b. Accessed: 2025-08-12.
- Charles Packer, Vivian Fang, Shishir G. Patil, Kevin Lin, Sarah Wooders, and Joseph E. Gonzalez. MemGPT: Towards LLMs as operating systems. *CoRR*, abs/2310.08560, 2023.
- Kishore Papineni, Salim Roukos, Todd Ward, and Wei jing Zhu. BLEU: A method for automatic evaluation of machine translation. pp. 311-318, 2002.
- Preston Rasmussen, Pavlo Paliychuk, Travis Beauvais, Jack Ryan, and Daniel Chalef. Zep: A temporal knowledge graph architecture for agent memory. *CoRR*, abs/2501.13956, 2025.
- Valerie F. Reyna and Charles J. Brainerd. Fuzzy-trace theory: An interim synthesis. *Learning and Individual Differences*, 7(1):1-75, 1995.
- Qingyu Tan, Hwee Tou Ng, and Lidong Bing. Towards robust temporal reasoning of large language models via a multi-hop QA dataset and pseudo-instruction tuning. In *Findings of ACL 2024*, pp. 6272-6286, 2024.
- Zhen Tan, Jun Yan, I-Hung Hsu, Rujun Han, Zifeng Wang, Long T. Le, Yiwen Song, Yanfei Chen, Hamid Palangi, George Lee, Anand Iyer, Tianlong Chen, Huan Liu, Chen-Yu Lee, and Tomas Pfister. In prospect and retrospect: Reflective memory management for long-term personalized dialogue agents. *CoRR*, abs/2503.08026, 2025.
- Denny Vrandecic and Markus Krötzsch. Wikidata: A free collaborative knowledgebase. *Communications of the ACM*, 57(10):78-85, 2014.
- Yifan Wei, Yisong Su, Huanhuan Ma, Xiaoyan Yu, Fangyu Lei, Yuanzhe Zhang, Jun Zhao, and Kang Liu. MenatQA: A new dataset for testing the temporal comprehension and reasoning abilities of large language models. In *Findings of EMNLP 2023*, pp. 1434-1447, 2023.
- Di Wu, Hongwei Wang, Wenhao Yu, Yuwei Zhang, Kai-Wei Chang, and Dong Yu. LongMemEval: Benchmarking chat assistants on long-term interactive memory. In *ICLR 2025*, 2025.
- Wujiang Xu, Zujie Liang, Kai Mei, Hang Gao, Juntao Tan, and Yongfeng Zhang. A-MEM: Agentic memory for LLM agents. *CoRR*, abs/2502.12110, 2025.
- Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik R. Narasimhan, and Yuan Cao. ReAct: Synergizing reasoning and acting in language models. In *ICLR 2023*, 2023a.
- Yunzhi Yao, Peng Wang, Bozhong Tian, Siyuan Cheng, Zhoubo Li, Shumin Deng, Huajun Chen, and Ningyu Zhang. Editing large language models: Problems, methods, and opportunities. In *EMNLP 2023*, pp. 10222-10240, 2023b.
- Yanzhao Zhang, Mingxin Li, Dingkun Long, Xin Zhang, Huan Lin, Baosong Yang, Pengjun Xie, An Yang, Dayiheng Liu, Junyang Lin, Fei Huang, and Jingren Zhou. Qwen3 Embedding: Advancing text embedding and reranking through foundation models. arXiv:2506.05176, 2025.
- Wanjun Zhong, Lianghong Guo, Qiqi Gao, He Ye, and Yanlin Wang. MemoryBank: Enhancing large language models with long-term memory. In *AAAI 2024*, pp. 19724-19731, 2024.
- Rolf A. Zwaan and Gabriel A. Radvansky. Situation models in language comprehension and memory. *Psychological Bulletin*, 123(2):162, 1998.

# 附录

## 附录目录

- A 大语言模型的使用
- B 形式化定义
- C 详细结果
  - C.1 LoCoMo
  - C.2 REALTALK
  - C.3 Complex-TR
  - C.4 Test of Time（语义）
  - C.5 语义记忆能力
  - C.6 消融研究
  - C.7 按时间类别划分的性能
- D 提示
- E 实现细节
- F 分析
  - F.1 对比分析：REMem 与 TISER
  - F.2 时间与空间效率
  - F.3 Token 用量
  - F.4 抽取示例
  - F.5 图属性

## 附录 A 大语言模型的使用

大语言模型未在本文的构思或写作中发挥重要作用。它们只用于语法检查以及对个别句子的轻微润色。

## 附录 B 形式化定义

令 $M=(V,E)$ 为一个有类型多重图，其中

$$
V=V_{\mathrm{gist}}\cup V_{\mathrm{phrase}},\qquad
E=E_{\mathrm{rel}}\cup E_{\mathrm{ctx}}\cup E_{\mathrm{syn}}
$$

因此，该图包含两种节点类型和三种边类型。

**节点。** 每个要旨节点 $g\in V_{\mathrm{gist}}$ 存储一段自然语言情景摘要 $\operatorname{text}(g)$ 及可选时间作用域 $\tau(g)$（时间点或区间）；它表示上下文层、便于人类阅读的事件描述，将参与者、动作、对象、地点、意图和数量等情境维度联合编码，并锚定到该时间作用域。每个短语节点 $p\in V_{\mathrm{phrase}}$ 存储一个短字符串 $\operatorname{name}(p)$，表示从事实三元组（主语、谓词、宾语）抽取出的概念层要素，通常指事件中的参与者、动作或对象。短语节点可以从底层事实继承 <code>point_in_time</code>、<code>start_time</code> 或 <code>end_time</code> 等时间限定符。

**边。** 每条关系边

$$
e=(p_s,r,p_o,\tau(e))\in E_{\mathrm{rel}}
$$

以谓词 $r$ 和有效区间 $\tau(e)$ 连接主语短语节点 $p_s$ 与宾语短语节点 $p_o$，编码事实层关系，例如“Alice $\xrightarrow{\text{购买}}$ 牛”，以及从相应限定符导出的时间作用域。

每条上下文边 $e=(g,p)\in E_{\mathrm{ctx}}$ 将要旨节点 $g$ 与短语节点 $p$ 相连；后者来自生成 $g$ 的同一源文本块。由此，抽象情景摘要（例如“Alice 于 1 月 17 日购买了牛”）便与其底层事实三元组（Alice，购买，牛）以及相关时间限定符（例如 <code>point_in_time = "Jan. 17th"</code>）相连。

每条同义边 $e=(g_i,g_j)\in E_{\mathrm{syn}}$ 连接文本嵌入相似度超过阈值的两个要旨节点，将语义等价或高度相关的情景聚成一组，并以联想链接丰富图结构。

**索引。** 我们为该图维护检索索引：在 $\operatorname{text}(g)$ 与 $\operatorname{name}(p)$ 上建立嵌入索引，并在要旨与事实的表层形式上建立词法（BM25）索引。

**算法。** 索引与智能体式推断的算法流程分别见算法 1 和算法 2。

**算法 1：** <code>Indexing(D; P_gist, P_fact, θ_syn)</code>

**要求：** 事件陈述 $D=\{d_i\}$（如有，则附时间戳）；要旨和事实抽取提示 $P_{\mathrm{gist}},P_{\mathrm{fact}}$；同义阈值 $\theta_{\mathrm{syn}}$。  
**确保：** 有类型记忆图 $M$ 和检索索引。

    1  初始化 V_gist, V_phrase, E_rel, E_ctx, E_syn ← ∅
    2  对每个 d ∈ D：
    3      G ← EXTRACT_GISTS(d; P_gist)
           ▷ LLM 返回一组带 text(g) 与 τ(g) 的要旨
    4      对每个 g ∈ G：
    5          将 g 加入 V_gist
    6      F ← EXTRACT_FACTS(d; P_fact)
           ▷ LLM 返回一组 (p_s, r, p_o, τ)
    7      P_d ← ∅
           ▷ 出现在 d 中的短语节点
    8      对每个 (p_s, r, p_o, τ) ∈ F：
    9          若为新节点，则将 p_s, p_o 加入 V_phrase
    10         将 e=(p_s, r, p_o, τ) 加入 E_rel
    11         将 p_s, p_o 加入 P_d
    12     对每个 g ∈ G：
    13         对每个 p ∈ P_d：
    14             将 (g,p) 加入 E_ctx
                   ▷ 将所有要旨绑定到同一 d 中的所有短语
    15 对 V_gist 中每对节点 (u,v)：
    16     若 SIM(u,v) ≥ θ_syn：
    17         将 (u,v) 加入 E_syn
    18 在 V 与 τ(·) 上构建嵌入/BM25 索引
    19 返回带索引的 M=(V,E)

**算法 2：** <code>AgenticInference(q, M, T_max)</code>

**要求：** 查询 $q$；带索引的混合记忆图 $M$；迭代上限 $T_{\max}$。

    1  E ← ∅
       ▷ 证据：累计的要旨与事实
    2  H ← []
       ▷ 交互历史（思考、动作、观察）
    3  k ← 0
       ▷ 迭代计数器
    4  当 k < T_max 时：
    5      t ← LLM_PLAN(q,E,H)
           ▷ 推理步骤
    6      a ← LLM_SELECTACTION(t)
    7      若 a.name = output_answer：
    8          返回 LLM_SYNTHESIZE(q,E,H)
               ▷ 根据已收集证据生成最终答案
    9      否则若 a.name = semantic_retrieve：
    10         (S,O) ← SEMANTIC_RETRIEVE(M,a.subquery)
               ▷ 嵌入搜索；返回种子节点 S 与观察 O
    11         E ← E ∪ O；H ← H ∥ [t,a,O]
    12     否则若 a.name = lexical_retrieve：
    13         (S,O) ← LEXICAL_RETRIEVE(M,a.subquery)
               ▷ BM25 搜索
    14         E ← E ∪ O；H ← H ∥ [t,a,O]
    15     否则若 a.name = find_gist_contexts：
    16         O ← FIND_GIST_CONTEXTS(M,a.seeds,a.temporal_constraints)
    17         E ← E ∪ O；H ← H ∥ [t,a,O]
    18     否则若 a.name = find_entity_contexts：
    19         O ← FIND_ENTITY_CONTEXTS(
                   M,a.subject,a.predicate,a.object,a.temporal_constraints)
               ▷ 按短语/谓词/时间约束筛选
    20         E ← E ∪ O；H ← H ∥ [t,a,O]
    21     k ← k+1
    22 返回 LLM_SYNTHESIZE(q,E,H)

> 原文算法 2 的普通文本提取结果重复了一次第 9 行；这里依据页面排版保留一次。

## 附录 C 详细结果

### C.1 LoCoMo

LoCoMo 上的性能见表 9 和表 10。对于 Oracle Message，虽然使用标注结果避开了检索难题，但 QA 仍非易事。总体而言，大型嵌入模型构成强基线，而结构增强型记忆方法性能较差。与 NV-Embed-v2 相比，HippoRAG 2 的 LLM-J 得分仅与之相当（+1.0）。相比之下，REMem-I 在 F1 和 J 得分上都优于 NV-Embed-v2（F1 +2.8，J +3.2）；在所有受评方法中，它的性能最接近预言机消息。REMem-S 又将 J 得分提高 1.3，在 LoCoMo 上取得最高 J 得分。

我们发现，在许多单会话设置中，REMem-S 优于 REMem-I；这说明大多数查询不需要复杂的多步检索，后者反而可能引入上下文噪声。这个基准有一项值得注意的偏差：多会话查询仅占全部查询的 14.2%。重要的是，与既有研究不同，我们还评估了对抗性设置下的性能；我们认为，这对评估系统的拒答能力和减少幻觉至关重要。模型拒答行为的更详细分析见第 6.2 节。

Full-Context 在多会话问题上取得最佳结果，这可能说明该基准对长上下文推理能力的要求有限。原文称 LoCoMo 的其他结果也见附录 C.1。默认情况下，在 LoCoMo 上使用嵌入模型时，我们采用 top-10 文本块。值得注意的是，我们还仅使用 top-3 文本块（消息）评估 NV-Embed-v2，发现它仍构成强基线。即使在这一受限设置下，它仍优于 Qwen3-Embedding-8B 和部分结构增强型记忆系统；唯一例外是配置为使用 top-3 会话的 HippoRAG 2。

**表 9：LoCoMo 上的性能（%，第 1/2 部分）。**

| 方法 | 平均 F1 | 平均 BLEU-1 | 平均 LLM-J | 单跳 F1 | 单跳 BLEU-1 | 单跳 LLM-J | 多跳 F1 | 多跳 BLEU-1 | 多跳 LLM-J |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Oracle Message | 48.0 | 38.3 | 81.0 | 36.5 | 28.2 | 85.7 | 41.6 | 26.9 | 84.8 |
| Full-Context | 37.8 | 28.6 | 76.7 | 29.1 | 20.6 | 84.4 | 34.2 | 18.8 | 75.2 |
| Qwen3-Embed-8B | 35.3 | 28.9 | 64.2 | 24.8 | 18.1 | 64.2 | 24.6 | 15.1 | 57.1 |
| NV-Embed-v2 (7B) | 39.6 | 31.0 | 73.0 | 30.9 | 22.6 | 77.9 | 29.4 | 17.8 | 64.9 |
| NV-Embed-v2（top-3 消息） | 39.0 | 32.1 | 67.2 | 26.5 | 20.2 | 68.2 | 21.1 | 13.6 | 51.4 |
| Mem0 | 25.1 | 18.0 | 49.7 | 10.2 | 6.3 | 32.7 | 29.9 | 15.9 | 52.1 |
| Graphiti | 33.7 | 28.9 | 52.5 | 6.4 | 3.4 | 30.8 | 25.1 | 16.1 | 53.6 |
| HippoRAG 2 | 39.0 | 30.8 | 74.0 | 25.8 | 18.0 | 72.0 | 30.7 | 19.5 | 72.3 |
| REMem-I | 42.4 | 32.7 | 76.2 | 37.9 | 25.2 | 81.3 | 32.6 | 18.7 | 70.2 |
| REMem-S | 41.3 | 31.5 | 77.5 | 35.4 | 22.6 | 86.0 | 31.6 | 18.9 | 69.5 |

> 样本量：平均 1,986；单跳 321；多跳 282。

**表 10：LoCoMo 上的性能（%，第 2/2 部分）。** 对抗类数值只能部分反映拒答性能；详见第 6.2 节。

| 方法 | 开放域 F1 | 开放域 BLEU-1 | 开放域 LLM-J | 时间类 F1 | 时间类 BLEU-1 | 时间类 LLM-J | 对抗类 F1 | 对抗类 BLEU-1 | 对抗类 LLM-J |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Oracle Message | 43.8 | 30.5 | 86.0 | 36.7 | 23.7 | 59.4 | 70.6 | 70.7 | 70.6 |
| Full-Context | 36.0 | 23.9 | 88.3 | 25.6 | 15.9 | 54.2 | 52.2 | 52.1 | 55.0 |
| Qwen3-Embed-8B | 27.9 | 19.6 | 67.3 | 25.1 | 15.3 | 45.8 | 65.9 | 66.0 | 66.8 |
| NV-Embed-v2 (7B) | 36.8 | 24.6 | 82.6 | 26.2 | 16.2 | 51.0 | 60.8 | 60.8 | 61.2 |
| NV-Embed-v2（top-3 消息） | 33.7 | 23.3 | 71.7 | 23.1 | 13.1 | 43.8 | 72.9 | 72.9 | 73.1 |
| Mem0 | 38.2 | 27.5 | 57.2 | 24.1 | 15.7 | 46.9 | 8.3 | 10.4 | 46.6 |
| Graphiti | 21.9 | 15.6 | 45.0 | 23.0 | 14.6 | 45.8 | 83.2 | 83.4 | 83.0 |
| HippoRAG 2 | 35.5 | 23.9 | 81.7 | 28.8 | 18.5 | 56.3 | 62.6 | 62.6 | 65.7 |
| REMem-I | 39.1 | 26.4 | 83.5 | 25.5 | 17.5 | 56.3 | 61.7 | 62.0 | 66.8 |
| REMem-S | 37.3 | 24.6 | 85.1 | 28.2 | 18.5 | 63.5 | 62.1 | 61.8 | 65.3 |

> 样本量：开放域 841；时间类 96；对抗类 446。

### C.2 REALTALK

所有方法的性能都持续偏低，证明 REALTALK 总体上比 LoCoMo 更具挑战。该数据集由真实人类交互构建，自然包含更多噪声与口语化表达。总体性能分布与 LoCoMo 上的观察相似：NV-Embed-v2 作为嵌入模型仍是强基线，结构增强型记忆方法则相形见绌。REMem 是唯一优于 Full-Context 的方法，在涉及时序推理的任务上尤其如此。

**表 11：REALTALK 上的性能（%）。**

| 方法 | 总体 F1 | 总体 BLEU-1 | 总体 LLM-J | 多跳 F1 | 多跳 BLEU-1 | 多跳 LLM-J | 常识 F1 | 常识 BLEU-1 | 常识 LLM-J | 时间类 F1 | 时间类 BLEU-1 | 时间类 LLM-J |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Full-Context | 25.3 | 18.6 | 65.1 | 27.8 | 21.7 | 60.1 | 19.5 | 15.1 | 55.6 | 24.9 | 16.8 | 73.0 |
| Qwen3-Embed-8B | 20.2 | 14.9 | 52.5 | 19.6 | 16.2 | 41.5 | 14.5 | 11.8 | 39.8 | 22.7 | 14.8 | 67.1 |
| NV-Embed-v2 (7B) | 23.8 | 17.7 | 59.5 | 24.5 | 19.4 | 51.2 | 16.6 | 13.0 | 48.2 | 25.6 | 17.7 | 71.2 |
| Mem0 | 9.8 | 7.2 | 14.3 | 14.0 | 9.6 | 20.9 | 12.0 | 9.0 | 22.2 | 5.1 | 4.3 | 5.3 |
| Graphiti | 15.1 | 11.5 | 35.3 | 19.6 | 15.5 | 39.5 | 16.7 | 13.2 | 44.4 | 10.5 | 7.1 | 28.2 |
| HippoRAG 2 | 21.9 | 16.2 | 55.8 | 26.1 | 20.9 | 51.8 | 18.2 | 13.7 | 54.6 | 19.2 | 12.6 | 59.9 |
| REMem-I | 25.6 | 18.1 | 63.7 | 26.6 | 21.6 | 55.8 | 16.1 | 13.0 | 45.4 | 27.9 | 16.5 | 77.4 |
| REMem-S | 26.2 | 19.2 | 65.3 | 28.6 | 23.1 | 58.8 | 18.2 | 14.7 | 53.7 | 26.7 | 17.1 | 75.2 |

> 样本量：总体 728；多跳 301；常识 108；时间类 319。

### C.3 Complex-TR

Complex-TR 上的性能见表 12。嵌入模型仍是强基线。在结构增强型记忆方法中，HippoRAG 2 略优于嵌入模型（J +1.1），这可能源于它与该数据集以实体为中心的特性相契合。与 REMem 相比，唯一具有竞争力的方法是使用 TISER 的 NV-Embed-v2；它采用多步时序推理提示，在时间到事件任务上甚至优于 REMem-I（J +1.4），但平均性能仍较低（J -1.3）。与事件到事件类型相比，时间到事件类型需要更直接的时间解析，因此更适合 TISER。

REMem 主要关注信息抽取和智能体式检索。对于推理部分，我们使用直接明了的提示（附录 D），利用上下文回答问题。在最终一步采用 TISER 作为推理提示后，使用 TISER 的 REMem 进一步提升，取得总体最高性能（相较 REMem，F1 +7.3，J +2.4）。此外，在这类复杂推理任务上，REMem-S 缺少多跳推理能力（J -6.0），证明采用智能体式检索执行自主推理是必要的。

**表 12：Complex-TR 上的性能（%）。**

| 方法 | 平均 F1 | 平均 BLEU-1 | 平均 LLM-J | 时间到事件 F1 | 时间到事件 BLEU-1 | 时间到事件 LLM-J | 事件到事件 F1 | 事件到事件 BLEU-1 | 事件到事件 LLM-J |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Full-Context | 74.2 | 68.0 | 81.6 | 73.0 | 65.6 | 80.3 | 75.6 | 70.8 | 83.2 |
| Qwen3-Embed-8B | 77.1 | 71.4 | 80.9 | 74.5 | 68.8 | 79.0 | 80.2 | 74.5 | 83.2 |
| NV-Embed-v2 (7B) | 77.5 | 71.9 | 80.4 | 77.0 | 70.8 | 79.2 | 78.1 | 73.2 | 81.8 |
| NV-Embed-v2 w/ TISER | 88.1 | 83.6 | 88.3 | 88.9 | 83.5 | 90.4 | 87.1 | 83.7 | 85.8 |
| Mem0 | 43.1 | 35.1 | 41.0 | 43.5 | 35.1 | 40.9 | 42.6 | 35.1 | 41.1 |
| Graphiti | 76.6 | 71.4 | 78.8 | 77.3 | 71.1 | 80.3 | 75.6 | 71.7 | 77.0 |
| HippoRAG 2 | 78.2 | 72.7 | 81.5 | 78.0 | 71.4 | 81.8 | 78.5 | 74.4 | 81.2 |
| REMem-I | 83.3 | 77.6 | 89.6 | 80.3 | 73.2 | 89.0 | 86.9 | 82.8 | 90.4 |
| REMem-I w/ TISER | 90.6 | 86.0 | 92.0 | 92.5 | 87.1 | 94.3 | 88.1 | 83.8 | 89.3 |
| REMem-S | 78.5 | 72.7 | 82.6 | 78.6 | 71.6 | 84.2 | 78.5 | 74.0 | 80.7 |

> 样本量：时间到事件 543；事件到事件 457。

### C.4 Test of Time（语义）

Test of Time 上的性能见表 13。这里，Full-Context 直接使用数据集的原始提示；每个问题平均包含数百项事实。由于 Test of Time 使用匿名实体标签和关系标签，除嵌入模型外，我们还使用 BM25 作为检索器。结果表明，即使是经过改进的 GPT-5-chat，以 Full-Context 方法运行时仍不及其前代、经济型的 GPT-4.1-mini（原文表述为“EM +4.8”）。

HippoRAG 2 高度依赖图遍历；由于不能理解复杂逻辑，其性能甚至不及嵌入模型。TISER 继续构成强基线，REMem 则是唯一超过 90% 的方法：相较带 TISER 的 Full-Context，EM 提升 8.2。除表 13 外，我们还观察到，REMem-I 搭配更强的 GPT-5-chat 后，可解决该基准八类挑战中的几乎全部（EM 99.0），进一步说明智能体式检索设计是合理的。

**表 13：Test of Time（语义）上的精确匹配率（%）。** 默认 LLM 为 GPT-4.1-mini。Full-Context 使用原数据集的完整提示；BM25 与 NV-Embed-v2 则通过检索 top-$k$ 事实缩短提示。BA：先后关系；ET：时间 $t$ 发生的事件；EW：事件发生于何时；FL：最先-最后；EA：另一事件发生时所发生的事件；NE：时间区间内的事件数；RD：关系持续时间；TL：时间线。每类包含 350 个样本。星号表示因全数据集评估成本高昂，使用 160 个样本的子集近似。

| 方法 | 平均 | BA | ET | EW | FL | EA | NE | RD | TL |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Full-Context (GPT-4.1-mini) | 79.7 | 72.0 | 86.9 | 98.0 | 81.1 | 83.7 | 76.3 | 93.4 | 46.3 |
| Full-Context w/ TISER | 84.9 | 88.3 | 91.4 | 97.7 | 89.1 | 90.6 | 64.0 | 94.6 | 63.4 |
| Full-Context (GPT-5-chat) | 84.5 | 80.0 | 90.0 | 99.4 | 84.9 | 91.1 | 90.0 | 82.3 | 58.0 |
| BM25 | 80.8 | 71.1 | 77.7 | 98.6 | 92.3 | 64.3 | 65.1 | 98.3 | 79.1 |
| Qwen-Embed-8B | 70.3 | 53.1 | 88.3 | 100.0 | 72.6 | 42.9 | 71.7 | 98.0 | 35.7 |
| NV-Embed-v2 (7B) | 68.9 | 43.1 | 86.6 | 100.0 | 74.6 | 48.0 | 75.7 | 96.9 | 26.6 |
| NV-Embed-v2 w/ TISER | 68.9 | 48.0 | 86.0 | 100.0 | 74.6 | 51.7 | 67.4 | 96.9 | 26.3 |
| HippoRAG 2* | 66.9 | 60.0 | 90.0 | 100.0 | 75.0 | 45.0 | 50.0 | 100.0 | 15.0 |
| REMem-I | 93.1 | 97.4 | 99.4 | 100.0 | 92.0 | 97.4 | 71.4 | 94.0 | 92.9 |
| REMem-I w/ TISER | 90.6 | 98.6 | 99.4 | 98.6 | 95.4 | 93.7 | 68.3 | 85.1 | 85.4 |
| REMem-S | 72.5 | 74.6 | 78.0 | 100.0 | 90.9 | 64.3 | 52.0 | 99.7 | 20.9 |

### C.5 语义记忆能力

尽管情景记忆是长期记忆的关键方面，也是本文的主要关注点，但语义记忆作为另一个方面同样不应被忽视。我们从 HippoRAG 2 使用的基准中选择 MuSiQue 与 2Wiki，评估这一能力。

从表 14 可以看到，Mem0 在所有指标上都远逊于 NV-Embed-v2 和 REMem，性能近乎崩溃。这揭示了 Mem0 记忆构建流程的局限：抽取出的事实有很大一部分没有加入记忆，而且它使用的固定对话抽取流水线与包含世界知识的段落结构并不匹配。Mem0 经常抽取不完整或去情境化的句子，甚至连主语都不明确。例如，它从标题为“Ferdinand Daučík”的段落中抽取出“曾执教包括 Barcelona、Atlético Bilbao、Atlético Madrid 和 Real Zaragoza 在内的 La Liga 俱乐部”，随后又没有把这个抽取句加入记忆或用其更新记忆。

> **译注：** 上句原文写作“From Table 7”，但对应内容实际位于表 14；译文按实际表号引用。

作为对比，我们的方法虽然主要关注情景记忆与推理，但在这一任务上仍取得与强嵌入模型相当的性能，展现出很强的可扩展性。

**表 14：使用语义记忆的 QA 任务性能（%）。**

| 方法 | 平均 F1 | 平均 BLEU-1 | 平均 LLM-J | MuSiQue F1 | MuSiQue BLEU-1 | MuSiQue LLM-J | 2Wiki F1 | 2Wiki BLEU-1 | 2Wiki LLM-J |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| NV-Embed-v2 | 38.2 | 30.5 | 57.5 | 37.5 | 31.9 | 56.3 | 38.8 | 29.1 | 58.6 |
| Mem0 | 6.9 | 5.1 | 8.0 | 7.4 | 5.7 | 9.2 | 6.3 | 4.5 | 6.8 |
| REMem | 38.2 | 32.1 | 55.3 | 37.9 | 33.5 | 53.2 | 38.6 | 30.8 | 57.4 |

> MuSiQue 与 2Wiki 各使用 1,000 个样本。

### C.6 消融研究

更详细的消融结果见表 15 和表 16。总体上，要旨提供主要上下文（LoCoMo 上 J -27.3%，Complex-TR 上 -8.7%），事实则提供不可或缺的补充支持。特别是在 LoCoMo 上，不使用事实的 REMem-I 在大多数任务上的表现与完整 REMem 相当，但在多跳问题上明显下降（J -7.1%）。这凸显出短语节点对跨会话连接概念并促进有效图探索的关键作用。

**表 15：LoCoMo 上的消融研究（%）。**

| 方法 | 平均 F1 | 平均 LLM-J | 单跳 F1 | 单跳 LLM-J | 多跳 F1 | 多跳 LLM-J | 开放域 F1 | 开放域 LLM-J | 时间类 F1 | 时间类 LLM-J | 对抗类 F1 | 对抗类 LLM-J |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| REMem-I | 42.4 | 76.2 | 37.9 | 81.3 | 32.6 | 70.2 | 39.1 | 83.5 | 25.5 | 56.3 | 61.7 | 66.8 |
| 不使用要旨 | 31.7 | 48.9 | 19.7 | 69.8 | 20.5 | 42.2 | 12.0 | 24.9 | 14.5 | 28.1 | 88.3 | 88.1 |
| 不使用事实 | 42.0 | 74.1 | 45.7 | 81.9 | 28.2 | 63.1 | 41.7 | 84.0 | 25.6 | 54.2 | 52.2 | 61.0 |
| REMem-S | 41.3 | 77.5 | 35.4 | 86.0 | 31.6 | 69.5 | 37.3 | 85.1 | 28.2 | 63.5 | 62.1 | 65.3 |

**表 16：Complex-TR 上的消融研究（%）。**

| 方法 | 平均 F1 | 平均 BLEU-1 | 平均 LLM-J | 时间到事件 F1 | 时间到事件 BLEU-1 | 时间到事件 LLM-J | 事件到事件 F1 | 事件到事件 BLEU-1 | 事件到事件 LLM-J |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| REMem-I | 83.3 | 77.6 | 89.6 | 80.3 | 73.2 | 89.0 | 86.9 | 82.8 | 90.4 |
| 不使用要旨 | 80.3 | 75.9 | 80.9 | 73.7 | 69.1 | 75.3 | 88.1 | 84.0 | 87.5 |
| 不使用事实 | 80.5 | 74.5 | 87.2 | 79.3 | 72.7 | 86.6 | 81.9 | 76.6 | 88.0 |
| REMem-S | 78.5 | 72.7 | 82.6 | 78.6 | 71.6 | 84.2 | 78.5 | 74.0 | 80.7 |

### C.7 按时间类别划分的性能

为进一步分析时间问题与非时间问题，我们按时间类别展示性能指标。表 17 给出了我们设定的若干时间类别；我们指示 GPT-4.1-mini 将每个查询归入其中一类。

**表 17：我们在 LoCoMo 上定义的时间类别。**

| 类别 | 数量 | 描述 | 查询示例 |
|---|---:|---|---|
| 存在性检查 | 14 | 给定特定时间范围，检查时间事实是否存在 | Andrew 在 2023 年 3 月期间养过宠物狗吗？ |
| 事件时间 | 298 | 确定事件发生的具体时间点或区间 | Melanie 什么时候画过日出？ |
| 事件属性 | 302 | 识别事件的属性或特征，例如地点、参与者等 | Caroline 在转型历程中面对过哪些变化？ |
| 顺序 | 22 | 理解事件之间的时间关系，例如先后、同时或重叠 | 公路旅行之后，Melanie 做了什么来放松？ |
| 持续时间 | 44 | 确定事件的持续时间，或两个事件之间的时间间隔 | Caroline 与目前这群朋友已经相识多久？ |
| 聚合 | 30 | 统计某事件在某时间范围内发生的次数，或统计事件的特定属性 | Melanie 在 2023 年去了多少次海滩？ |
| 其他 | 6 | 上述类别未覆盖的其他时序推理任务 | Caroline 会想很快搬回祖国吗？ |
| 非时间 | 1,270 | 不要求时序推理，但需要其他情境要素 | Oliver 曾经把骨头藏在哪里？ |

随后，我们依照上述类别在表 18 中报告 REMem-I 在 LoCoMo 上的性能；其中“时间类”是所有时间类别的平均值，“总体”是“时间类”与“非时间类”的平均值。这些结果说明，REMem-I 在非时间问题上的表现与时间问题相当。

**表 18：按时间类别划分的 LoCoMo 性能（%）。**

| 时间类别 | 样本数 | F1 | BLEU-1 | LLM-J |
|---|---:|---:|---:|---:|
| 总体 | 1,986 | 42.4 | 32.7 | 76.2 |
| 非时间 | 1,086 | 41.8 | 32.4 | 75.0 |
| 时间类 | 900 | 43.2 | 33.1 | 77.7 |
| 存在性检查 | 15 | 59.5 | 11.9 | 86.7 |
| 事件属性 | 450 | 45.9 | 37.9 | 76.7 |
| 事件时间 | 298 | 39.1 | 26.0 | 82.6 |
| 聚合 | 35 | 39.5 | 33.9 | 60.0 |
| 持续时间 | 50 | 35.2 | 26.8 | 72.0 |
| 顺序 | 49 | 48.4 | 45.0 | 71.4 |
| 其他 | 3 | 42.4 | 37.8 | 100.0 |

> **译注：** 表 17 与表 18 的类别计数在原文中不一致（例如“存在性检查”为 14/15，“事件属性”为 302/450，“持续时间”为 44/50，“顺序”为 22/49，“其他”为 6/3，“非时间”为 1,270/1,086）。本译文忠实保留两表原值，不擅自修正。

## 附录 D 提示

本节展示 REMem 的详细信息。用于要旨抽取和事实抽取的提示分别见图 3、图 4；用于工具选择的提示及相应工具描述见图 5-8。工具选择器会在每一步依据工具描述，从可用工具中选择一种。

### 图 3：要旨抽取提示

**要旨抽取指令与演示**

> 你是一名一丝不苟的信息抽取器。你的目标是将消息中的个人情景记忆提炼为结构化 JSON 格式。
>
> **核心任务**  
> 对于给定消息，识别其中每一项单独的事实、事件或主张。将每一项改写为简洁、自包含的英文句子。
>
> **输入格式**  
> 用户会提供当前时间与消息文本。你必须使用 <code>current_time</code> 解析所有相对时间表达（例如“昨天”“上周”）。
>
> **输出格式**
>
> - 输出必须是单个有效 JSON 对象。
> - JSON 对象必须只包含一个键：<code>"gists"</code>。
> - <code>"gists"</code> 的值是字符串列表。
> - 不要添加任何解释、注释或尾随逗号。
>
> **要旨规则**
>
> 1. **分解：** 将复杂句子分解为多条要旨。每条要旨应表示一个单独的原子事实或事件。
> 2. **时间戳前缀：** 每条要旨均以方括号中的消息时间戳开头，例如 <code>[20 January 2025, 2:28 pm]</code>。
> 3. **时间解析：** 在任何时间指称之后，以括号添加完整解析后的绝对日期或日期范围。时间点示例：<code>...last Thursday (16 January 2025).</code>；持续时间示例：<code>...last week (12 January 2025 to 18 January 2025).</code>
> 4. **完整性：** 捕捉每项事实的所有细节：参与者、动作、对象、数量、地点、意图等。
> 5. 为便于后续检索，尽可能多地推断上述维度中的合理细节，但不要虚构新信息。

**输入 1**

> 日期：2024 年 1 月 20 日 15:57  
> Alice：我上周一修好了篱笆，然后在 1 月 15 日从 Peter 那里买了 3 头牛。

**输出 1**

    {
      "gists": [
        "[20 January 2024, 3:57 pm] Alice fixed the fence last Monday (15 January 2024).",
        "[20 January 2024, 3:57 pm] Alice bought 3 cows from Peter on Jan 15th (15 January 2024)."
      ]
    }

**输入 2**

> 日期：2025 年 1 月 20 日 14:28  
> Bob：我上周四上午和导师见了面，两天后提交了提案。

**输出 2**

    {
      "gists": [
        "[20 January 2025, 2:28 pm] Bob met with his advisor last Thursday morning (16 January 2025).",
        "[20 January 2025, 2:28 pm] Bob submitted the proposal two days later (18 January 2025)."
      ]
    }

**图 3 图注：** 要旨抽取提示。指令与演示以不同颜色标示。

### 图 4：事实抽取提示

**事实抽取指令与演示**

> 从个人情景记忆消息中抽取结构化事实。返回一个 JSON 对象，其中唯一的键为 <code>facts</code>；其值为事实列表，每项事实包括：
>
> - <code>subject</code>（字符串）：执行/经历动作的实体；
> - <code>predicate</code>（字符串）：动作、关系或状态；
> - <code>object</code>（字符串）：动作所作用的实体/概念；
> - <code>qualifiers</code>（字典）：下列各属性优先采用 <code>%d %B %Y, %l:%M %p</code> 格式：
>   - <code>record_time</code>（字符串，必需）：消息创建时间；
>   - <code>point_in_time</code>（字符串，可选）：只用于指示事件发生的单个时间点；使用该字段时，忽略 <code>start_time</code> 和 <code>end_time</code>；
>   - <code>start_time</code>（字符串，可选）：事件开始时间，用于表示时间范围；
>   - <code>end_time</code>（字符串，可选）：事件结束时间，用于表示时间范围。
>
> 使用简短的“事件句柄”（例如 <code>fence fixing</code>），以便复用。通过共享实体连接事件。
>
> **规则**
>
> - 捕捉所有事实主张、数量、时间指称和关系；
> - 包含所有内容；宁可多收录；
> - 使用有文本支持的解释，而非假设，以免产生幻觉；
> - 若提供附加要旨，则加以利用；
> - 始终包含所给日期/时间对应的 <code>record_time</code>；
> - 只返回有效 JSON，不要添加额外键或注释。

**输入**

> 日期：2024 年 1 月 20 日 15:57  
> Alice：我上周日修好了篱笆，然后在 1 月 15 日从 Peter 那里买了 3 头牛。

**输出**

    {
      "facts": [
        {
          "subject": "Alice",
          "predicate": "completed task",
          "object": "fence fixing",
          "qualifiers": {
            "record_time": "20 Jan 2024, 3:57 pm",
            "point_in_time": "14 Jan 2024"
          }
        },
        {
          "subject": "Alice",
          "predicate": "purchased",
          "object": "3 cows",
          "qualifiers": {
            "record_time": "20 Jan 2024, 3:57 pm",
            "point_in_time": "15 Jan 2024"
          }
        },
        {
          "subject": "cow purchase",
          "predicate": "source",
          "object": "Peter",
          "qualifiers": {
            "record_time": "20 Jan 2024, 3:57 pm",
            "point_in_time": "15 Jan 2024"
          }
        }
      ]
    }

**图 4 图注：** 事实抽取提示。指令与演示以不同颜色标示。

### 图 5：工具选择提示

**工具选择：指令与演示**

> 你是一名智能体，正在探索一个大型知识图谱（KG）以回答查询。请根据当前上下文，选择最佳工具及其参数。
>
> **指令**
>
> - 理解先前步骤与当前步骤的上下文，并判断是需要继续探索 KG，还是输出答案。
> - 注意先前步骤的观察结果；它们会告诉你找到了什么、检索了多少信息。
> - 提前规划，从可用工具中选择当前步骤的唯一最佳工具，并给出参数。
>
> **工具使用策略**
>
> 1. **初始检索：** 先使用 <code>semantic_retrieve</code> 或 <code>lexical_retrieve</code> 理解数据模式并收集初步上下文，目标是识别 KG 的结构。对于复杂查询，将其分解为更简单的子查询，以指导后续步骤。
>    - 除非查询明确针对实体标识符或要求精确匹配，否则优先使用 <code>semantic_retrieve</code> 而非 <code>lexical_retrieve</code>。
>    - 注意：这些工具返回截断内容。初次扫描用于识别更详细搜索所需的关键信息（例如实体 ID、时间范围）。
> 2. **聚焦探索：** 随后使用 <code>find_gist_contexts</code> 或 <code>find_entity_contexts</code> 检索具体、详细的信息。
>    - 使用初始检索步骤中发现的关键信息，填写这些工具的参数（例如 <code>subject</code>、<code>object</code>、<code>limit</code>、<code>ordering</code>）。
>    - 优先使用 <code>find_gist_contexts</code> 而非 <code>find_entity_contexts</code>。只有当查询明确针对你已经见过的、具有定义良好 KG 模式的实体时，才使用后者；否则，不要尝试用 <code>find_entity_contexts</code> 精确匹配短语。
> 3. **答案生成：** 收集到信息并确信答案后，使用 <code>output_answer</code> 给出最终结果。
>
> **时序推理示例**
>
> - 查找 1950 年以后开始的事实：<code>start_time='1950', start_operator='gt'</code>
> - 查找 1960 年以前结束的事实：<code>end_time='1960', end_operator='lt'</code>
> - 查找在特定时间点 1955 年存在的事实：<code>start_time='1955', start_operator='le', end_time='1955', end_operator='ge'</code>
> - 查找仅存在于 $[1950,1960]$ 内的事实：<code>start_time='1950', start_operator='ge', end_time='1960', end_operator='le'</code>
> - 查找与 $[1950,1960]$ 重叠的事实：<code>start_time='1960', start_operator='le', end_time='1950', end_operator='ge'</code>（这是开始时间大于结束时间的特殊情形）
>
> **输出格式：** 以 JSON 对象回答，例如：

    {
      "reasoning": "简要说明此步骤为何选择该工具和这些参数",
      "function": "列表中的精确工具名",
      "parameters": {
        "name": "value"
      }
    }

对于查询“找出在 E95 担任 E57 的 R11 之后，紧接着担任 E57 的 R11 的实体”，各步骤如下。

1. 查找“E95 担任 E57 的 R11”这一事件的结束时间；它就是下一个“担任 E57 的 R11”事件的开始时间。调用 <code>lexical_retrieve</code>，查询 <code>E95 was the R11 of E57</code>。观察：BM25 词法匹配检索到 5 条要旨和 3 个三元组。截断后的顶部内容包括：（1）E95 于 2020 年 1 月 15 日获任 E57 的 R11；该职位涉及管理 E57 的技术运营与战略规划。（2）<code>(E95, was the R11 of, E57) {"start_time":"2020-01-15","end_time":"2022-03-20"}</code>。（3）2020 至 2022 年间，E57 的 R11 职位由 E95 担任；其间该组织的基础设施得到显著改善。
2. 查找时间 <code>t1</code> 之后下一个“担任 E57 的 R11”的事件。调用 <code>find_entity_contexts</code>，参数包括 <code>object="E57"</code>、<code>relation="was the R11 of"</code>、<code>start_time="t1"</code>。观察：找到与实体 E57 关联的 12 条要旨和 8 个三元组（按从 <code>t1</code> 至任意时间筛选），并展示 15 条要旨和 15 个三元组。截断后的顶部内容包括：（1）<code>(E0, was the R11 of, E57) {"start_time":"2022-03-21","end_time":"2023-06-15"}</code>。（2）E0 于 2022 年 3 月 21 日接替 E95 担任 E57 的 R11，恰在 E95 任期结束后；E0 为该职位带来新的视角和策略。（3）E95 的相应任期为 2020-01-15 至 2022-03-20。
3. 最接近 <code>t1</code> 且匹配“担任 E57 的 R11”的实体是 E0。调用 <code>output_answer</code>，回答 <code>E0</code>。

**图 5 图注：** 工具选择提示。指令与演示以不同颜色标示。

### 图 6-8：工具描述

**<code>output_answer</code>**

- 描述：当你认为已掌握信息时，细致分析检索信息和相应问题，回答原始查询并结束搜索过程。
- 必需参数：<code>answer</code>（字符串）。提供简洁、确定、不附加展开说明的回答；也可按原始查询要求使用整数或其他格式。

**<code>semantic_retrieve</code>**

- 描述：使用嵌入相似度，在 KG 中搜索语义最相关的要旨和事实（三元组），同时返回两类结果中的 top 项。用于需要语义理解与概念匹配的场景。
- 必需参数：<code>query</code>（搜索查询字符串）。
- 可选参数：<code>start_time</code>、<code>end_time</code>（按事实的时间限定符筛选的时间范围起点/终点）；<code>start_operator</code>、<code>end_operator</code>（相应时间比较运算符：<code>lt</code>、<code>le</code>、<code>ge</code>、<code>gt</code>、<code>eq</code>）。

**图 6 图注：** <code>output_answer</code> 与 <code>semantic_retrieve</code> 的工具描述。

**<code>lexical_retrieve</code>**

- 描述：依据 BM25 得分，在 KG 中搜索最相关的要旨与事实（三元组），同时返回两类结果中的 top 项。用于基于关键词的匹配或精确术语匹配，例如标识符。
- 参数与 <code>semantic_retrieve</code> 相同；<code>query</code> 必需，其余时间筛选参数可选。

**<code>find_gist_contexts</code>**

- 描述：针对特定要旨，经由同义关系探索相关要旨和相连事实（三元组），可附带时间筛选器。
- 必需参数：<code>gist_id</code>，即从上一步选择、要探索的要旨节点索引（不是内容），从 1 开始。
- 可选参数：<code>start_time</code>、<code>end_time</code>、<code>start_operator</code>、<code>end_operator</code>，含义同上。

**图 7 图注：** <code>lexical_retrieve</code> 与 <code>find_gist_contexts</code> 的工具描述。

**<code>find_entity_contexts</code>**

- 描述：查找符合给定条件的事实（三元组）。必须至少提供 <code>subject</code>、<code>object</code> 或 <code>predicate</code> 之一。
- <code>subject</code>：事实的主语，例如 <code>(E1, was born in, E2)</code> 中的 E1。
- <code>object</code>：事实的宾语，例如上述三元组中的 E2。
- <code>predicate</code>：按精确名称筛选特定关系，例如 <code>was born in</code>。
- <code>start_time</code>：按事实开始时间筛选，例如 <code>1952-01-01</code>；与 <code>start_operator</code> 配合控制比较。
- <code>end_time</code>：按事实结束时间筛选，例如 <code>1957-12-31</code>；与 <code>end_operator</code> 配合控制比较。
- <code>start_operator</code>：开始时间比较运算符，可选 <code>ge</code>（≥）、<code>gt</code>（>）、<code>le</code>（≤）、<code>lt</code>（<）、<code>eq</code>（=）；默认为 <code>ge</code>。
- <code>end_operator</code>：结束时间比较运算符，选项同上；默认为 <code>le</code>。
- <code>limit</code>（整数）：限制返回结果数。与 <code>ordering</code> 配合获取最先/最后项目；若希望得到更多上下文，可不设置。
- <code>ordering</code>：按时间排序。<code>asc</code> 按 <code>start_time</code> 升序（最早在先），<code>desc</code> 按 <code>end_time</code> 降序（最晚在先）。
- <code>offset</code>（整数）：跳过前 $N$ 条结果。与 <code>ordering</code> 和 <code>limit=1</code> 配合，可获得第 $N$ 个项目（例如“第二次”）。
- <code>aggregation</code>：执行聚合并返回单个数字。可选 <code>count</code>、<code>count_unique_subjects</code>、<code>count_unique_objects</code>。

> **译注：** 图 8 的文字说明要求至少提供 <code>subject</code>、<code>object</code> 或 <code>predicate</code> 之一，但其 JSON Schema 中的 <code>required</code> 列表为空；本译文同时保留这两项原始信息。

**图 8 图注：** <code>find_entity_contexts</code> 的工具描述。

## 附录 E 实现细节

我们复现的是 Mem0（Chhikara et al., 2025）和 Graphiti（Rasmussen et al., 2025）的开源版本，而非专有版本；我们尽可能对齐各项设置，包括骨干 LLM、嵌入模型和上下文规模。

我们观察到，Mem0 经常自主选择拒绝将输入文本加入记忆。因此，我们采用更细的粒度：以消息为单位添加记忆，鼓励系统将更多信息纳入记忆。对于 Graphiti，我们发现频繁调用 LLM 与嵌入模型会带来过高的时间和经济成本，因此以会话为单位添加记忆。随后，Graphiti 将会话内的信息索引为多项事实，供后续检索。因此，Mem0 与 Graphiti 实际都在事实/句子层而非会话层存储信息。

**表 19：运行时间与内存用量。**

| REMem-I | 索引时间（秒） | 每查询推断时间（秒） | 最大内存用量（GB） |
|---|---:|---:|---:|
| LoCoMo | 3,604.2 | 4.3 | 2.6 |
| Complex-TR | 378.7 | 12.6 | 1.3 |

## 附录 F 分析

### F.1 对比分析：REMem 与 TISER

我们在 Complex-TR 上将 REMem 与使用 TISER 的 NV-Embed-v2（Bazaga et al., 2025）进行比较，并给出若干示例。本段将后者简称为 TISER。

REMem 通常通过更全面地处理多跳时序推理、恢复所需实体的完整集合而优于 TISER。例如，当问题是“Nancy L. Ross 在 ASU 之前曾在哪里接受教育？”时，金标准答案是“BC Cancer Research Centre”。我们的方法给出“Virginia Tech”和“BC Cancer Research Centre”，评审接受其为正确；TISER 则只返回 Virginia Tech。

反过来，TISER 通常通过精确定位指定时间窗口内的目标而优于 REMem；我们的回答则可能列出过于冗长的清单，或漂移到错误的时间跳。例如，对于“Barack Obama 在 State Elementary School Menteng 01 之后曾在哪里接受教育？”这一查询，金标准答案为“Punahou School”。TISER 精确返回该答案，REMem 却列出了后来就读的大学。

这一问题很可能源于“before”与“after”含义的歧义：使用 REMem 提供的工具时，这两个词可能使系统无法确定不等式是否应包含等号，导致实际工具调用并非总能与问题意图一致。然而，往往可以从检索到的上下文中更可靠地推断“before”与“after”的精确关系，因为可以直接在一系列事件的时间作用域内判定它。

### F.2 时间与空间效率

在索引阶段，Graphiti 的效率比 Mem0 和 REMem 低两个数量级。这可能是因为，它处理每个情景时都要多轮调用生成式 LLM 和嵌入模型。在推断阶段，REMem 的单步运行时间与 Mem0、Graphiti 相当，而多步运行时间随步数线性增长。

我们还在表 19 中报告 REMem 的运行时间和内存用量；表中仅统计 REMem 本身的内存消耗。若在本地部署 LLM 或嵌入模型服务，其主内存和 GPU 内存另行统计。实验在一台服务器上开展，配置为双 AMD EPYC 7643 48 核 CPU（96 个硬件线程）、4 块 NVIDIA A100-SXM4-80GB GPU，以及 1 TB 系统内存。

每个查询的推断时间受许多因素影响，包括 worker 数和 LLM 服务响应时间。报告的时间不包含任何 LLM 或嵌入缓存，也未使用多线程。启用缓存或多线程后，实际吞吐量会更高。

### F.3 Token 用量

表 20 展示 REMem 在 LoCoMo 上的 token 用量。REMem-S 在推断阶段的 token 消耗与嵌入基线几乎相同；REMem-I 的 token 消耗则随迭代次数增加。

**表 20：LoCoMo 上的 token 用量（1,986 个查询）。** 使用 OpenAI <code>o200k_base</code> 编码；估计成本按 GPT-4.1-mini 标准定价计算。

| 方法 | 索引输入 | 索引输出 | 推断输入 | 推断输出 | 总计 | 估计成本（美元） |
|---|---:|---:|---:|---:|---:|---:|
| NV-Embed-v2 | 不适用 | 不适用 | 1.19M | 0.17M | 1.36M | \$0.75 |
| REMem-I | 0.90M | 0.72M | 18.10M | 0.79M | 20.51M | \$10.02 |
| REMem-S | 0.90M | 0.72M | 1.83M | 0.18M | 3.63M | \$2.53 |

### F.4 抽取示例

表 21 给出 MuSiQue 的一段文章，以及 Mem0 与 REMem 的抽取结果。相比之下，Mem0 的抽取缺少事实覆盖，可能妨碍后续的理解与 QA 任务。REMem 的抽取更全面，尤其是最后一项（第 14 项）事实：它甚至推断出 RFEF 2002-03 赛季的结束时间是 2003 年 6 月。Mem0 为所存记忆添加“user”前缀，假定工作场景始终是对话消息；但对于描述客观知识的长段落，这并无必要。

**表 21：一段 MuSiQue 文章及 Mem0、REMem 的抽取结果。**

**文章：Lionel Messi**

在巴塞罗那青训学院 La Masia 度过一年后，Messi 终于在 2002 年 2 月注册加入西班牙皇家足球协会（RFEF）。他此时参加所有赛事，并与队友结为朋友，其中包括 Cesc Fàbregas 和 Gerard Piqué。14 岁完成生长激素治疗后，Messi 成为“Baby Dream Team”不可或缺的一员；这是巴塞罗那有史以来最出色的青年队。在他的第一个完整赛季（2002-03），他为 Cadetes A 出场 30 次、攻入 36 球，成为最佳射手；该队史无前例地包揽联赛、西班牙杯和加泰罗尼亚杯三项冠军。

加泰罗尼亚杯决赛以 4-1 战胜 Espanyol，后来在俱乐部传说中被称为 <em>partido de la máscara</em>，即“面具决赛”。此前一周，Messi 在一场联赛中颧骨骨折；他获准在决赛首发，条件是佩戴塑料护具。面具很快妨碍了他的发挥，他将其摘下，并在被换下前 10 分钟内攻入两球。赛季结束时，他收到加盟 Arsenal 的邀请——这是他第一次收到外国俱乐部的邀请；Fàbregas 和 Piqué 不久便前往英格兰，他却选择留在巴塞罗那。

**Mem0 抽取**

> 用户提到了 Lionel Messi 早期足球生涯的细节，包括：他在巴塞罗那青训学院 La Masia 的经历；于 2002 年 2 月注册加入西班牙皇家足球协会；与队友 Cesc Fàbregas 和 Gerard Piqué 的友谊；14 岁时完成生长激素治疗；2002-03 赛季为 Cadetes A 出场 30 次、攻入 36 球并成为最佳射手；赢得联赛、西班牙杯和加泰罗尼亚杯三冠王；颧骨骨折后戴着塑料面具比赛并攻入两球；以及收到 Arsenal 的邀请却选择留在巴塞罗那。

**REMem 要旨**

1. Lionel Messi 在巴塞罗那青训学院 La Masia 度过一年后，于 2002 年 2 月注册加入西班牙皇家足球协会（RFEF）。
2. Messi 在参加所有赛事期间与队友 Cesc Fàbregas 和 Gerard Piqué 结为朋友。
3. 14 岁完成生长激素治疗后，Messi 成为巴塞罗那史上最出色青年队“Baby Dream Team”不可或缺的一员。
4. 在第一个完整赛季（2002-03），Messi 为 Cadetes A 出场 30 次、攻入 36 球，成为最佳射手。
5. Cadetes A 在 2002-03 赛季史无前例地包揽联赛、西班牙杯和加泰罗尼亚杯三项冠军。
6. 加泰罗尼亚杯决赛以 4-1 战胜 Espanyol，后来被称为 <em>partido de la máscara</em>，即“面具决赛”。
7. 在联赛中颧骨骨折一周后，Messi 获准在加泰罗尼亚杯决赛首发，条件是佩戴塑料护具。
8. Messi 在加泰罗尼亚杯决赛中摘下塑料护具，并在被换下前 10 分钟内攻入两球。
9. 2002-03 赛季结束时，Messi 收到加盟 Arsenal 的邀请，这是他第一次收到外国俱乐部的邀请。
10. Fàbregas 与 Piqué 不久便前往英格兰，Messi 则选择留在巴塞罗那。

**REMem 事实**

1. （Lionel Messi，注册加入，西班牙皇家足球协会（RFEF））[时间点：2002-02]
2. （Lionel Messi，结识并成为朋友，Cesc Fàbregas 和 Gerard Piqué）
3. （Lionel Messi，成为不可或缺的一员，巴塞罗那史上最出色青年队“Baby Dream Team”）[时间点：2002-03]
4. （Lionel Messi，成为最佳射手，代表 Cadetes A 出场 30 次攻入 36 球）[时间点：2002-03]
5. （Cadetes A，赢得，联赛、西班牙杯和加泰罗尼亚杯史无前例的三冠王）[时间点：2002-03]
6. （加泰罗尼亚杯决赛，战胜，Espanyol）[时间点：2002-03]
7. （加泰罗尼亚杯决赛，被称为，<em>partido de la máscara</em>，即“面具决赛”）
8. （Lionel Messi，遭受，联赛中的颧骨骨折）[时间点：2002-03]
9. （Lionel Messi，获准首发，在佩戴塑料护具的条件下参加加泰罗尼亚杯决赛）[时间点：2002-03]
10. （Lionel Messi，摘下，塑料护具）[时间点：2002-03]
11. （Lionel Messi，攻入，在加泰罗尼亚杯决赛中被换下前 10 分钟内的两球）[时间点：2002-03]
12. （Lionel Messi，收到加盟邀请，Arsenal——他的第一家外国俱乐部）[时间点：2003-06]
13. （Lionel Messi，选择留在，巴塞罗那）
14. （Cesc Fàbregas 和 Gerard Piqué，前往，英格兰）[时间点：2003-06]

### F.5 图属性

对于评估所用的四个基准，表 22 报告了所构建记忆图的规模，包括短语节点、要旨节点、边、三元组的数量及相应 token 数。统计表明，LoCoMo、REALTALK 和 Complex-TR 会产生大型、稠密连接的图，包含大量上下文边和同义边。Test of Time 的图则小得多，而且没有要旨层标注；这是因为其陈述使用匿名实体且已经高度形式化，所以我们只抽取事实层信息。

**表 22：四个评估基准上的图属性；每项为每张图的平均值。**

| 属性 | LoCoMo | REALTALK | Complex-TR | Test of Time |
|---|---:|---:|---:|---:|
| 短语节点数 | 777.5 | 974.0 | 1,066.0 | 16.0 |
| 要旨节点数 | 730.1 | 889.0 | 1,095.0 | - |
| 关系边数 | 736.2 | 891.0 | 1,062.0 | - |
| 上下文边数 | 25,172.4 | 56,332.0 | 2,190.0 | - |
| 同义边数 | 1,082.2 | 606.0 | 748.0 | - |
| 三元组数 | 763.9 | 917.0 | 1,095.0 | 275.4 |
| 输入 token 数 | 15,965.8 | 19,250.0 | 26,158.0 | 4,109.7 |
| 短语节点 token 数 | 5,105.8 | 8,350.0 | 5,536.0 | 32.1 |
| 要旨节点 token 数 | 21,743.8 | 30,106.0 | 26,514.0 | - |
| 短语节点度 | 34.0 | 59.7 | 4.1 | 9.7 |
| 要旨节点度 | 34.4 | 63.4 | 2.0 | - |

## Sources

- papers/agent-memory/REMem - Reasoning with Episodic Memory in Language Agents/REMem - Reasoning with Episodic Memory in Language Agents.pdf
