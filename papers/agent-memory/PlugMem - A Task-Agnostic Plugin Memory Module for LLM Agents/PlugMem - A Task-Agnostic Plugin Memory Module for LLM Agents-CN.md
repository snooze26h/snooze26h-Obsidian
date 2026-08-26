# PlugMem：面向 LLM 智能体的任务无关插件式记忆模块

Ke Yang$^{*1}$，Zixi Chen$^{*2}$，Xuan He$^{*1}$，Jize Jiang$^{*1}$  
Michel Galley$^3$，Chenglong Wang$^3$，Jianfeng Gao$^3$，Jiawei Han$^1$，ChengXiang Zhai$^1$

$^*$ 同等贡献  
$^1$ 伊利诺伊大学厄巴纳-香槟分校　$^2$ 清华大学　$^3$ 微软研究院  
{key4,xuanhe4,jizej2,hanj,czhai}@illinois.edu  
chenzixi23@mails.tsinghua.edu.cn  
{mgalley,chenwang,jfgao}@microsoft.com

arXiv:2603.03296v1 [cs.CL] 6 Feb 2026

> 译注：方法名、模型名与数据集名保留英文；术语首次出现时酌情附英文。公式、图表、脚注、引文及原文中的数值和符号均予以保留。参考文献保留原文书目信息，以便准确检索。

## 摘要

长期记忆对于在复杂环境中运行的大语言模型（Large Language Model，LLM）智能体至关重要；然而，现有记忆设计要么是任务特定的、不可迁移，要么虽与任务无关，却因任务相关性低，以及检索原始记忆造成上下文爆炸而效果较差。我们提出 PlugMem——一种任务无关的插件式记忆模块，无需进行任务特定的重新设计，即可挂接到任意 LLM 智能体上。鉴于决策相关信息集中存在于抽象知识而非原始经验之中，我们借鉴认知科学，将情景记忆结构化为一个紧凑、可扩展、以知识为中心的记忆图，显式表示命题性知识与处方性知识。借助这种表示，系统能够高效检索记忆并对任务相关知识进行推理，而不必处理冗长的原始轨迹。它也有别于 GraphRAG 等其他图方法：后者以实体或文本块为记忆访问与组织单元，而本文以知识为单元。我们在三个异构基准上使用完全相同的 PlugMem 进行评估，分别为长时程对话问答、多跳知识检索和 Web 智能体任务。结果表明，PlugMem 持续优于任务无关基线，并超越任务特定的记忆设计；在统一的信息论分析下，它还实现了最高的信息密度。代码和数据见 <https://github.com/TIMAN-group/PlugMem>。

**图 1：智能体记忆方法的效用-成本可视化。**[^1] PlugMem 在需要处理多种记忆类型的异构基准上保持不变地接受评估，以最低的智能体侧记忆成本实现了最高的记忆决策效用。横轴为归一化记忆 token 数，纵轴为归一化 PMI。图中方法包括 PlugMem（LME）、PlugMem（HPQA）、PlugMem（WA）、Vanilla Retrieval、A-Mem、LiCoMemory、RAPTOR、HippoRAG2 和 AWM；箭头方向表示“效用更高、成本更低”。

[^1]: 每个点对应一种至少在三个基准之一接受评估的记忆方法，即 LongMemEval（Wu et al., 2024）、HotpotQA（Yang et al., 2018）或 WebArena（Zhou et al., 2024）。各点绘制在归一化效用-成本空间中，以支持跨任务可视化，并容纳并非适用于全部基准的基线。两个坐标轴均采用最小-最大归一化，将数值线性缩放至固定范围（例如 $[0,1]$），同时保持方法之间的相对次序。曲线通过扫描记忆 token 预算得到。归一化仅用于可视化；使用原始 PMI 值和 token 数的详细基准特定分析见第 4 节与图 5。

## 1 引言

大语言模型（LLM）智能体越来越多地在需要长期记忆的环境中运行；在这类环境里，相关信息分散在漫长的交互历史中，必须加以复用以支持未来决策（Wang et al., 2024a; Park et al., 2023）。然而，若将过去交互作为原始上下文简单累积，很快便会导致记忆无限增长、计算成本高昂以及性能下降（Packer et al., 2024; Liu et al., 2023）。为解决这一问题，许多智能体架构依靠外部记忆模块来存储和检索过去经验。可是，现有大多数记忆设计都把记忆表示与检索紧密耦合到特定任务或基准，并依赖人工设计的启发式规则来决定存储什么、如何使用（Wang et al., 2024b; Gutiérrez et al., 2025）。这类任务特定的记忆模块即使在狭窄场景中有效，也无法泛化：针对长时程对话优化的记忆系统不能直接迁移到 Web 导航，反之亦然。这一局限使我们需要一种任务无关、即插即用的记忆模块：它无需任务特定的重新设计即可挂接到任意 LLM 智能体，既减少逐任务工程开销，又使同一个记忆模块能在多种智能体场景中复用。

设计这样一种通用插件式记忆模块并不容易。最直接的任务无关方案是基于检索的记忆：把所有过去经验存成原始文本块，再按相关性检索（也可像 RAG 一样辅以推理）。然而，由于**知识稀疏性**，这种范式在实践中往往失效。真正有助于决策的记忆通常是浓缩、抽象的知识，而原始记忆冗长、具有情景性，并且充斥低层信息。例如，为用户推荐食谱时，智能体真正需要的是用户的饮食偏好和限制——这些是从多次交互中蒸馏出的紧凑事实命题——而不是重读漫长的对话历史。同样，在一个从未见过的 Web 界面上购物时，智能体需要的是可泛化的程序性知识，即如何搜索、筛选和结账，而不是包含整页观察、夹杂数千无关 token 的原始轨迹。因此，把原始情景记忆当作可直接使用的知识，会施加不必要的上下文负担，并掩盖与决策最相关的信息。

以往工作尝试通过压缩或摘要记忆来弥合这一差距，但大多采用任务特定的方式。记忆模块通常被专门设计为在某个特定基准上表现良好，例如对话式长期记忆或 Web 智能体，并隐式假设只有一种占主导地位的记忆类型（Wang et al., 2024b; Gutiérrez et al., 2025）。将这些设计用于不同任务领域时，如果不做大量任务特定修改，通常无法迁移。这便提出一项关键挑战：如何设计一个通用记忆模块，使其既能同时支持多种记忆类型，又能适应不同智能体任务的多样需求；同时，如何以一种既反映决策效用和智能体侧成本、又允许跨任务比较的方式来评估它？

针对这一挑战的设计层面，我们把方法建立在认知科学对记忆组织的原则性解释之上。数十年的研究表明，人脑从根本上区分**情景记忆**（经验的详细记录）与**知识层记忆**；后者还可进一步分为语义记忆（knowing that，即“知道什么”；事实命题）和程序性记忆（knowing how，即“知道怎么做”；面向行动的处方）（Tulving, 1972; Squire, 2004）。情景记忆是抽象出命题性与处方性知识的来源，而后两种知识最直接地服务于推理和决策。这一视角表明，有效的智能体记忆不应只是检索过去经验，而应主动把原始情景记忆转化为结构化、知识密集的表示。

基于这些原则，我们提出一种新型插件式记忆模块 PlugMem。它执行从记忆到知识的抽象，并支持跨智能体任务统一管理多种关键记忆类型。如图 2 所示，我们的方法包括：（i）结构化模块，对异构原始记忆进行标准化，通过层级抽象提取命题性与处方性知识，并将其组织成记忆图；（ii）检索模块，选择任务相关子图；（iii）推理模块，进一步为基础智能体适配和压缩检索到的知识。传统知识图谱在实体和关系上运作，而我们的记忆图在知识单元（即命题与处方）上运作；这些知识单元构成记忆访问与操作的基本单位。PlugMem 可以看作一种面向记忆管理、以知识为中心的 GraphRAG（Edge et al., 2025）：图节点是知识，而非实体或文本块。

**图 2：PlugMem 组织原始记忆并输出精炼的记忆 token，以辅助基础智能体决策。** 流程依次为：输入异构非结构化记忆，经结构化模块形成标准化、知识密集的记忆；检索模块产生检索记忆；推理模块结合思考与反思，输出任务相关记忆，使记忆帮助基础智能体执行任务。

作为通用记忆模块设计的补充，我们提出一种新的效用-成本分析框架，既刻画性能提升，也刻画记忆效率，从而能够在不同任务间公平比较不同记忆设计。具体而言，我们度量记忆的**信息密度**，将其定义为记忆向基础智能体提供的、每个记忆 token 所对应的决策相关信息增益。我们实现了 PlugMem，并在三个异构且具有挑战性的基准上评估它：长时程对话问答（Wu et al., 2024）、基于 Wikipedia 的多跳知识检索（Yang et al., 2018），以及 Web 智能体任务（Zhou et al., 2024）。我们在所有场景中使用同一套记忆模块实现，证明它相较朴素的任务无关基线和任务特定记忆模块均能带来稳定的性能增益，同时产生更低的智能体侧记忆成本，如图 1 所示。消融研究进一步阐明了各组件的作用。由于智能体记忆最终服务于决策，任务性能主要取决于检索能否在决策时调取最有用的记忆。结构化模块把异构经验组织成以知识为中心的记忆单元，以增量方式强化检索，由此带来额外性能提升。与此同时，推理模块通过任务自适应浓缩，令记忆 token 用量减少一至两个数量级，显著提高效率。附录 A 给出了示例性的基准案例。

概括而言，本文有四项贡献：

- **设计原则：** 为 LLM 智能体中的任务无关记忆提出由认知科学启发的原则。
- **评估框架：** 提出一种关于记忆效用与效率的信息论度量。
- **通用记忆模块：** 提出一种可应用于异构智能体基准的插件式记忆系统。
- **可复现性：** 发布代码与实验结果。

## 2 相关工作

**基于认知科学的智能体记忆。** 认知科学把长期记忆描述为持久存储，并区分情景、语义和程序性记忆（Atkinson & Shiffrin, 1968; Tulving, 1972; Squire, 2004）。在智能体环境中，交互天然具有情景性，并可从中导出可复用的抽象。例如，用户偏好等事实信息可以被蒸馏为语义记忆（Li et al., 2025），动作策略则可以抽象为程序性记忆（Wang et al., 2024b）。这些抽象与更一般的知识概念一致：命题性知识是事实陈述，处方性知识则是一般化的目标导向程序（Diebolt & Perrin, 2013）。

在这些洞见基础上，近期系统引入不同记忆类型，以支持长上下文理解（Lee et al., 2024）、长期交互（Li et al., 2025）、结构化记忆组织（Anokhin et al., 2024; Rasmussen et al., 2025）以及经验驱动的策略精炼（Zhao et al., 2024; Wang et al., 2024b; Fang et al., 2025）。然而，大多数方法都针对特定任务，且不能联合支持多种记忆类型。相比之下，PlugMem 对异构情景经验进行标准化，把抽取的知识组织成命题-处方结构，从而实现任务无关的复用。

**表 1：代表性智能体记忆系统比较。** M2K 表示系统是否将工作记忆转化为可跨任务泛化的、可复用的抽象知识。KaU 表示系统是否采用知识作为记忆单元。Mech. 表示记忆模块的设计，包括 Str.（结构化记忆表示）、Ret.（精炼检索策略）和 Rea.（检索后推理或压缩）。Mem. 表示系统支持的记忆类型，包括情景记忆（E）、语义记忆（S）和程序性记忆（P）。Agn. 表示记忆设计是否与任务无关。

| 方法 | M2K | KaU | Str. | Ret. | Rea. | E | S | P | Agn. |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Vanilla Retrieval | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✗ | ✗ | ✓ |
| Vanilla RAG | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✓ |
| GraphRAG | ✓ | ✗ | ✓ | ✓ | ✓ | ✗ | ✓ | ✗ | ✓ |
| A-Mem | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ | ✓ | ✗ | ✓ |
| Zep | ✗ | ✗ | ✓ | ✓ | ✗ | ✓ | ✓ | ✗ | ✗ |
| MemoryOS | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ | ✓ | ✗ | ✗ |
| HippoRAG2 | ✓ | ✗ | ✓ | ✓ | ✓ | ✗ | ✓ | ✗ | ✗ |
| AWM | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ |
| ReasoningBank | ✓ | ✓ | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ | ✗ |
| **PlugMem** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

**记忆模块设计。** 许多智能体记忆系统采用基于检索的外部记忆，在推理时存储并检索过去交互。早期任务无关方法依赖对非结构化情景记忆进行扁平检索，例如 Vanilla Retrieval 和 RAG（Lewis et al., 2021; Wang et al., 2024a），因而造成冗余且局限于特定情景的记忆复用。后续工作在情景记忆之上引入结构，包括采用层级检索和图检索（例如 GraphRAG）来支持多跳推理（Edge et al., 2025; Sarthi et al., 2024; Gutiérrez et al., 2025; Wang & Han, 2025）。这些方法虽提高了访问效率，却大体仍将情景轨迹保留为主要记忆单元。与此同时，任务特定系统会显式地把经验转换为更高层表示，例如时序知识图谱或工作流记忆（Gutiérrez et al., 2025; Wang et al., 2024b; Ouyang et al., 2025）。

尽管这些方法在固定任务内有效，其抽象却与任务假设紧密耦合，因而限制了可迁移性。

总体而言，如表 1 所示，先前工作凸显了两类方向之间的区别：改进对情景记忆的检索，以及将经验转化为可复用知识；只有后者支持跨任务泛化。PlugMem 采纳这一原则，围绕命题性与处方性知识单元组织记忆，从而更有效地检索决策相关信息，同时把情景轨迹保留为可验证证据。

关于相关工作的更详细讨论，包括上述各节的完整版本及有关记忆基准的补充讨论，见附录 B。

## 3 方法

我们提出 PlugMem——一种支持 LLM 智能体长期决策的任务无关插件式记忆模块。PlugMem 不把过去交互当作扁平的情景文本，而是将经验结构化为知识层表示；这种表示更加紧凑、更具泛化性，也与下游检索和推理直接相关。

如图 2 所示，PlugMem 包含三个核心组件：结构化模块，将异构情景记忆标准化，并归纳出命题性和处方性知识；检索模块，通过抽象感知检索从结构化记忆图中选择相关知识；推理模块，进一步把检索到的知识适配成基础智能体可以执行的指导。下文将详细介绍各个组件。

### 3.1 结构化模块

如图 3 所示，结构化模块是 PlugMem 的基础。它把对话轮次、文档片段或情景轨迹等原始情景经验，转化为可复用、紧凑且与智能体决策相契合的知识表示。具体而言，我们根据不同信息在推理和动作选择中扮演的角色来组织记忆。

我们的设计遵循认知科学人类记忆理论所启发的三项原则。第一，情景记忆记录具体交互轨迹，主要充当可验证证据，而不是可直接执行的知识。第二，决策相关信息最适合在知识层表示：语义记忆编码事实命题（“knowing that”），程序性记忆编码可复用策略（“knowing how”）。第三，有效的长期记忆需要把知识抽象与任务特定的执行细节分离，使记忆能够跨异构环境泛化。这些原则意味着，应以符合不同记忆类型自身属性的结构单元和组织逻辑来表示它们。表 2 总结了情景、语义和程序性记忆如何映射至相应的图单元与结构化机制，反映它们在抽象、检索和验证中的功能角色。

基于这一设计，结构化模块分两个阶段实现记忆抽象：（i）把异构交互轨迹标准化为统一的情景表示；（ii）归纳出可独立索引、跨任务复用的命题性和处方性知识。具体如下。

#### 3.1.1 标准化

情景记忆是产生语义记忆和程序性记忆的基本基底（Tulving, 1972）。对智能体而言，情景记忆来自异构来源，包括用户-智能体交互（Wu et al., 2024）、事实文档（Yang et al., 2018）以及复杂环境中的动作轨迹（Zhou et al., 2024）。这种异构性要求一种统一、任务无关的表示，以支持下游知识归纳。

**情景形式化。** 我们把一条原始交互轨迹表示为观察-动作对序列：

$$
\tau=[(o_t,a_t)]_{t=1}^{T}。
$$

尽管情景记忆已广泛用于智能体系统，其内部结构往往仍被视作非结构化文本。与此不同，我们把每次交互映射为结构化元组，在步骤层面显式地形式化情景记忆。具体而言，每一对 $(o_t,a_t)$ 均被标准化为：

$$
e_t=(o_t,s_t,a_t,r_t,g_t)，
$$

其中，$s_t$ 表示时刻 $t$ 的智能体状态，$g_t$ 表示与执行 $a_t$ 相关的子目标，$r_t$ 表示动作相对于 $g_t$ 的奖励。

状态 $s_t$ 通过基于 LLM 的信息抽取，由 $(s_{t-1},a_{t-1},o_t)$ 推导得到。$g_t$ 和 $r_t$ 均由 LLM 在任务指令和局部交互上下文条件下进行标注。汇总所有标准化步骤，得到情景记忆序列：

$$
M_{\mathrm{epi}}=[e_t]_{t=1}^{T}。
$$

情景标准化的实现细节（包括提示模板）见附录 C.1。

**图 3：PlugMem 的结构化模块把异构记忆转化为形式化、知识密集的记忆图。** 第一步“标准化”：将任意原始记忆（例如“用户：我刚买了一套 Revell 模型。助手：听起来不错！”、“第 3 段：Yesterday Once More 是专辑……”，以及“观察：[购物网站]；动作：点击‘Contact Us’”）转为逐步的观察 $(o)$-动作 $(a)$ 对；标注子目标 $(g)$、状态 $(s)$ 和奖励 $(r)$；形成标准化的 $(g,s,a,r,s')$ 元组。第二步“提取知识”：提取命题与概念，例如“用户于 6 月 5 日购买了一套 Revell 模型”（概念：Revell），以及“Carpenter 发行了 Yesterday Once More”（概念：专辑）；还会依据子目标相似度切分工作流，并提取处方和意图，例如“前往‘Contact Us’”“起草邮件”及相应的“步骤 1……步骤 2……”。

**表 2：记忆属性到图架构的映射。** 我们分析长期记忆的属性，以推导记忆图设计。主要依据是使记忆结构与知识被获取、存储和推理时的粒度保持一致；在这里，它体现为确保核心节点表示完整、自包含且可验证的知识块（命题/处方），从而提高下游图操作的效率与保真度。

| 记忆类型 | 性质与描述 | 知识密集单元 | 子图结构化逻辑 |
|---|---|---|---|
| 语义 | **陈述性且静态。** 与上下文无关的概念、事实和世界知识。 | **命题（事实块）。** 传达可验证真相的完整陈述。 | **以概念为中心的语义记忆结构化。** 逻辑：概念充当轻量索引，指向承载大量内容的命题。结构：概念 $\xleftarrow{\mathrm{mentions}}$ 命题。 |
| 程序性 | **执行性且以目标为导向。** 用于解决问题的动态“如何做”知识。 | **处方（工作流块）。** 执行复杂任务的完整动作序列。 | **以意图为中心的程序性记忆结构化。** 逻辑：意图（用户目标）充当查找整体解决方案块的键。结构：意图 $\xleftarrow{\mathrm{solves}}$ 处方。 |
| 情景 | **自传性且线性。** 过去交互和观察的原始记录，体量庞大。 | **源轨迹（事件窗口）。** 用于锚定和验证的轨迹片段。 | **情景记忆作为锚点。** 逻辑：情景记忆充当验证抽象知识图的“ground truth”层。结构：知识 $\xleftarrow{\mathrm{proves}}$ 源。 |

#### 3.1.2 提取知识

本节重点介绍知识抽取与组织过程的设计。模型选择、参数设置和提示配置等实现细节留待附录 C.2 说明。

给定标准化情景记忆 $M_{\mathrm{epi}}$，我们归纳两种互补的长期记忆：语义记忆和程序性记忆。二者都从情景经验中抽取，并组织成带有来源信息的结构化记忆图。

**语义记忆。** 语义记忆模块从情景记忆中抽取并存储事实知识，以支持后续检索。给定情景单元 $e_t$，模块使用 LLM 抽取一组原子命题，用以描述交互所蕴含的显著事实。每条命题都附带一组相关概念，作为用于索引的语义标签。例如，一条命题可以是：“Tam Sventon，在瑞典语中名为 Ture Sventon，是一位以斯德哥尔摩为基地的虚构私家侦探。”对应的概念集合为 {Tam Sventon, fictional private detective, Stockholm}。为保证抽取质量，我们在 LLM 抽取过程中施加多项约束，包括共指消解、命题去重和长度控制。

抽取出的命题和概念存储在语义图 $G^S$ 中。每条命题和每个概念均被实例化为一个节点，并缓存其稠密嵌入。系统构建两类边：（i）成员边，把命题连接至相关概念；（ii）来源边，把命题连接至情景图 $G^E$ 中对应的源情景单元。借助这一设计，检索到的语义知识可以追溯至其原始经验。

**程序性记忆。** 程序性记忆模块从情景轨迹中抽取可复用的动作策略，以支持未来决策。给定情景序列 $M_{\mathrm{epi}}$，模块首先检测相邻子目标 $g_{t-1}$ 和 $g_t$ 的相似度何时低于预定义阈值，并以此确定边界，把轨迹切分成连贯的子轨迹。对于每个轨迹片段，模块使用 LLM 归纳一个紧凑的（意图，处方）对。意图表示该片段中追求的目标；处方则给出一种与环境无关的动作工作流，捕捉成功执行所需的关键步骤和因果模式。例如，一条处方可以是：“要确定某件商品的最低价格，使用搜索栏搜索该商品，按价格对结果排序，并核实不同变体中的最低值。”为了支持质量感知的复用，每条归纳出的处方都会被赋予一个标量回报分数。该分数由基于 LLM 的评估器给出，评估意图是否实现，以及处方执行得如何。

抽取出的意图与处方存储在程序性记忆图 $G^P$ 中。每个意图和每条处方均被实例化为一个节点，并缓存其稠密嵌入。$G^P$ 中的边编码两种关系。第一，层级边将每个高层意图节点连接至相关的低层处方节点。第二，来源边把处方节点连接至情景图 $G^E$ 中对应的源情景单元，使程序性知识可以追溯至具体交互经验。

**图 4：PlugMem 以知识为中心的记忆图设计及其支持的标准图操作。** 记忆图包含语义子图、程序性子图和情景层：源节点“观察：价格为 \$0.99；动作：stop [最低价格：\$0.99]”支撑命题节点“杯子的最低价格是 \$0.99”，后者连接概念节点“Cup”；程序性子图中，意图节点“寻找某件商品的最低价格”连接处方节点“要找到杯子的最低价格，智能体先……，然后……”。图操作包括：（1）创建——标准化并抽取知识；（2）检索——先按知识与任务的相关性初始化检索，再经高层跳转精炼检索，最后追溯详细记忆源；图中区分新检索、先前检索和不参与检索的节点；（3）更新——原子化合并与拆分；（4）删除——衰减与压缩；当节点的效用发生原图所示条件变化时执行相应转换。

### 3.2 检索模块

本节介绍语义记忆图和程序性记忆图上的高层检索过程。详细提示模板、逐步检索算法和更多技术细节见附录 C.3。

在结构化阶段，PlugMem 构建三个彼此连接的记忆图：情景图 $G^E$、语义图 $G^S$ 和程序性图 $G^P$。$G^S$ 与 $G^P$ 都保留指向 $G^E$ 的显式来源链接，从而使检索到的知识与经验拥有可验证的依据。图 4 展示了整体记忆组织方式。

给定任务描述或查询 $Q$，基于 LLM 的检索器首先确定应重点使用哪些记忆类型：情景、语义或程序性记忆。检索主要在 $G^S$ 和 $G^P$ 上进行，采用抽象层与具体层交错的策略。当情景记忆优先时，系统仍应用下文将介绍的同一检索过程，但最终返回 $G^E$ 中通过来源关系连接的情景节点。

检索首先把 $Q$ 编码为嵌入 $q$，并用它对全部低层节点（即命题或处方节点）打分，以初始化候选集 $C_0$。在第 $t$ 跳，检索器以 $(Q,C_t)$ 为条件生成抽象查询 $q_t^a$。对于 $G^S$，$q_t^a$ 表示为一组概念；对于 $G^P$，它表示为一组意图。

抽象查询 $q_t^a$ 会与高层节点（即概念或意图节点）匹配；这些节点充当路由信号，激活相邻的低层节点，并把后者加入 $C_{t+1}$。候选集中只保留低层节点，高层节点仅作为中间遍历信号。当 $|C_t|$ 超过预定义预算（例如 top-$K$）时，系统依据相关性和重要性对候选项重新排序并剪枝。该多跳检索过程持续迭代，直至积累足够证据，或达到最大跳数。

### 3.3 推理模块

推理模块在测试时运行，把检索到的记忆转化为执行智能体可立即采取行动的指导。许多情况下，检索结果可能包含多条相互重叠或冗长的过往交互描述；它们各自都相关，但合在一起对当前决策而言存在冗余。推理模块使用 LLM 聚合并浓缩这些信息，形成紧凑、与任务对齐的表示，把多条消息的共同信号蒸馏为一条可执行摘要。更多技术细节见附录 C.4。

### 3.4 总结与支持的操作

如图 3 和图 4 所示，PlugMem 从智能体原始交互出发，对情景记忆进行标准化，抽取语义与程序性知识，将其组织为结构化记忆图，并支持对已存经验进行检索和推理，以辅助下游决策。在系统层面，PlugMem 支持一组基本记忆图操作，包括：（i）**创建**，把新观察到的情景经验插入结构化记忆；（ii）**检索**，根据任务或查询检索相关的语义、程序性或情景记忆；（iii）**更新**，当新证据出现时修订已有记忆条目；（iv）**删除**，移除过时或低效用记忆。

主文中的基准评估主要考察创建和检索操作。有关更新和删除操作有效性的补充实验见附录 C.5。

## 4 实验

### 4.1 评估框架

我们使用各基准的标准指标（如准确率、F1 分数和成功率等）评估 PlugMem，以衡量终端任务性能。然而，仅凭这些指标不足以评估智能体记忆，因为它们无法刻画决策相关效用与智能体侧成本之间的权衡。因此，我们提出一种信息论度量，用于量化记忆模块贡献的、每个记忆 token 所对应的决策相关信息增益。具体而言，对于每个状态为 $s$、标准最优动作为 $a^*$ 的决策实例，设基础智能体的先验信念为 $P_{\mathrm{base}}(a^*\mid s)$，读取记忆 $m$ 后、记忆增强智能体的后验信念为 $P_{\mathrm{mem}}(a^*\mid s,m)$。我们将**决策信息增益**定义为逐点互信息（point-wise mutual information，PMI）：

$$
\operatorname{PMI}(a^*;m\mid s)
=\log_2\frac{P_{\mathrm{mem}}(a^*\mid s,m)}{P_{\mathrm{base}}(a^*\mid s)}。
\tag{1}
$$

再以记忆长度 $|m|$（单位为 token）进行归一化，得到**记忆信息密度**（bit/token）：

$$
\rho(a^*,m)=\frac{\operatorname{PMI}(a^*;m\mid s)}{|m|}。
\tag{2}
$$

在整个数据集上，我们通过总和之比报告一种全局摊销密度：

$$
\rho_{\mathrm{global}}
=\frac{\sum_i\operatorname{PMI}(a_i^*;m_i\mid s_i)}{\sum_i|m_i|}。
\tag{3}
$$

该指标以 bit/token 为单位，与任务无关，因而可跨任务比较。其数值在不同任务上的差异反映效用-成本权衡：记忆带来的决策相关增益更大，或使用的 token 更少时，密度更高；基础智能体本就能很好地解决任务，或有用记忆必须以冗长形式表达时，密度较低。完整分析框架以及主文说明之外的其他组成部分详见附录 D。

### 4.2 通用实验设置

我们在三个异构基准上使用完全相同的 PlugMem 进行评估；这些基准分别考察智能体记忆的不同方面：（i）LongMemEval（Wu et al., 2024），考察长时程对话记忆；（ii）HotpotQA（Yang et al., 2018），考察多跳知识检索与推理；（iii）WebArena（Zhou et al., 2024），考察基于 Web 的交互式决策。

我们在所有基准上采用统一的记忆评估协议，以确保公平比较。除非另有说明，基于嵌入的检索使用 NV-Embed-v2（Lee et al., 2025）。检索在固定预算下执行（例如 top-$K$ 检索）；在每个基准内，各方法的预算保持一致。对于 PlugMem，结构化与推理模块使用 Qwen2.5-32B/72B-Instruct（Qwen et al., 2025）和 GPT-4o（OpenAI, 2024）实例化；除非明确说明，所有基础智能体均由同一模型驱动。各方法的解码参数保持固定，以消除混杂效应。

对于 WebArena，我们区分在线与离线评估阶段，以评估知识迁移和复用；详细实验设计见第 4.5 节。

基线分为三类：（i）**Vanilla**，不依赖外部记忆；（ii）**任务无关**，采用并非为相应基准定制的通用检索或智能体记忆机制；（iii）**任务特定**，引入基准特定的记忆表示或检索启发式规则。

具体基准的详细设置、提示模板和基准层分析见附录 E。

### 4.3 RQ1：PlugMem 能否跨任务提升性能与记忆效率？

第一个研究问题考察：单一的任务无关记忆模块能否在异构任务上持续提升智能体性能，同时减少记忆消耗。LongMemEval、HotpotQA 和 WebArena 的结果（表 3、表 4、表 5）尽管面对差异巨大的任务结构和交互模态，却呈现一致模式。第一，PlugMem 的终端任务性能优于任务无关基线和任务特定基线。第二，这些增益是在向智能体上下文注入显著更少记忆 token 的情况下实现的。因此，在第 4.1 节提出的统一信息论分析下，PlugMem 取得最高的信息增益密度。图 5 的效用-成本可视化进一步说明了这种权衡：在三个基准上，PlugMem 始终向“效用更高、智能体侧成本更低”的方向移动。

结果表明，PlugMem 检索到了更多与决策相关的记忆。通过把原始经验抽象为紧凑的命题性和处方性知识，记忆模块能够提供更高的单 token 效用，使基础智能体在严格的上下文预算下进行更有效的推理。

**图 5：各基准上的效用-成本分析。** 每个点代表一种记忆方法；横轴为智能体侧记忆成本（单位为 token），纵轴为决策相关效用（单位为 bit）。连接某点与原点的直线斜率对应信息密度（bit/token）。曲线通过在随机抽取的基准任务子集上扫描记忆 token 预算得到，展示记忆效用如何随预算增加而先上升、继而饱和，并最终可能下降；例如，新增记忆会给决策引入噪声或干扰，从而产生反作用。PlugMem 始终实现更有利的效用-成本权衡：在各基准上，它都能以更小的记忆预算提供更高的决策相关效用，因而支配先前方法。三个子图分别为 LongMemEval、HotpotQA 和 WebArena；横轴均为记忆 token 数，纵轴均为 PMI，箭头方向表示“效用更高、成本更低”。

**表 3：LongMemEval 结果。** #TokAvg. 表示记忆 token 的平均长度。实验使用 NV-Embed-v2（简称 NVE）作为检索嵌入模型，并使用 Qwen2.5-32B（Q32）、Qwen2.5-72B（Q72）或 GPT-4o（4o）作为结构化和推理的基础 LLM。$^*$ 表示取自先前工作的结果；$^\dagger$ 表示只在完整基准的一个子集上评估的方法。最佳结果以粗体表示。

| 类别 | 方法 | Emb | LLM | Acc. | #TokAvg. | 信息密度 |
|---|---|---|---|---:|---:|---:|
| 朴素基线 | 无上下文 | - | Q72 | 14.8 | - | - |
| 朴素基线 | 全部上下文 | - | Q72 | 62.4 | 107K | 4.2e-5 |
| 任务无关 | Vanilla Retrieval | NVE | Q72 | 63.6 | 3742.52 | 1.2e-3 |
| 任务无关 | A-Mem$^\dagger$ | NVE | 4o + Q72 | 61.0 | 4225.85 | 1.0e-3 |
| 任务特定 | Zep$^*$ | BGE-m3 | 4o | 71.2 | 1600 | - |
| 任务特定 | LiCoMemory$^\dagger$ | NVE | 4o + Q72 | 73.0 | 5914.85 | 9.3e-4 |
| 本文 | **PlugMem** | NVE | Q32 + Q72 | **75.1** | **362.58** | **1.6e-2** |

注：其他缩写包括 Llama3.3-70B（L70）。

**表 4：HotpotQA 结果。** EM 表示精确匹配（Exact Match）。$^*$ 表示取自先前工作的结果。上界性能以下划线表示，最佳结果以粗体表示。

| 类别 | 记忆方法 | Emb | LLM | EM | F1 | #TokAvg. | 信息密度 |
|---|---|---|---|---:|---:|---:|---:|
| 朴素基线 | 无上下文 | - | Q32 | 22.1 | 31.0 | - | - |
| 朴素基线 | <u>黄金上下文</u> | - | Q32 | <u>69.2</u> | <u>82.1</u> | 86.5 | 1.6e-1 |
| 任务无关 | Vanilla Retrieval | NVE | Q32 | 51.7 | 62.7 | 659.2 | 1.2e-2 |
| 任务无关 | A-Mem | NVE | Q32 | 43.8 | 53.6 | 695.6 | 1.2e-2 |
| 任务特定 | GraphRAG$^*$ | NVE | L70 | 55.2 | 68.6 | - | - |
| 任务特定 | RAPTOR | NVE | Q32 | 56.7 | 69.7 | 806.3 | 1.1e-2 |
| 任务特定 | PropRAG | NVE | Q32 | 57.8 | 72.1 | 626.1 | 1.9e-2 |
| 任务特定 | HippoRAG2 | NVE | Q32 | 60.0 | 73.3 | 595.1 | 1.9e-2 |
| 本文 | **PlugMem** | NVE | Q32 | **61.4** | **74.1** | **81.6** | **1.4e-1** |

**表 5：WebArena 结果。** SR 表示成功率（Success Rate）。每个网站域均被划分为（在线/离线）集合。Shopping 包含（38/149）个任务，GitLab 包含（37/143）个任务，Multi-site 包含（10/38）个任务。$^*$ 表示取自先前工作的结果。AWM 原生不支持 Multi-site 任务。最佳 SR 以粗体表示。

| 类别 | 方法 | Emb | Agent | Shopping SR%（on/off） | GitLab SR%（on/off） | Multi-site SR%（on/off） | #TokAvg. | 信息密度 |
|---|---|---|---|---:|---:|---:|---:|---:|
| 朴素基线 | AgentOccam$^*$ | - | 4o | 42.1/43.6 | 37.8/39.2 | 20.0/15.8 | - | - |
| 任务无关 | Van. Retrieval | NVE | Q32+4o | 43.0/42.3 | 40.5/41.3 | 10.0/18.4 | 8733 | 2.0e-6 |
| 任务无关 | A-Mem | NVE | Q32+4o | 44.7/44.3 | 37.8/38.5 | 20.0/15.8 | 20516 | 3.4e-7 |
| 任务特定 | AWM | - | 4o | 26.3/28.2 | 27.0/27.3 | - | 696 | -7.9e-4 |
| 本文 | **PlugMem** | NVE | Q32+4o | **52.6/58.4** | **51.4/55.2** | **20.0/21.6** | **301** | **1.4e-3** |

### 4.4 RQ2：PlugMem 各组件分别发挥什么作用？

接下来，我们通过在全部基准上的消融实验（表 6、表 7、表 8），分析 PlugMem 中各组件的贡献。移除检索会造成跨任务最严重的性能下降，这强调了：只有在决策时能够访问相关经验，记忆才有用。然而，这并不意味着性能增益仅由检索驱动。更准确地说，检索决定记忆是否能发挥作用，而检索的有效性又受记忆表示方式的限制。结构化模块在适当的抽象层次上组织记忆，从而改进检索，使检索器能更有效地识别和访问任务相关知识。推理模块则发挥互补作用，主要通过控制检索知识的压缩和使用方式来影响记忆效率。

总体而言，检索决定记忆是否有帮助，结构化决定可以检索到什么，推理则决定如何高效使用检索到的记忆。因此，检索构成决定性的瓶颈；而当检索机制就位后，结构化与推理会调节其有效性和效率。

**表 6：LongMemEval 消融研究。**

| 方法 | Emb | LLM | Acc. | #TokAvg. | 信息密度 |
|---|---|---|---:|---:|---:|
| PlugMem | NVE | Q32 + Q72 | 75.1 | 362.58 | 1.6e-2 |
| 无结构化 | NVE | Q32 + Q72 | 62.8 | 311.12 | 1.4e-2 |
| 无检索 | - | Q32 + Q72 | 57.2 | 591.2 | 6.8e-3 |
| 无推理 | NVE | Q32 | 72.4 | 9478.59 | 5.8e-4 |

**表 7：HotpotQA 消融研究。**

| 方法 | Emb | LLM | EM | F1 | #TokAvg. | 信息密度 |
|---|---|---|---:|---:|---:|---:|
| PlugMem | NVE | Q32 | 61.4 | 74.1 | 81.6 | 1.4e-1 |
| 无结构化 | NVE | Q32 | 51.4 | 62.0 | 116.7 | 6.8e-2 |
| 无检索[^2] | - | Q32 | 20.0 | 24.3 | 8.01 | -3.8e-1 |
| 无推理 | NVE | Q32 | 59.3 | 71.8 | 635.1 | 1.7e-2 |

[^2]: 在“无检索”设置中，我们随机抽取语料条目，使其适配推理模块的上下文窗口。抽取的条目通常无关，因此推理模块几乎不输出蒸馏后的上下文，致使 #TokAvg. 小得多（例如 8）。

**表 8：WebArena 消融研究。** “无人工演示”表示在线评估与离线评估之间不向记忆图插入人工演示。我们为 Shopping/GitLab/Multi-site 分别收集了 23/18/5 条演示。

| 方法 | Emb | Agent | Shopping SR%（on/off） | GitLab SR%（on/off） | Multi-site SR%（on/off） | #TokAvg. | 信息密度 |
|---|---|---|---:|---:|---:|---:|---:|
| PlugMem | NVE | Q32+4o | 52.6/58.4 | 51.4/55.2 | 20.0/21.6 | 301 | 1.4e-3 |
| 无结构化 | NVE | Q32+4o | 50.0/51.7 | 41.7/42.0 | 20.0/18.4 | 243 | 7.2e-4 |
| 无检索 | NVE | Q32+4o | 42.1/46.3 | 45.8/44.0 | 20.0/15.8 | 286 | 3.8e-4 |
| 无推理 | NVE | Q32+4o | 52.6/53.7 | 41.7/43.4 | 20.0/18.4 | 374 | 5.6e-4 |
| 无人工演示 | NVE | Q32+4o | 52.6/52.3 | 51.4/51.0 | 20.0/18.4 | 261 | 1.2e-3 |

### 4.5 RQ3：知识迁移与记忆复用

第三个研究问题评估智能体记忆能否支持可迁移知识，使其跨任务实例和环境泛化。为此，我们在 WebArena 上设计了一套专门的评估协议。

我们重点考察 Shopping、GitLab 和 Multi-site 子集。Shopping 和 GitLab 是程序密集型领域，任务数量大且基线成功率较低（Yang et al., 2025b），可以避免饱和效应。Multi-site 子集还要求智能体跨多个网站运用组合技能，因此尤其困难，也非常适合评估跨任务知识复用。

为显式检验记忆演化与复用，我们依据 WebArena 的意图模板，把任务划分为在线集和离线集。对于每个模板，一个实例分入在线集，其余实例构成离线集。首先在在线集上评估智能体，此时允许 PlugMem 插入和检索记忆。随后，我们向记忆模块补充少量高质量人工演示；这些演示代表程序性知识的外部来源，类似教程或经验分享。最后，我们在离线集上评估，此时基本禁用记忆插入，只允许检索。该协议把记忆作为可复用知识而非情景回忆来评估。离线集上的受评智能体可以视为一个继承预建记忆图的新智能体，用于检验积累的程序性和语义知识能否缓解冷启动问题。其他实验细节和实现说明见附录 E.3。

如表 5 所示，PlugMem 显著提高了各领域离线集上的成功率，并在 Multi-site 任务上取得强劲增益。这些结果表明，积累的程序性和语义知识能够得到有效复用，既缓解冷启动问题，也支持组合泛化。

### 4.6 讨论：为何任务无关记忆能够胜过任务特定设计？

一个自然的问题是：任务无关记忆模块为何能胜过专门针对单个基准设计的系统？我们的结果表明，关键差异并不在于排斥任务特定启发式规则，而在于优先解决让智能体记忆真正有效的根本因素。任务特定设计常通过定制记忆单元或转换来编码基准特定洞见，并隐式假设需要时总能取得相关记忆。这类方法虽在其适用范围内有效，却混淆了记忆转换与记忆效用。相比之下，我们的发现强调，智能体记忆从根本上由检索驱动。正如各项消融实验一致表明的，如果检索无效，无论是任务特定抽象，还是精心设计的记忆表示，都无法转化为性能提升。与此同时，仅有检索也不够。它的有效性受记忆结构方式限制，因为结构化决定经验的哪些方面可以被索引和恢复。以知识为中心的结构化使系统能够检索具有语义意义、与决策相关的抽象，让有用信息在决策时显现出来。

重要的是，PlugMem 被设计为任务无关的记忆骨干，专门处理各类智能体记忆设计共同面临的检索和表示挑战。从这一角度看，任务特定的记忆方法和启发式规则可以自然地叠加在 PlugMem 之上，而不必被视为它的替代方案。因此，我们的框架提供了一个公共基础，可以在其上应用任务特定的适配，以进一步改善性能。我们通过额外的任务适配实验，从经验上验证了这一观点：把先前基线中具有代表性的任务特定启发式规则和记忆转换策略集成进 PlugMem。这些适配始终能在单独使用 PlugMem 的基础上进一步提升性能，说明任务特定技术与我们的任务无关记忆设计相互补充。详细实验设置与结果见附录 F。

## 5 结论

本文提出 PlugMem——一种任务无关的插件式记忆模块。它把智能体经验组织成以知识为中心的表示，使系统能够在多种智能体任务中有效检索决策相关记忆。大量实验表明，在统一的效用-成本评估框架下，PlugMem 能够持续改善终端任务性能，同时降低智能体侧记忆成本。除独立使用外，PlugMem 还可作为一种通用记忆骨干，叠加任务特定启发式规则；任务适配实验显示，这能带来进一步增益。总体而言，这些结果表明 PlugMem 为 LLM 智能体中的可迁移、高效记忆提供了一个具有原则依据的基础，并为面向长时程决策、更加通用且可扩展的记忆系统指明方向。

## 参考文献

Anokhin, P., Semenov, N., Sorokin, A., Evseev, D., Burtsev, M., and Burnaev, E. Arigraph: Learning knowledge graph world models with episodic memory for llm agents, 2024. URL https://arxiv.org/abs/2407.04363.

Atkinson, R. C. and Shiffrin, R. M. Human memory: A proposed system and its control processes. Psychology of Learning and Motivation, 1968.

Bai, Y., Lv, X., Zhang, J., Lyu, H., Tang, J., Huang, Z., Du, Z., Liu, X., Zeng, A., Hou, L., Dong, Y., Tang, J., and Li, J. Longbench: A bilingual, multitask benchmark for long context understanding, 2024. URL https://arxiv.org/abs/2308.14508.

Diebolt, C. and Perrin, F. From stagnation to sustained growth: The role of female empowerment. American Economic Review, 103(3):545-49, May 2013. doi: 10.1257/aer.103.3.545. URL https://www.aeaweb.org/articles?id=10.1257/aer.103.3.545.

Edge, D., Trinh, H., Cheng, N., Bradley, J., Chao, A., Mody, A., Truitt, S., Metropolitansky, D., Ness, R. O., and Larson, J. From local to global: A graph rag approach to query-focused summarization, 2025. URL https://arxiv.org/abs/2404.16130.

Fang, R., Liang, Y., Wang, X., Wu, J., Qiao, S., Xie, P., Huang, F., Chen, H., and Zhang, N. Memp: Exploring agent procedural memory. CoRR, abs/2508.06433, 2025. doi: 10.48550/ARXIV.2508.06433. URL https://doi.org/10.48550/arXiv.2508.06433.

Furuta, H., Lee, K.-H., Nachum, O., Matsuo, Y., Faust, A., Gu, S. S., and Gur, I. Multimodal web navigation with instruction-finetuned foundation models, 2024. URL https://arxiv.org/abs/2305.11854.

Gutiérrez, B. J., Shu, Y., Qi, W., Zhou, S., and Su, Y. From rag to memory: Non-parametric continual learning for large language models, 2025. URL https://arxiv.org/abs/2502.14802.

Huang, Z., Tian, Z., Guo, Q., Zhang, F., Zhou, Y., Jiang, D., Xie, Z., and Zhou, X. Licomemory: Lightweight and cognitive agentic memory for efficient long-term reasoning, 2026. URL https://arxiv.org/abs/2511.01448.

Kwiatkowski, T., Palomaki, J., Redfield, O., Collins, M., Parikh, A., Alberti, C., Epstein, D., Polosukhin, I., Devlin, J., Lee, K., Toutanova, K., Jones, L., Kelcey, M., Chang, M.-W., Dai, A. M., Uszkoreit, J., Le, Q., and Petrov, S. Natural questions: A benchmark for question answering research. Transactions of the Association for Computational Linguistics, 7:452-466, 2019. doi: 10.1162/tacl_a_00276. URL https://aclanthology.org/Q19-1026/.

Lee, C., Roy, R., Xu, M., Raiman, J., Shoeybi, M., Catanzaro, B., and Ping, W. Nv-embed: Improved techniques for training llms as generalist embedding models, 2025. URL https://arxiv.org/abs/2405.17428.

Lee, K.-H., Chen, X., Furuta, H., Canny, J., and Fischer, I. A human-inspired reading agent with gist memory of very long contexts, 2024. URL https://arxiv.org/abs/2402.09727.

Lewis, P., Perez, E., Piktus, A., Petroni, F., Karpukhin, V., Goyal, N., Küttler, H., Lewis, M., tau Yih, W., Rocktäschel, T., Riedel, S., and Kiela, D. Retrieval-augmented generation for knowledge-intensive nlp tasks, 2021. URL https://arxiv.org/abs/2005.11401.

Li, Z., Xi, C., Li, C., Chen, D., Chen, B., Song, S., Niu, S., Wang, H., Yang, J., Tang, C., Yu, Q., Zhao, J., Wang, Y., Liu, P., Lin, Z., Wang, P., Huo, J., Chen, T., Chen, K., Li, K., Tao, Z., Lai, H., Wu, H., Tang, B., Wang, Z., Fan, Z., Zhang, N., Zhang, L., Yan, J., Yang, M., Xu, T., Xu, W., Chen, H., Wang, H., Yang, H., Zhang, W., Xu, Z.-Q. J., Chen, S., and Xiong, F. Memos: A memory os for ai system, 2025. URL https://arxiv.org/abs/2507.03724.

Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., and Liang, P. Lost in the middle: How language models use long contexts, 2023. URL https://arxiv.org/abs/2307.03172.

Liu, X., Yu, H., Zhang, H., Xu, Y., Lei, X., Lai, H., Gu, Y., Ding, H., Men, K., Yang, K., Zhang, S., Deng, X., Zeng, A., Du, Z., Zhang, C., Shen, S., Zhang, T., Su, Y., Sun, H., Huang, M., Dong, Y., and Tang, J. Agentbench: Evaluating llms as agents, 2025. URL https://arxiv.org/abs/2308.03688.

Maharana, A., Lee, D.-H., Tulyakov, S., Bansal, M., Barbieri, F., and Fang, Y. Evaluating very long-term conversational memory of llm agents, 2024. URL https://arxiv.org/abs/2402.17753.

OpenAI. Gpt-4o system card, 2024. URL https://openai.com/index/gpt-4o-system-card/.

Ouyang, S., Yan, J., Hsu, I.-H., Chen, Y., Jiang, K., Wang, Z., Han, R., Le, L. T., Daruki, S., Tang, X., Tirumalashetty, V., Lee, G., Rofouei, M., Lin, H., Han, J., Lee, C.-Y., and Pfister, T. Reasoningbank: Scaling agent self-evolving with reasoning memory, 2025. URL https://arxiv.org/abs/2509.25140.

Packer, C., Wooders, S., Lin, K., Fang, V., Patil, S. G., Stoica, I., and Gonzalez, J. E. Memgpt: Towards llms as operating systems, 2024. URL https://arxiv.org/abs/2310.08560.

Park, J. S., O'Brien, J. C., Cai, C. J., Morris, M. R., Liang, P., and Bernstein, M. S. Generative agents: Interactive simulacra of human behavior, 2023. URL https://arxiv.org/abs/2304.03442.

Petroni, F., Piktus, A., Fan, A., Lewis, P., Yazdani, M., Cao, N. D., Thorne, J., Jernite, Y., Karpukhin, V., Maillard, J., Plachouras, V., Rocktäschel, T., and Riedel, S. Kilt: a benchmark for knowledge intensive language tasks, 2021. URL https://arxiv.org/abs/2009.02252.

Qwen, :, Yang, A., Yang, B., Zhang, B., Hui, B., Zheng, B., Yu, B., Li, C., Liu, D., Huang, F., Wei, H., Lin, H., Yang, J., Tu, J., Zhang, J., Yang, J., Yang, J., Zhou, J., Lin, J., Dang, K., Lu, K., Bao, K., Yang, K., Yu, L., Li, M., Xue, M., Zhang, P., Zhu, Q., Men, R., Lin, R., Li, T., Tang, T., Xia, T., Ren, X., Ren, X., Fan, Y., Su, Y., Zhang, Y., Wan, Y., Liu, Y., Cui, Z., Zhang, Z., and Qiu, Z. Qwen2.5 technical report, 2025. URL https://arxiv.org/abs/2412.15115.

Rasmussen, P., Paliychuk, P., Beauvais, T., Ryan, J., and Chalef, D. Zep: A temporal knowledge graph architecture for agent memory, 2025. URL https://arxiv.org/abs/2501.13956.

Sarthi, P., Abdullah, S., Tuli, A., Khanna, S., Goldie, A., and Manning, C. D. Raptor: Recursive abstractive processing for tree-organized retrieval, 2024. URL https://arxiv.org/abs/2401.18059.

Shridhar, M., Yuan, X., Côté, M.-A., Bisk, Y., Trischler, A., and Hausknecht, M. Alfworld: Aligning text and embodied environments for interactive learning, 2021. URL https://arxiv.org/abs/2010.03768.

Sodhi, P., Branavan, S. R. K., Artzi, Y., and McDonald, R. Step: Stacked llm policies for web actions, 2024. URL https://arxiv.org/abs/2310.03720.

Squire, L. R. Memory systems of the brain: a brief history and current perspective. Neurobiology of Learning and Memory, 82(3):171-177, nov 2004. ISSN 1074-7427. doi: 10.1016/j.nlm.2004.06.005.

Tan, Z., Yan, J., Hsu, I.-H., Han, R., Wang, Z., Le, L. T., Song, Y., Chen, Y., Palangi, H., Lee, G., Iyer, A., Chen, T., Liu, H., Lee, C.-Y., and Pfister, T. In prospect and retrospect: Reflective memory management for long-term personalized dialogue agents, 2025. URL https://arxiv.org/abs/2503.08026.

Trivedi, H., Balasubramanian, N., Khot, T., and Sabharwal, A. Musique: Multihop questions via single-hop question composition, 2022. URL https://arxiv.org/abs/2108.00573.

Trivedi, H., Khot, T., Hartmann, M., Manku, R., Dong, V., Li, E., Gupta, S., Sabharwal, A., and Balasubramanian, N. Appworld: A controllable world of apps and people for benchmarking interactive coding agents, 2024. URL https://arxiv.org/abs/2407.18901.

Tulving, E. Episodic and semantic memory. Organization of Memory, 1972.

Wang, J. and Han, J. Proprag: Guiding retrieval with beam search over proposition paths. In Proceedings of the 2025 Conference on Empirical Methods in Natural Language Processing, pp. 6223-6238. Association for Computational Linguistics, 2025. doi: 10.18653/v1/2025.emnlp-main.317. URL http://dx.doi.org/10.18653/v1/2025.emnlp-main.317.

Wang, L., Ma, C., Feng, X., Zhang, Z., Yang, H., Zhang, J., Chen, Z., Tang, J., Chen, X., Lin, Y., Zhao, W. X., Wei, Z., and Wen, J. A survey on large language model based autonomous agents. Frontiers of Computer Science, 18(6), March 2024a. ISSN 2095-2236. doi: 10.1007/s11704-024-40231-1. URL http://dx.doi.org/10.1007/s11704-024-40231-1.

Wang, Z. Z., Mao, J., Fried, D., and Neubig, G. Agent workflow memory, 2024b. URL https://arxiv.org/abs/2409.07429.

Wu, D., Wang, H., Yu, W., Zhang, Y., Chang, K.-W., and Yu, D. Longmemeval: Benchmarking chat assistants on long-term interactive memory. 2024. URL https://arxiv.org/abs/2410.10813.

Xu, W., Liang, Z., Mei, K., Gao, H., Tan, J., and Zhang, Y. A-mem: Agentic memory for llm agents, 2025. URL https://arxiv.org/abs/2502.12110.

Yang, K., Liu, Y., Chaudhary, S., Fakoor, R., Chaudhari, P., Karypis, G., and Rangwala, H. Agentoccam: A simple yet strong baseline for llm-based web agents, 2025a. URL https://arxiv.org/abs/2410.13825.

Yang, K., Liu, Y., Chaudhary, S., Fakoor, R., Chaudhari, P. A., Karypis, G., and Rangwala, H. Agentoccam: A simple yet strong baseline for llm-based web agents. In Yue, Y., Garg, A., Peng, N., Sha, F., and Yu, R. (eds.), International Conference on Representation Learning, volume 2025, pp. 97533-97565, 2025b.

Yang, Z., Qi, P., Zhang, S., Bengio, Y., Cohen, W. W., Salakhutdinov, R., and Manning, C. D. Hotpotqa: A dataset for diverse, explainable multi-hop question answering, 2018. URL https://arxiv.org/abs/1809.09600.

Zhang, X., Chen, Y., Hu, S., Xu, Z., Chen, J., Hao, M. K., Han, X., Thai, Z. L., Wang, S., Liu, Z., and Sun, M. Inftybench: Extending long context evaluation beyond 100k tokens, 2024. URL https://arxiv.org/abs/2402.13718.

Zhao, A., Huang, D., Xu, Q., Lin, M., Liu, Y.-J., and Huang, G. Expel: Llm agents are experiential learners, 2024. URL https://arxiv.org/abs/2308.10144.

Zhou, J., Wang, J., Zhang, W., Liu, W., Zhang, Z., Lou, X., Zhang, W., Deng, H., and Wang, J. Colorbrowseragent: An intelligent gui agent for complex long-horizon web automation, 2026. URL https://arxiv.org/abs/2601.07262.

Zhou, S., Xu, F. F., Zhu, H., Zhou, X., Lo, R., Sridhar, A., Cheng, X., Ou, T., Bisk, Y., Fried, D., Alon, U., and Neubig, G. Webarena: A realistic web environment for building autonomous agents, 2024. URL https://arxiv.org/abs/2307.13854.

## 附录 A 展示 PlugMem 为何优于任务无关基线与任务特定基线的真实基准案例

### A.1 LongMemEval

**案例研究：情景记忆的层次化检索。** 表 9 展示了 PlugMem 在 LongMemEval 上层次化检索情景记忆的一个示例。

**表 9：展示情景记忆层次化检索的检索轨迹示例 1。**

**LongMemEval 案例研究：层次化检索**

**查询。** “今年我参加过多少场婚礼？”

检索到的语义记忆

▷ 语义记忆 669：用户姐姐的婚礼非常棒；用户与姐姐一起筹备婚礼，包括挑选礼服和确定菜单，度过了一段愉快时光。

▷ 语义记忆 141：用户正在筹备自己的婚礼，梦想在海滩或公园举办一场小型户外仪式。用户最近参加了大学室友在城里举行的婚礼；仪式在一座俯瞰天际线的屋顶花园举行。

▷ 语义记忆 146：可使用 Instagram 和 Pinterest 等社交媒体平台关注婚礼策划师、场地和摄影师。在 Yelp 或 Google 上阅读评论，可以了解场地的口碑。

...

检索到的情景记忆

▷ 情景记忆，会话 24：日期：2023/10/15（周日）04:44。用户：我正在筹备自己的婚礼，想问问你能否给我一些如何选择完美场地的建议。顺便说一下，我上周刚参加完一位朋友的婚礼，真的很棒——新娘 Jen 穿着波西米亚风格的礼服，美极了；她的丈夫 Tom 显然被她迷住了。婚礼在乡间一座质朴的谷仓举行，非常温馨、放松……

▷ 情景记忆，会话 6：日期：2023/10/15（周日）10:57。用户：我正在筹备自己的婚礼，需要帮忙找场地。我一直梦想举办一场小型户外仪式，也许在海滩或公园。你有没有寻找完美地点的建议或资源？顺便说一下，我刚参加完大学室友在城里举行的婚礼，非常美——他们在俯瞰天际线的屋顶花园举行仪式，十分浪漫。我的朋友 Emily 终于和她的伴侣 Sarah 喜结连理；看到她们在等待婚姻平权之后如此幸福，真是太棒了……

...

**推理**

▷ 为确定你今年参加了多少场婚礼，让我们回顾所提供的相关记忆中的信息：

1. 堂/表亲在葡萄园的婚礼（八月）：你提到在八月参加了堂/表亲 Rachel 在葡萄园举行的婚礼。
2. 大学室友在城市的婚礼：你提到参加了大学室友的婚礼，该婚礼在屋顶花园举行仪式。
3. 朋友 Jen 在质朴谷仓的婚礼：你提到上周参加了朋友 Jen 在乡间质朴谷仓举行的婚礼...

**回答。** “因此，你今年参加过 3 场婚礼。”

**要点。** 该示例凸显了 PlugMem 在 LongMemEval 上优于任务无关基线的一个关键原因。Zep（Rasmussen et al., 2025）的图中虽有情景节点，却只在语义层面（语义实体与事实）执行检索；LiCoMemory（Huang et al., 2026）则只依据实体对原始对话块执行图检索。与二者不同，PlugMem 利用从情景记忆中抽取的语义记忆来检索相应的情景记忆。具体而言，检索模块先识别与查询相关的语义记忆。在本例中，语义记忆 669、141 和 146 均与“婚礼”相关，而婚礼正是问题主题。随后，系统定位每条已检索语义记忆对应的源情景记忆，并选出包含足够多已检索语义记忆的源情景。比如，语义记忆 669 与其他若干已检索记忆均从情景记忆会话 24 中抽取，因此会话 24 被选中；会话 6 亦然。可以看到，语义记忆 141 和 146 虽与问题相关，但未提供足以判定各自所指具体婚礼的信息，因而在计数时存在歧义。PlugMem 通过回溯源情景记忆成功区分了它们，这说明情景记忆十分重要，而 Zep 忽略了这一点。此外，LiCoMemory 虽会取回原始对话块，但其检索基于语义实体。与单一实体相关的情景记忆必然多于与某条特定语义记忆相关的情景记忆，因此这种方式效率较低，也可能引入更多噪声。

### A.2 HotpotQA

**案例研究：桥接实体多跳检索。** 表 10 和表 11 展示了 HotpotQA 上的两个多跳检索示例；检索过程在概念等抽象节点与语义节点等具体节点之间交替进行。

**表 10：展示桥接实体发现与证据压缩的检索轨迹示例 1。**

**HotpotQA 案例研究 1：桥接实体多跳检索**

**查询。** “‘One Less Set of Footsteps’ 是一首由某位歌手创作并演唱的歌曲；这位歌手出生于哪一年？”

第 0 跳（初始化）。查询标签：[One Less Set of Footsteps，词曲作者，演唱者，出生日期]

▷ node 5891 : “One Less Set of Footsteps” 是一首由 Jim Croce 创作并演唱的歌曲。它于 1973 年作为其专辑 “Life and Times” 的首支单曲发行。（识别出的桥接实体：Jim Croce）

▷ node 5892 : ”One Less Set of Footsteps” 在 ”Billboard” Hot 100 上最高排名第 37 位，在榜上停留了十周。

......

▷ node 15424: Mariah Carey，生于 1969 年或 1970 年 3 月 27 日，是美国歌手、词曲作者、唱片制作人及演员。

新查询 = 整合（查询，top-$k$ 节点）

第 1 跳（精炼并扩展）。查询标签：[Jim Croce，出生年份]

▷ node 5897 : James Joseph “Jim” Croce 出生于 1943 年 1 月 10 日，逝世于 1973 年 9 月 20 日。（检索到的黄金事实：Jan 10, 1943）

node 5895 : James Joseph ”Jim” Croce 是一名活跃于 1966 年至 1973 年的美国民谣与摇滚歌手。在此期间他发行了五张录音室专辑及单曲。

......

node 5883 : ”It Doesn’t Have to Be That Way” 最初于 1973 年初作为 ”One Less Set of Footsteps” 单曲的 B-side 发行，并于同年 12 月作为专辑 ”Life and Times” 的第三支也是最后一支单曲重新发行。

**推理（压缩后）。**

▷ “One Less Set of Footsteps” 由 Jim Croce 创作并演唱。

▷ Jim Croce 出生于 1943 年 1 月 10 日。

**回答。** 1943。

**表 11：展示桥接实体发现与证据压缩的检索轨迹示例 2。**

**HotpotQA 案例研究 2：桥接实体多跳检索**

**查询。** “Bill Nelson 曾作为有效载荷专家搭乘一架航天飞机飞行；该航天飞机首次发射于哪一年？”

第 0 跳（初始化）。查询标签：[Bill Nelson，有效载荷专家，航天飞机，首次发射]

▷ node 1375 (v=0.501): 1986 年 1 月，Clarence William Nelson II 成为首位进入太空的在任美国众议院议员，在 Space Shuttle Columbia 上担任 Payload Specialist。（识别出的桥接实体：Space Shuttle Columbia）

▷ node 1371: 1983 年，Byron Kurt Lichtenberg 和 Ulf Merbold 成为首批搭乘航天飞机飞行的 Payload Specialists。

......

▷ node 1383: Dirk Dries David Damiaan, Viscount Frimout，作为 payload specialist 搭乘 NASA Space Shuttle 任务 STS-45 飞行，使他成为首位进入太空的比利时人。

新查询 = 整合（查询，top-$k$ 节点）

第 1 跳（精炼并扩展）。查询标签：[哥伦比亚号航天飞机，首次发射]

▷ node 1365: Space Shuttle “Columbia”（Orbiter Vehicle Designation: OV-102）于 1981 年 4 月 12 日在任务 STS-1 中首次发射，标志着 Space Shuttle program 的首次飞行。（检索到的黄金事实：1981）

▷ node 1380: STS-61-C 的七人机组包括 ... 以及众议员 Bill Nelson (D-FL)，第二位进入太空的在任政治人物。

......

▷ node 1384: payload specialist (PS) 是被选拔并训练 ... 以执行 NASA Space Shuttle 任务的人员。

**推理（压缩后）。**

▷ Bill Nelson（Clarence William Nelson II）在 Space Shuttle Columbia 上担任 Payload Specialist。

▷ Space Shuttle Columbia 于 1981 年 4 月 12 日首次发射（STS-1）。

**回答。** 1981。

**要点。** 在两个案例研究中，PlugMem 之所以成功，是因为它明确地将“下一步检索什么”与“证据位于何处”分离开来。在第 0 跳中，检索器通常会找出一小组在语义上接近问题的低层命题，但这些命题可能只能部分解决问题。关键在于，PlugMem 不会反复在这些命题周围的局部邻域扩展——许多基于图的检索器依赖的正是 1–2 跳邻接——而是把中间结果提升为抽象路由信号，即桥接概念（例如 Jim Croce）或桥接实体/上下文（例如 Space Shuttle Columbia）。这一抽象步骤把多跳问答转化为一系列有针对性的子查询：先识别缺失的桥接，再检索作答所需的具体事实。

从技术上看，这一行为由记忆图的二分组织结构支持：高层节点（概念/意图）连接许多低层节点（命题/处方）。经由抽象层路由，检索器可以“跳出”初始种子命题的狭窄邻域，激活范围广得多的候选证据集合。这对 HotpotQA 尤其重要，因为支撑事实往往分布在不同文章或语料库中相距较远的位置。在两个示例中，第一跳检索到一条包含桥接信息的陈述，下一跳利用该桥接直接访问包含答案的陈述（出生年份或首次发射年份）。一旦桥接实体确定，与问题无关但表面相关的候选——例如其他有效载荷专家或其他创作型歌手——自然会被降低优先级。

最后，推理模块通过将检索到的候选池压缩为最小充分证据集（在这些示例中通常为两条命题）发挥互补作用，从而提升稳健性与 token 效率。总体而言，PlugMem 的从抽象到具体的路由以及有预算的候选控制扩展了可达搜索空间并加深了证据链接，而不受限于从任意起始节点出发的短程邻居扩展。

### A.3 WebArena

**案例研究：PlugMem 通过检索与推理为智能体动态构建有用指导。** 该 WebArena 案例研究表明，PlugMem 的优势不在于正确执行某一条特定工作流，而在于指导如何在决策过程中被构建、调整和使用。如表 12 所示，PlugMem 使智能体无需依赖静态工作流或冗长的情景回忆便可完成任务；它转而生成紧凑、逐步自适应的指导，直接反映智能体当前的信息瓶颈。

与任务无关的基于检索的方法相比，该任务所暴露的核心局限不在于检索覆盖率，而在于检索粒度。扁平检索系统可以浮现过去的轨迹或相似的任务描述，但这些记忆通常要么过于具体（绑定于特定页面或布局），要么过于冗长而无法直接付诸行动。因此，智能体必须在内部解释并调和大量松散相关的信息，导致上下文爆炸和脆弱的决策。相比之下，PlugMem 检索程序性知识单元——例如类别导航、基于约束的过滤，以及部分可观测条件下的排序——并进一步通过其推理模块将它们压缩为简洁的、与决策对齐的指令。其收益不仅是更少的 token，更是检索到的记忆与智能体即时控制决策之间更锐利的对齐。

在另一端，AWM 和 SteP 等任务特定的网页智能体会为每个领域硬编码或累积一条规范工作流。当任务与预设结构高度匹配时，这种方法很有效；但随着任务展开，此类工作流缺乏调整内部逻辑的能力。本案例表明，网页导航任务并非单一整体：智能体会在性质不同的推理范式之间切换，例如定位正确的语义范围、施加数值约束，以及在信息不完整时处理并列情况。静态工作流无法预见这些变化。尽管 AWM 会在任务完成后更新工作流，但它不会在单项任务内部修订指导，也不会根据中间观察调整指导。相比之下，PlugMem 在每一步都重新计算指导，使智能体策略能够随环境揭示的新结构而演化。

更广泛地说，该示例表明 PlugMem 的优势来自于将知识作为记忆访问的单元，而非原始轨迹或固定工作流。任务无关方法之所以失败，是因为它们检索了过多非结构化经验；任务特定方法之所以失败，是因为它们检索的适应性过少。PlugMem 占据中间地带：它将经验抽象为可复用的程序性知识，同时保留通过推理将该知识情境化并重新特化的能力。这种组合使 PlugMem 能够在 WebArena 任务间泛化，同时仍能产生高度定向的、步骤特定的指导，而这是任务无关设计与任务特定设计都无法可靠实现的。

**表 12：PlugMem 辅助 WebArena 智能体的示例 1。**

**WebArena 案例研究：自适应检索与精细化指导**

**查询。** “ 在 100 到 200 的预算范围内，从 meat substitute 类别中购买评分最高的产品。 ”

**PlugMem 推理。**

▷ 推理：
用户的目标是在 100 到 200 的预算范围内，从“meat substitute”类别中购买评分最高的产品。当前观察并未提供对“meat substitute”类别或相关产品的直接访问。此外，页面上可见的产品似乎并不属于“meat substitute”类别。为了高效定位所需产品，智能体应使用下拉菜单导航至“meat substitute”类别，然后应用价格筛选，将选项缩小到指定的预算范围。如果该平台支持高级搜索，也可以用它按类别和价格范围进行筛选。

▷ 最终信息：
1. 使用下拉菜单导航至“Grocery & Gourmet Food”类别，然后定位“meat substitute”子类别。
2. 一旦进入“meat substitute”类别，应用价格筛选以显示 100 到 200 范围内的产品。
3. 按评分对结果排序，以识别预算内评分最高的产品。
4. 如果该平台支持高级搜索，使用它按类别（“meat substitute”）和价格范围（100 到 200）进行筛选，以简化该过程。
5. 一旦识别出评分最高的产品，继续将其加入购物车。

**AgentOccam 动作 1：** hover [1490]。（智能体操作下拉菜单）
...

**PlugMem 推理。**

▷ 推理
用户的目标是在 100 到 200 的预算范围内，从“Meat Substitutes”类别中购买评分最高的产品。当前页面列出了该类别中的各种产品，但预算约束将选项缩小到仅那些定价在 100 到 200 之间的产品。为了高效识别该范围内评分最高的产品，智能体应应用 100-199.99 的价格筛选，然后按评分对结果排序。这确保只考虑相关产品，并选出评分最高的那一个。

▷ 最终信息
1. 通过点击“Price”部分下的对应链接，应用 100.00 - 199.99 范围的价格筛选。
2. 一旦显示筛选后的结果，按“Rating”对产品排序，以识别评分最高的产品。
3. 从筛选列表中选择评分最高的产品并点击“Add to Cart”。
4. 在继续之前，确认该产品满足预算与评分标准。

**AgentOccam 动作 3：** click [6093]。（智能体选择价格范围）

**PlugMem 推理。**

▷ 推理
用户的目标是在 100 到 200 的预算范围内，从“Meat Substitutes”类别中购买评分最高的产品。当前页面已经显示按指定价格范围筛选后的产品。下一步是在列出的选项中识别评分最高的产品并将其加入购物车。该观察提供了关于产品的细节，包括其名称、价格和评分，这些对于做出选择是必要的。

▷ 最终信息
1. 查看当前页面上列出的产品并识别评分最高的那一个。基于该观察：
- “Beyond Meat Beef Beefy Crumble, 5 Pound – 2 per case” 的评分为 50，价格为 108.27。
- 其他产品在观察中没有可见评分。
2. 由于“Beyond Meat Beef Beefy Crumble”具有最高评分且落在预算范围内，继续点击该产品的“Add to Cart”按钮。
3. 在进一步继续之前，确认该产品已成功加入购物车。

**AgentOccam 动作 4：** click [10183]。（智能体检查该商品信息）
...

## 附录 B 相关工作

### B.1 基于认知科学的智能体记忆

认知科学长期以来研究人类记忆的机制，为如何在智能体中利用记忆提供了有价值的洞见。Atkinson–Shiffrin 记忆模型将长期记忆定义为可以在较长时期内保持并回忆的信息（Atkinson & Shiffrin, 1968）。长期记忆进一步被划分为三种主要类型（Tulving, 1972; Squire, 2004）：语义记忆，存储概念与事实性知识；程序性记忆，负责知道如何执行动作与技能；以及情景记忆，表示个人经历与事件的原始记录。对于智能体而言，与环境的大多数交互都是情景记忆，例如原始上下文与工作轨迹。从这些情景经验中，智能体可以提取更可复用的表征。例如，用户偏好可以被提炼为语义记忆（Li et al., 2025），而网页导航技能可以被抽象为程序性记忆（Wang et al., 2024b）。在此意义上，语义记忆与程序性记忆并非孤立的组件，而是从情景记忆中导出的结构化抽象。知识可以被视为对记忆的进一步抽象。具体而言，命题知识对应于语义记忆的高度结构化且可验证的抽象，而处方知识则表示程序性记忆的一般化、可复用形式，描述如何完成目标导向的任务（Diebolt & Perrin, 2013）。

受这些认知理论启发，许多研究探索了智能体的记忆机制。情景记忆与语义记忆已被引入以处理长上下文信息，例如文档理解智能体中的长文档（Lee et al., 2024）以及会话智能体中的长期用户–智能体交互（Li et al., 2025）。此外，一些方法构建同时包含情景子图与语义子图的知识图谱，以更好地组织和管理智能体记忆（Anokhin et al., 2024; Rasmussen et al., 2025）。程序性记忆也被纳入决策智能体，尤其是网页智能体，其中累积的经验被用于改进未来策略（Zhao et al., 2024）。这一研究方向走向基于传入记忆持续精炼其决策策略的自进化智能体（Wang et al., 2024b; Fang et al., 2025; Ouyang et al., 2025）。

然而，大多数现有方法是为特定任务设计的，无法同时有效支持所有记忆类型。相比之下，PlugMem 跨多样化任务标准化异构情景记忆，并将提取的知识存储在命题–处方双知识图谱中，从而克服先前方法中存在的任务可泛化性局限。

### B.2 记忆模块设计

在智能体系统中，外部记忆最常见的实现方式是通过检索。智能体将过去的交互、观察或文档存储在模型上下文之外，并在推理时检索相关信息以条件化决策制定。这一检索优先范式构成了外部记忆的基本原型，几乎所有后续记忆设计都可以被视为建立在其上的扩展。

早期的任务无关记忆系统对非结构化情景记忆采用扁平检索。原版检索与原版 RAG 将原始交互历史或文档存储为文本，并基于与当前查询的相似度检索相关片段（Lewis et al., 2021; Wang et al., 2024a）。这些方法提供了简单且广泛适用的记忆接口，但检索到的内容仍然紧密绑定于特定情景。因此，检索到的记忆在跨不同决策上下文复用时，往往包含冗余细节并表现出低信息密度。

为提高检索有效性，后续工作在情景记忆之上引入结构。基于图的与层次化检索系统组织存储内容，以支持多跳检索、聚合或全局推理（Edge et al., 2025; Sarthi et al., 2024）。相关方法通过改变记忆图的内部组织来进一步精炼检索，例如对路径排序或链接命题（Gutiérrez et al., 2025; Wang & Han, 2025）。尽管这些方法在结构与遍历策略上有所不同，它们在很大程度上保持了相同的检索导向抽象。记忆仍然围绕情景痕迹、实体或文本跨度来组织，结构化机制主要服务于改进对已存储经验的访问，而非转化其内容。

并行地，任务特定记忆系统探索更为激进的记忆处理形式。面向会话智能体的时序知识图谱、面向网页导航的工作流记忆，以及推理导向的记忆库，显式地从经验中提取更高层次的表征（Gutiérrez et al., 2025; Wang et al., 2024b; Ouyang et al., 2025）。这些系统表明，将经验抽象为可复用表征可以在固定任务设置中显著提升性能。然而，抽象过程通常由任务特定假设所引导，这限制了记忆模块跨异构智能体与环境的复用。

综合来看，先前工作揭示了在如何对待记忆上的一个关键区分。一些系统侧重于改进对情景经验的检索，而另一些则隐式或显式地将经验转化为更抽象的形式。只有后者使记忆能够跨情景与任务泛化，因为情景记忆本质上绑定于特定智能体与交互轨迹，而语义知识与程序性知识则抽象掉任务特定的执行细节。这一观察将记忆到知识的转化激励为可复用智能体记忆的一项关键操作。

PlugMem 建立在这一演进之上，使记忆到知识的转化成为记忆模块的一个显式且核心的组件。PlugMem 不是将情景痕迹作为检索的主要单元，而是围绕对应于语义记忆与程序性记忆的命题知识单元与处方知识单元来组织记忆。这些知识单元作为记忆组织、检索与推理的基础，而情景记忆被保留为可验证的源证据。通过将知识提升为记忆操纵的单元，PlugMem 使记忆能够在无需任务特定重新设计的情况下跨智能体与任务复用，同时保持与基于检索的外部记忆范式兼容。

### B.3 记忆基准与评估范围

广泛的基准已被用于研究语言模型与智能体中的记忆相关行为，涵盖多样化的任务设置与评估目标。一条突出的工作线侧重于长上下文利用，评估模型是否能够在单个推理窗口内访问并推理大规模输入，例如长文档理解、叙事理解，或 needle in a haystack 式检索（Liu et al., 2023; Bai et al., 2024; Zhang et al., 2024）。这些基准评估上下文使用与注意力行为，但不对跨时间的记忆累积、更新或复用建模。另一条工作线侧重于长时程会话智能体与以用户为中心的记忆，评估系统是否能够在延展的交互中累积、维持并复用诸如偏好、个人属性与对话上下文等信息（Wu et al., 2024; Maharana et al., 2024）。这些基准显式强调情景累积以及对冗余与噪声的鲁棒性，并常被用于研究交互设置中的记忆抽象。另一类基准评估在静态知识源上的检索增强问答，包括事实型与多跳 QA 任务，其中记忆被当作固定的外部仓库，主要挑战在于检索与证据组合而非记忆演化（Petroni et al., 2021; Kwiatkowski et al., 2019; Yang et al., 2018; Trivedi et al., 2022）。尽管对研究语义检索与组合有效，这些基准假定一个静态记忆库，并不刻画记忆如何通过交互被构建或精炼。在智能体设置中，额外的基准评估交互环境中的端到端任务性能，包括网页导航、工具使用与具身推理，其中记忆与规划、感知及动作执行相互作用（Zhou et al., 2024; Trivedi et al., 2024; Furuta et al., 2024; Shridhar et al., 2021; Liu et al., 2025）。在此类基准中，记忆往往与其他智能体能力纠缠在一起，使得难以分离记忆特定的贡献。

尽管这些基准在表面形式上差异很大，它们各自只探测记忆行为的有限切片。总体而言，先前工作揭示了一个碎片化的评估图景，其中没有任何单一基准对记忆如何在智能体决策中被累积、抽象、组织与复用提供全面覆盖。

在这一图景中，记忆基准可以按记忆如何被构建、组织与复用来区分。一些基准强调情景经验的长期累积，要求智能体从延展的交互历史中提取稳定信息。另一些侧重于对语义知识的结构化访问，强调对记忆的多跳检索与组织。第三类强调程序性复用，其中智能体必须将从先前经验中获得的技能或工作流泛化到新的任务实例。没有任何单一基准同时捕捉所有这些方面，这激励了对互补评估套件的需求。

基于这一视角，我们有意选择三个探测不同且互补的记忆行为的基准。LongMemEval 面向长时程会话设置，其中记忆随时间累积，且必须从嘈杂的情景交互中抽象出相关信息（Wu et al., 2024）。HotpotQA 评估在大规模静态知识源上的语义记忆组织与多跳检索，作为结构化记忆访问与组合的典范基准（Yang et al., 2018）。WebArena 评估交互环境中的程序性记忆，其中智能体必须跨从共享意图模板导出的多个任务实例复用并适应先前获得的策略（Zhou et al., 2024）。

除了对情景累积、语义组织与程序性复用的互补覆盖之外，所选基准在其各自的评估设置中具有代表性。LongMemEval、HotpotQA 与 WebArena 在先前工作中均已被广泛采用，已有广泛的记忆与智能体基线在这些基准上得到评估。这一既定用法使现有系统能够在一致协议下被可靠复现与比较，减少由任务特定调优引入的评估偏差。因此，在这些基准上评估记忆模块能够实现跨异构记忆设计的更公平、更稳定的比较，同时支持超越任何单一任务或交互模式而泛化的结论。

## 附录 C 实现细节

**表 13：通过标准化形式化构建分层记忆系统。** PlugMem 采用分层记忆图：首先将原始交互（$\tau$）形式化为情景记忆（阶段 1）。随后，以这段形式化历史为输入，归纳语义和程序性知识块（阶段 2）。所有记忆类型均通过通用 API，借助标准数据操作进行管理。

| 记忆类型 | 构建逻辑（输入 $\rightarrow$ 输出） | 图结构 | 通用 API 与认知启发式规则 |
|---|---|---|---|
| **阶段 1：标准化**；情景形式化 | 输入：原始智能体轨迹（$\tau$）。过程：$\operatorname{Formalize}(\tau)\rightarrow M_{\mathrm{epi}}$。作用：把非结构化日志转为标准化源节点。 | 源节点（事件窗口）。边：知识 $\xleftarrow{\mathrm{proves}}$ 源。 | **1. 创建（摄取与归纳）**：`memory.create(trajectory)`；把原始 $\tau$ 形式化为标准化情景记忆，继而触发归纳，从新的情景实体中抽象事实/技能。**2. 检索（联想）**：`memory.retrieve(goal, state)`；将目标和状态映射至意图/概念索引并检索知识块，随后回溯至情景源以获取上下文。**3. 更新（反思与精炼）**：`memory.update(feedback)`；检测并缓解矛盾（语义），或优化工作流效率（程序性）。**4. 删除（遗忘与剪枝）**：`memory.delete(criteria)`；使低效用节点（语义/程序性）衰减，同时压缩旧情景节点以节省空间。 |
| **阶段 2：结构化**；语义事实归纳 | 输入：情景记忆（$M_{\mathrm{epi}}$）。过程：$\operatorname{Induce}_{\mathrm{fact}}(M_{\mathrm{epi}})\rightarrow M_{\mathrm{sem}}$。作用：从具体事件轨迹中蒸馏静态真相。 | 命题节点（事实块）；概念节点（实体/术语）。边：概念 $\xleftarrow{\mathrm{mentions}}$ 命题。 | 同上。 |
| **阶段 2：结构化**；程序性技能归纳 | 输入：情景记忆（$M_{\mathrm{epi}}$）。过程：$\operatorname{Induce}_{\mathrm{skill}}(M_{\mathrm{epi}})\rightarrow M_{\mathrm{proc}}$。作用：从成功历史中综合出可复用工作流。 | 处方节点（工作流块）；意图节点（用户目标/任务）。边：意图 $\xleftarrow{\mathrm{solves}}$ 处方。 | 同上。 |

**概述。** 本附录提供面向实现的细节，作为第 3.1 节高层描述的补充。如表 13 所概括，记忆结构化模块包含两个主要阶段：阶段 1“标准化”和阶段 2“结构化”。在阶段 1，为处理不同类型的情景记忆，PlugMem 将原始智能体轨迹 $\tau$ 标准化为统一的情景记忆 $M_{\mathrm{epi}}$。具体而言，对于每轮观察和动作 $\{o_t,a_t\}$，我们提示 LLM 推导状态 $s_t$、子目标 $g_t$ 和奖励 $r_t$。在阶段 2，PlugMem 使用归纳过程，将该情景记忆 $M_{\mathrm{epi}}$ 蒸馏为语义记忆与程序性记忆。随后把这些记忆插入记忆图，并使用一系列通用 API 对其进行管理。

### C.1 情景标准化

为对时刻 $t$ 的情景记忆进行标准化，我们把原始智能体轨迹的每一轮表示为观察-动作对 $\{o_t,a_t\}$，并通过三个阶段将其形式化：

首先，我们使用前一状态 $s_{t-1}$、前一动作 $a_{t-1}$、当前观察 $o_t$ 和初始目标 $G$ 推导智能体状态 $s_t$。纳入 $G$ 可确保状态始终以智能体的初始目标为依据。具体推导方式是使用清单 1 中的提示查询 LLM。所得状态是简洁的上下文摘要；在处理长上下文智能体任务时，它可以有效缩短为蒸馏程序性记忆而使用的总体上下文长度（Wu et al., 2024; Zhou et al., 2024）。

接着，我们根据 $s_t$、$o_t$、$a_t$ 和 $G$ 推断子目标 $g_t$，以分析智能体如何把全局任务分解为渐进步骤。该步骤使用清单 2 中的提示，具有两个目的：丰富可供知识抽取使用的信息，并支持轨迹切分。具体而言，我们可以依据相邻子目标嵌入的余弦相似度对情景记忆进行分块，把原始轨迹组织成工作流。

最后，我们使用清单 4 中的提示，根据 $s_t$、$a_t$、$g_t$ 和 $o_{t+1}$ 确定奖励 $r_t$。该奖励量化动作在实现预期目标方面的有效性，从而便于抽取具有质量感知能力的程序性记忆。

**清单 1：给定前一状态 $s_{t-1}$、前一动作 $a_{t-1}$ 和当前观察 $o_t$，推导当前状态 $s_t$ 的模板。**

```text
============================================================
Prompt Get_State
============================================================
你将收到四项信息：
目标：智能体当前的目标或任务。
前一状态（时刻 t）：一段自然语言摘要，描述智能体迄今为止的上下文、历史和部分进展。
动作（时刻 t）：智能体决定接下来执行的动作，以自然语言表达。
观察（时刻 t+1）：该动作产生的结果或反馈。

你的任务是推导更新后的新状态——一段连贯的自然语言摘要，整合前一状态、动作和新观察中的所有相关信息。

应遵循的步骤：
解释输入：
检查目标、前一状态、动作和观察，以理解智能体所处情形发生了什么变化，以及有关位置和时间的详细信息。

对更新进行推理：
描述新状态应当以何种逻辑过程区别于前一状态。确定取得了哪些进展、获得了哪些新信息，以及上下文或关注点如何发生了转移。

生成更新后的状态：
写出一段清晰、简洁的自然语言描述，总结智能体在时刻 t+1 的新状态。新状态应当：
-- 包含动作和观察中的全部详细信息，尤其是有关位置和时间的信息。
-- 整合最新动作与观察的结果。

输出格式：
### Reasoning
（逐步说明应如何根据输入更新新状态。）

### State
（在此提供最终的更新状态摘要。）

---
输入：
目标：{goal}
前一状态（时刻 t）：{state}
动作（时刻 t）：{action}
观察（时刻 t+1）：{observation}
```

**清单 2：给定初始目标 $G$、当前状态 $s_t$、观察 $o_t$ 与动作 $a_t$，以及下一观察 $o_{t+1}$，推导第 $t$ 轮子目标 $g_t$ 的模板。**

```text
============================================================
Prompt Get_Subgoal
============================================================
在时刻 t，智能体根据其状态、观察和总体目标采取一个动作。
你的任务是推断最能解释智能体为何选择这一动作的子目标——即时或中间目标。

使用以下信息作为上下文：总体目标：{goal} 当前状态（过去上下文的摘要）：{state} 当前观察：{observation} 时刻 t 的动作：{action}

步骤 1：推理
分析当前状态和观察与总体目标之间的关系。说明给定动作如何帮助智能体向该目标推进——可能是通过实现一个更小的中间目标。说明应明确且具有因果性：描述为什么该动作在给定上下文下是合理的。

步骤 2：子目标推断
完成推理后，推断智能体可能的子目标——用一句简短的自然语言陈述描述动作背后的即时目的。

输出格式：
### Reasoning
[你的推理过程——用几句话说明动作与目标和上下文的关系]

### Subgoal
[描述所推断子目标的简短句子]
```

**清单 3：从程序性记忆推导回报的提示模板。**

```text
============================================================
Prompt Get_Return
============================================================
你将收到：
目标描述：说明智能体试图实现什么目标的文本。
过程描述：描述智能体为实现该目标而采取的动作、作出的决策和取得的进展的文本。

你的任务：
分析智能体的过程，并结合以下方面，判断目标完成了多少：

评分标准（1--10 分）：
10：智能体完整实现目标，没有重大遗漏；动作与目标完全一致。
8--9：智能体完成了目标的大部分内容，仅有轻微缺口；高度一致但并不完美。
6--7：部分完成；智能体覆盖了许多关键要素，但仍有显著部分未完成或执行不佳。
4--5：进展有限；智能体尝试实现目标，但完成程度不足一半，或完成方式低效。
2--3：完成极少；动作与目标几乎没有关联，或只取得最低限度的结果。
1：没有有意义的进展；动作完全无助于实现目标。

重要说明：
评分只能依据完成程度以及与所述目标的一致性。
除非另有要求，否则不要提供解释或评论。
输出必须严格遵循下方格式。

输出格式：
### Score
[1 到 10 的数字]

输入：
---
目标：
{subgoal}
过程：
{procedural_memory}
```

**清单 4：给定当前状态 $s_t$、目标 $g_t$ 与动作 $a_t$，以及下一观察 $o_{t+1}$，推导动作 $a_t$ 的奖励 $r_t$ 的模板。**

```text
============================================================
Prompt Get_Reward
============================================================
你将得到：
目标：智能体的总体目标。
状态（时刻 t）：智能体在采取动作之前已知和已经完成的事项。
动作（时刻 t）：智能体所选择的单个动作。
```

```text
观测（在时刻 t + 1）：该动作产生的即时结果。
你的任务是推断奖励——即基于所得观测，用自然语言作出的一项评价，说明智能体的动作对实现总体目标的贡献（正面或负面）。
请仔细遵循以下步骤：
1. 推理过程：
解释在给定状态下该动作如何关联到目标，以及观测是否符合预期的有帮助或无帮助结果。
考虑该动作是推进了进度、造成了挫折、揭示了新的有用信息，还是浪费了精力。
总结你关于该动作对目标之因果贡献的推理。
2. 最终奖励：
使用描述性语言，对智能体的动作写出简明的自然语言评价。
该奖励应表达该动作在多大程度上、以何种方式帮助或阻碍了目标的实现。
输入：
目标：{goal}
状态（在时刻 t）：{state}
动作（在时刻 t）：{action}
观测（在时刻 t + 1）：{observation}
输出格式：
### Reasoning
[此处写你的推理过程]
### Reward
[评价该智能体动作的自然语言奖励陈述]
```

`some prompt`（原文如此）

### C.2 抽取知识

**语义。** 给定一个标准化情景单元 $e_t$，我们将语义记忆抽取为命题–标签对。我们使用清单 5 中的提示查询 LLM；该提示以固定模式返回至多 $N_{\max}=10$ 个条目。每个条目包含一个 `Statement` 字段（主文中的命题）和一个 `Tags` 字段（相关概念列表）。我们将 LLM 输出确定性地解析为 $(p,T)$ 对，其中 $p$ 是命题文本，$T$ 是相应的概念集合。

我们将这些输出物化到语义图 $G_S$ 中：为每个 $p$ 创建一个低层命题节点，为每个 $c \in T$ 创建高层概念节点，并为所有节点缓存稠密嵌入与元数据。我们添加两类边：i) 将命题连接到其概念标签的成员边（$p \rightarrow c$），以及 ii) 将每个命题链接回情景图 $G_E$ 中其来源情景单元的来源边（$p \rightarrow e_t$）。当多个命题源自同一情景单元时，我们额外用兄弟边将它们连接，以保留局部共现结构。

**清单 5：用于从情景轨迹中抽取语义知识的提示模板。**

```text
============================================================
提示 Get_Semantic
============================================================
你是一位擅长从文档中抽取精确、事实性信息的专家。
你的输出必须优先保证具体性，避免歧义，消除冗余，并严格遵守所有格式规则。
**核心指令：**
1. 事实/陈述抽取与去重：
* 从文档中识别互不相同的事实性陈述。
* 消解模糊指代：若某条陈述的主语是代词或模糊描述（例如，'the film'、'the band'、'the company'、'he'、'she'、'they'），你必须基于对整篇文档的理解改写该陈述，使主语成为取自该文档的、完全指定的具体实体名称。你不得在最终事实/陈述中保留模糊主语。
* 消解模糊指代的示例：
BAD: 'The film was directed by xxx.', 'This movie is produced by xxx'
GOOD: 'Vaada Poda Nanbargal was directed by Manikai.', 'Vaada Poda Nanbargal is produced by xxx'
* 具体表述：每条陈述必须使用明确、可识别的名称或标题来措辞。禁止使用任何模糊指代，包括但不限于：'the tour'、'the film'、'the movie'、'the band'、'it'、'he'、'she' 或 'they'。
* 具体表述的示例：
BAD: 'The tour earned over $50 million.'
GOOD: 'NSYNC's Second II None Tour earned over $50 million.'
* 陈述长度策略（重要）：每条事实/陈述不必是单个短句。一条陈述可以是将紧密相关信息归组在一起的紧凑多句块，但总共最多只能包含 4 个句子。这些句子应来自原始文档材料，仅可在消解模糊指代时做轻度编辑。
* 避免冗余：你必须将相似或重叠的事实合并为单一的、全面的陈述。不要创建多条仅以细微差异重复同一核心信息的陈述。
2. 标签生成：
* 为每条事实/陈述生成一个标签列表。
* 每条事实的标签数量是灵活的，应反映该陈述的信息密度。
* 标签应覆盖关键片段，例如：实体名称、年份、数字、国籍、语言、体裁、角色、对象类型、描述性词语等。
* 大多数标签应为从陈述中逐字复制的短文本片段（精确子串）。这些标签直接锚定于文本的表层形式。
* 在下列情况下，你可以偶尔创建并非当前陈述字面子串的额外标签：
* 你将相邻词组合成一个短语（例如，'romantic comedy film'）。
* 你从文档的另一部分引入一个表层片段（精确短语），以使当前陈述的含义明确（例如，在消解代词或诸如 'the film'、'the band' 等短语时）。
* 当某条陈述的主语是代词或模糊描述（例如，'the movie'、'the band'、'the hotel'）时，你必须至少添加一个明确点名底层实体的标签，并使用文档其他位置出现过的精确表层形式。该跨陈述标签可以来自之前或之后的句子，只要显然是同一实体。
* 若标签是动词，你必须使用其原形（lemma）形式（例如，'play'，而不是 'played'、'playing' 或 'to play'）。
* 无模式/类型标签：禁止使用元标签或类本体的类别名称，例如：'Name'、'Person'、'Year'、'Date'、'City'、'Country'、'Location'、'Genre'、'Language'、'FilmTitle' 等。标签不是类型标签；它们是承载内容的短语。
* 标签应相对较短，并按需要足够细粒度。例如，诸如 'Indian'、'Tamil-language'、'romantic'、'comedy' 等形容词与修饰语，若出现在陈述中，通常应作为分开的标签。
输出约束：
* 至多抽取 10 条事实，但质量优先于数量。若真正互不相同的事实少于 10 条，则输出更少。
* 每条事实必须提供未被其他事实覆盖的独特信息。
* 绝对不要使用泛指——每条陈述必须显式点名具体实体。
**文档：**
{observation}
输出格式：
### Facts
1. **Statement:** [statement]
**Tags:** [tag0, tag1, tag2, tag3, ...]
2. **Statement:** [statement]
**Tags:** [tag0, tag1, tag2, tag3, ...]
...
```

**程序性。** 给定带有子目标标注 $g_t$ 的标准化情景序列 $M_{\mathrm{epi}}=[e_t]_{t=1}^{T}$，我们首先通过检测相邻子目标的变化，将完整轨迹切分为连贯的工作流块。我们计算每个子目标的嵌入；若两个相邻子目标的相似度低于设定阈值，则开始一个新片段。然后，将每个片段 $M^{(i)}_{\mathrm{epi}}$ 线性化为紧凑的过程轨迹（例如逐步的状态–动作–奖励描述），并使用清单 6 中的提示送入 LLM，生成结构化的（意图，处方）对 $(u^{(i)},\pi^{(i)})$；其中 $u^{(i)}$ 抽象该片段目标，$\pi^{(i)}$ 描述一个与环境无关、可在未来任务中复用的程序。

为支持质量感知的复用，我们通过一个独立的 LLM 评估器（清单 3），以意图和片段轨迹为条件，为每个处方分配一个标量回报分数 $\rho^{(i)}\in[1,10]$。然后将 $(u^{(i)},\pi^{(i)},\rho^{(i)})$ 物化到程序性图 $G_P$ 中：创建一个意图节点和一个处方节点，并为二者缓存嵌入与元数据。为减少意图增殖，将传入的意图嵌入 $\phi(u)$ 与已有意图节点匹配；若最佳余弦相似度超过阈值 $\theta_{\mathrm{equal}}$，则通过 LLM 改写（清单 7）合并两个意图字符串并刷新节点嵌入，否则创建新的意图节点。层次关系表示为从意图到处方的有向邻接，即 $u\xrightarrow{\mathrm{SOLVES}}\pi$（实现为意图节点的子列表，因此是单向的）。最后，每个处方节点通过来源边 $\pi\rightarrow e_t$（对所有 $e_t\in M^{(i)}_{\mathrm{epi}}$）链接回其源情景证据，从而支持下游验证，并可在检索到的程序需要展开时恢复具体交互上下文。

**清单 6：用于从情景轨迹中抽取程序性经验的提示模板。**

```text
============================================================
提示 Get_Procedural
============================================================
你将得到一段智能体的话语。你的任务是分析该话语并推导
-- 该智能体正在追求的一个主目标。
-- 一条经验性洞见——概括该智能体行为的简明反思。
生成回复时请遵循此过程：
推理：分析该话语，写下可作为未来任务记忆的可泛化信息与模式。
输出目标与经验性洞见：用抽象语言写出一句描述该话语总体目标的句子。用自然语言写出一段，清晰表达该经验性洞见以及概括该智能体行为的反思。
输出格式：
### Reasoning
[你的推理过程]
### Goal
[一句归结该目标的句子]
### Experiential Insight
[表达该经验性洞见的段落]
输入：Trajectory: {trajectory}
```

```text
============================================================
Prompt Get_Return
============================================================
你将收到：
Goal Description：一段文字，说明智能体试图达成什么。
Process Description：一段文字，描述智能体为实现该目标所采取的行动、决策与进展。
你的任务：
分析智能体的过程，并判定该目标完成了多少，
同时考虑以下方面：
评分标准（分数 1--10）：
10：智能体完全达成目标，无显著遗漏；行动与目标完全对齐。
8--9：智能体完成了目标的大部分，仅有轻微缺口；对齐较强但不完美。
6--7：部分完成；智能体覆盖了许多关键要素，但留下明显部分未完成或执行不佳。
4--5：进展有限；智能体尝试了该目标，但完成不足一半，或以低效方式完成。
2--3：完成极少；行动几乎与目标无关，或只取得最低限度的结果。
1：无有意义进展；行动对实现目标毫无贡献。
重要说明：
分数仅基于完成程度以及与所述目标的对齐情况。
除非被要求，否则不要提供解释或评论。
输出必须严格遵循以下格式。
输出格式：
### Score
[1 到 10 的数字]
输入：
---
Goal：
{subgoal}
Process：
{procedural_memory}
```

**清单 7：用于合并两个相似意图的提示模板。**

```text
============================================================
Prompt Get_New_Subgoal
============================================================
每个目标可能包含重叠或互补的信息。
你的任务是将它们仔细合并为一个单一、连贯且结构良好的
目标，并保留两者中的全部重要细节。
避免冗余，并确保合并后的目标在语气上自然、一致。
输入：
Earlier goal：{goal_1}
Later goal：{goal_2}
输出：
Merged goal：[在此写下统一后的目标]
```

### C.3 交织抽象层与具体层的检索

本附录提供面向实现的细节，以补充第 3.2 节中的高层描述。PlugMem 在语义图 $G_S$ 与程序性图 $G_P$ 上执行交织式多跳检索，在以下两种操作之间交替：i）经由概念/意图节点在抽象层路由；ii）在命题/处方节点上进行具体层扩展。给定查询 $Q$，检索器由三个 LLM 提示协调：GETMODE（清单 8）选择需要侧重的记忆类型；GETPLAN（清单 9）提出下一跳的抽象信号（对 $G_S$ 为概念标签，对 $G_P$ 为子目标/意图）；MULTIHOPCTRL（清单 10）作为检索控制器，判断当前证据是否足以停止，或下一跳应优先考虑哪些低层节点。
在第 $t$ 跳，检索从低层候选池 $C_t$ 开始。我们先通过两条互补通道扩展候选。**嵌入通道：** 将当前查询文本 $Q_t$ 嵌入，并通过稠密相似度检索一组高分低层节点。**链接通道：** 并行运行 GETPLAN，产生一个抽象计划（对 $G_S$ 为概念标签，对 $G_P$ 为意图/子目标），将其与相应的高层节点匹配，再通过隶属边激活相邻的低层节点。随后取两个候选集的并集，并使用价值函数——主要依据相关性，也可结合重要性、新近性等元数据——进行重排序，再剪枝至固定预算（top-$K$）。在整个过程中，高层节点只充当中间路由信号，不会保留在 $C_t$ 中。
得到预算内候选集后，我们调用 MULTIHOPCTRL 评估其充分性。具体而言，MULTIHOPCTRL 返回一个严格的 JSON 对象，其中包含 `enough` 标志和可选的已选低层节点 ID 列表（`top node ids`）；该列表必须是可用候选的子集。当 `enough=true` 时，列表为空，检索提前终止并返回当前候选；若请求的是情景记忆，则把候选映射为源链接所指向的情景节点。否则，我们只保留已选节点（实验中设有数量上限）作为“焦点集”，并将其与上一查询整合，形成下一跳查询 $Q_{t+1}$。这种整合可由基于 LLM 的改写器执行；但我们发现，简单拼接 $Q_t$ 与所选事实的表现相当，因此出于效率考虑，将拼接设为默认方式。
检索循环重复，直到 MULTIHOPCTRL 发出充分性信号或达到跳数上限。整体控制流总结于表 14。
**清单 8：用于从查询推断可能使用的记忆类型的提示模板。**

```text
============================================================
Prompt Get_Mode
============================================================
你将得到智能体正在追求的任务描述以及来自该任务的观察
请分析任务描述与观察，以判定有效完成该任务所需的记忆类型。可能的记忆类型有三种：
情景记忆：若任务要求你基于事件回答问题，则需要此类记忆。例如，根据历史对话回答用户的问题
。
语义记忆：若任务要求你回忆客观信息，则需要此类记忆。例如，基于客观知识或信息回答问题。
程序性记忆：若任务是在交互环境中完成一个子目标，且智能体需要执行工作流，则需要此类记忆。例如，在网页导航任务中完成一条指令。
首先分析该任务与观察，并决定只需要哪一种记忆类型。
当存在冲突时，做决策时优先考虑任务描述中的信息。
输出格式：
### Reasoning
[你根据任务与观察分析需要何种记忆]
### Memory Type
## [你的最终决定，episodic_memory 或 semantic_memory 或 procedural_memory]
输入：
Task Description：{task_type}
Observation：{observation}
```

**清单 9：用于推断检索下一步抽象概念的提示模板。**

```text
============================================================
Prompt Get_Plan
============================================================
你是分析智能体目标与当前观察、并为目标导向问答系统生成检索标签的专家。
你的设定：
* Goal：智能体要完成的总体目标。
* Current Subgoal：智能体当前正在追求的子目标。（可以为 None）
* Current State：对智能体当前内部状态的描述。（可以为 None）
* Input (Current Observation)：智能体最近的观察（通常是一个问题或任务指令）。
* Task：提取一组按优先级排列的高质量标签，这些标签最有可能检索到直接有助于完成 Goal 的信息。
说明：
1. 目标导向的标签选择（CRITICAL）：
* 仔细阅读 Goal 与 Current Observation。
* 只生成 HIGHLY LIKELY 检索到解决/完成 Goal 所需证据的标签。
* 优先选择能标识以下内容的标签：
* Goal 所询问的目标实体（人物、组织、地点、作品、事件）。
* 由观察所暗示、多跳检索很可能需要的桥接实体。
* 显式约束：日期、年份、角色、头衔、独特描述语、数字。
* 避免不太可能检索到有用证据的低信号或泛化标签（例如
，"known for"、"famous"、"character"，除非 Goal 明确依赖它们）
。
2. 具体、有根据的标签：
* 标签的 MAJORITY 必须是从 Goal 或 Current Observation 中 VERBATIM 复制的短文本跨度（精确子串）。
* 你 MAY 仅在标签简短、被强烈暗示、且对检索明显必要时，添加 SMALL 数量的非字面标签（例如，规范名称展开或标准别名）。
* 若标签是动词，你 MUST 使用其基本（lemma）形式（例如，"direct"，而不是 "directed
" 或 "directing"）。
3. 优先级与数量：
* 标签总数应当适当。（不是"尽可能多"）。
* 按预期检索有用性对标签排序（最有用的在前）。
* 确保标签具有内容且相对较短。
4. CRITICAL -- 禁止的标签：
* 不要生成标签 "user"。
* 不要使用元标签或类型名，例如 "Name"、"Person"、"Year"、"Date"、"City
"、"Country"、"Location"、"Genre"、"FilmTitle" 等。
```

```text
输出格式：
### Reasoning
[你分析信息并完成任务的过程]

### Tags
**Tags:** ["tag0", "tag1", "tag2", "tag3", ...]
（例如："Central Area"、"focal point"、"famous for" 等）

### Next Subgoal
## [智能体现在应当追求的一个最佳下一子目标。]

输入：
目标：{goal}
当前子目标：{subgoal}
当前状态：{state}
当前观察：{observation}
```

**清单 10：控制多跳检索过程的提示模板。**

```text
============================================================
Prompt Multi-hop_Retrieval
============================================================
你是一个用于多跳问答的检索控制器。
只返回**严格 JSON**：
{
  "enough": true/false,
  "top_node_ids": [int, int, ...]
}

**约束：**
-- top_node_ids 长度 <= {n_facts_new_query}
-- top_node_ids 必须是可用 id 的子集
-- 如果 enough=true => top_node_ids=[]

**问题：**
{question}

**可用节点 id：**
{available_ids}

**已检索事实：**
{semantic_memory_str}
```

### C.4 推理

我们根据每种记忆类型独特的结构特征，为其设计了定制推理提示。

为契合第 3.3 节所述推理模块聚合和浓缩已检索记忆的目标，我们使用清单 12 所详述的提示结构，查询 LLM 以抽取与问题相关的信息。

由于情景记忆往往冗长且非结构化，我们观察到，信息抽取效率不高，因为大多数 LLM 并未针对该任务进行专门训练。相反，我们发现，像清单 11 中的提示一样，直接要求 LLM 对每条记忆进行推理并回答问题，能够成功利用 LLM 的推理能力，同时在推理过程中保留大部分有用信息。随后，包含推理过程在内的全部输出都作为所抽取的信息予以保留。

程序性记忆主要用于多轮决策任务。为此，我们提示 LLM 整合由相似任务中的成功记忆与失败记忆共同构成的多样经验。如清单 13 所示，该过程生成连贯、可执行的指导，为智能体的下一动作提供依据。

**表 14：多跳检索控制流程（伪代码风格）。**

```text
输入：查询 Q；图 (G^E, G^S, G^P)，其中 G^S、G^P 通过来源链接连接至 G^E；预算 K；跳数上限 T_max
输出：检索上下文 R

F ← TaskAdapter(Q)                 { F ⊆ {E,S,P} }
C ← InitLowLevel(Q,{G^S,G^P})      { 对所有低层节点打分 }
C ← TopK(C,K)
for t = 1 to T_max do
    if Stop(Q,C) then break
    q^a ← Abstract(Q,C)             { G^S 使用概念；G^P 使用意图 }
    C ← C ∪ ExpandLowLevel(q^a,{G^S,G^P})
    C ← RerankPrune(C,K)            { 例如相关性等 }
end for
if E ∈ F then
    R ← ToEpisodic(C,G^E)           { 通过来源关系映射 }
else
    R ← C
end if
return R
```

**清单 11：对检索到的情景记忆进行推理和压缩的提示模板。**

```text
============================================================
Prompt Reason_Episodic
============================================================
我会向你提供你与一名用户之间的历史聊天信息。请根据信息回答问题。
请逐步回答问题：先抽取所有相关信息，再对记忆进行推理以得到答案。

信息：
{information}

当前日期：{time}
问题：{question}
答案（逐步作答）：
```

**清单 12：对检索到的语义记忆进行推理和压缩的提示模板。**

```text
============================================================
Prompt Reason_Semantic
============================================================
我会向你提供若干条检索到的事实。抽取所有与问题相关的有用信息。
在输出的推理信息中，尽可能使用检索事实中的原始措辞，不要用同义词或近义词替换。
如果没有发现有用信息，只返回 "null"。

## 输出格式：
### Reasoning
（抽取信息的过程）

### Information
## （你抽取的有用信息）

输入：
事实：{semantic_memory}
当前日期：{time}
问题：{observation}
```

**清单 13：对检索到的程序性记忆进行推理和压缩的提示模板。**

```text
============================================================
Prompt Reason_Procedural
============================================================
将提供以下信息：
问题：用户正在询问的问题。
信息：可能与问题相关的若干条信息。

你的任务：
1. 仔细阅读用户的问题。
2. 分析每一条检索到的信息，并判断它对回答问题有多相关、多有用。
3. 根据分析，把所有有用信息整合成一段连贯内容，帮助智能体回答问题。当你认为信息不足或相互矛盾时，生成最有可能的信息。整合后的内容应当简洁、准确，并与用户的问题相关。

## 输出格式：
### Reasoning
（你对给定问题与信息进行分析时的推理。）

### Final Information
## （应向智能体提供的综合信息。）

输入：
问题：{observation}
信息：{procedural_memory}
```

### C.5 记忆图上的完整操作

**设置。** 我们在由 HotpotQA 归纳出的语义子图上开展多轮受控实验。选择 HotpotQA 有两个实际原因：（i）其证据大多是事实性、抽取式的，因此语义记忆相对稳定，对风格差异不那么敏感；（ii）所得子图足够轻量，可以快速迭代不同的合并阈值和更新策略。

给定一个语义节点，我们的更新例程首先执行候选发现：收集所有与该节点共享至少一个概念/标签的语义节点（即标签诱导投影下的邻居）。随后，计算当前节点与每个候选项之间的嵌入相似度，按相似度对候选项排序，并选出相似度超过预定义阈值 $\tau$ 的 top-$m$ 个候选项（默认使用 $m=1$，即只考虑超过阈值的最相似候选项）。若选出一对 $(s_i,s_j)$，我们通过调用 LLM 触发一次合并操作：（i）综合出一条能更好概括合并信息的新语义记忆；（ii）根据提示中编码的显式规则，决定是否停用原始节点（两者都不删除、两者都删除等）。完整提示见清单 14。

**清单 14：用于在记忆更新中检查并合并语义节点的提示模板。**

```text
============================================================
Prompt Merge_Semantic
============================================================
你将得到两条关于相关主题的记忆条目。一条来得较早（Information 1）
另一条来得较晚（Information 2）。
你的任务：
(1) 将它们合并为一条改进后的、清晰、简洁的陈述。不要编造新事实。
(2) 决定合并后是否停用（软删除）原来的两个节点。
输入：
Information 1 (Earlier Information): {memory_earlier}
Information 2 (Later Information): {memory_later}
停用决策规则（恰好选择一种情形）：
Case A: "UPDATE_SAME_FACT"
* 条件：Information 1 与 2 本质上描述同一事实/事件，且
Information 2 主要是更新/纠正/细化 Information 1 的细节。
* 动作：停用两个原始节点（较早的与较晚的），因为合并后的节点完全
取代它们。
Case B: "SAME_TOPIC_MERGE_WELL"
* 条件：Information 1 与 2 在同一主题下强相关，且
合并后的陈述读起来自然地像一份统一摘要（而不是别扭的拼接）。
* 动作：停用两个原始节点（较早的与较晚的）。
Case C: "WEAK_RELATED_STITCH_RISK"
* 条件：Information 1 与 2 仅弱相关；合并感觉像是在拼接
两个片段；并且停用任一原始节点都很可能损害未来检索。
* 动作：两个原始节点都不停用。
硬约束：
* 输出必须是有效 JSON（不要 Markdown，不要额外文本）。
* relationship 必须是上述三个标签之一。
* 若 relationship 为 Case A 或 B => deactivate_earlier=true AND deactivate_later=true。
* 若 relationship 为 Case C => deactivate_earlier=false AND deactivate_later=false。
* 若两条记忆冲突，以 Information 2 作为更新近的一方优先。
* 输出你做出该决策的简单推理。
输出必须是有效 JSON，且恰好包含这些键：
* merged_statement (string)
* relationship ("UPDATE_SAME_FACT" | "SAME_TOPIC_MERGE_WELL" | "
WEAK_RELATED_STITCH_RISK")
* deactivate_earlier (boolean)
* deactivate_later (boolean)
* simple_reasoning (string)
只返回该 JSON 对象。
```

**阈值与评估。** 我们在同一 HotpotQA 子集上评估两个合并阈值，$\tau \in \{0.6,0.7\}$。我们报告下游 QA 性能（EM/F1）以及更新前后的图统计量。结果表明性能保持在正常波动范围内，同时图变得更紧凑，候选扇出更可控。具体而言，当 $\tau=0.6$ 时，477/3413 个语义节点触发合并；当 $\tau=0.7$ 时，171/3413 个节点触发合并，反映了激进与保守之间的预期权衡。

我们比较了同一 HotpotQA 子集在两种合并阈值下语义图更新前后的下游 QA 性能。不更新时，系统达到的 EM/F1 为 61.00/74.39。在 $\tau=0.6$ 下更新时，EM/F1 变为 63.00/73.97，在 $\tau=0.7$ 下更新时，EM/F1 变为 62.00/74.65。总体而言，这些差异处于正常的运行间波动范围内，表明语义图更新/合并不会实质性地降低该子集上的端到端性能。

**图质量：紧凑性与候选可控性。** 我们沿两个轴刻画更新/合并的结构效应。

- **紧凑性。** 我们用活跃语义节点数 $N_s$、已用标签数 $N_t$，以及语义–标签二部边数 $E_{\mathrm{bip}}$（附着关系）来度量紧凑性。在更严格的阈值 $\tau=0.7$ 下，更新后的图将活跃语义节点从 3413 减至 3242（−5.0%），已用标签从 12501 减至 11812（−5.5%），二部边从 23230 减至 20604（−11.3%），表明语义记忆及其标签附着中的冗余更低。我们还报告了标签诱导语义共现对的未去重上界，$\sum_t \binom{\deg(t)}{2}$；该值从 78279 降至 57581（−26.5%）。这说明更新/合并有效剪除了冗余的共同附着关系，否则它们会造成稠密却缺乏信息量的连通结构。

- **候选可控性。** 我们的检索/更新流水线使用标签诱导的候选生成：对一个语义节点，我们取其标签所附着的语义 ID 的并集作为候选集。为量化候选扇出的可控程度，我们对活跃语义节点进行采样，并计算其候选集大小（经标签可达的唯一邻居，排除自身）。当 $\tau=0.7$ 时，候选集大小显著变小，因而更可控：采样均值从 38.36 降至 31.28（−18.5%），采样中位数从 18.0 降至 14.0（−22.2%）。直观上，通过合并近重复的语义条目并停用冗余的原始节点，该图减少了经由共享标签的不必要扩展，从而在不降低下游 EM/F1 的情况下改善计算稳定性。

总体而言，这些结果表明语义更新/合并能够在保持检索支持的同时改善图的紧凑性并减少冗余，两种阈值下稳定的 HotpotQA 准确率即为例证。

## 附录 D 记忆效率的信息论评估

我们提出一个框架来量化所生成记忆的效用。我们将记忆效率分解为两个维度：准确性（逐点）与确定性（分布性），并对二者的联系给出统一解释。

### D.0.1 预备知识与记号

我们定义运行环境与智能体的决策过程。

- **状态空间 (S)：** 所有可能的环境状态或上下文的集合。令 $s \in S$ 表示当前状态。
- **动作空间 (A)：** 所有可能动作的集合。令 $a \in A$ 表示智能体采取的一个动作。
- **最优动作（$a^{*}$）：** 令 $a^{*} \in A$ 表示在给定当前状态 $s$ 下的真实最优动作（或由示范者/Oracle 选择的实际动作）。
- **记忆空间 (M)：** 记忆生成模块的输出空间。令 $m \in M$ 表示一条生成的记忆序列。此处，$m$ 不必是原始语料的子序列，而可以是为增强智能体在当前状态 $s$ 下的决策而优化的构造表示（例如抽象、推断或摘要）。
- **记忆生成器（$G_{\phi}$）：** 一个参数化函数（例如搜索引擎或 LLM），将当前状态 $s$ 与原始记忆库 $K$ 映射为记忆产物。我们将该过程定义为 $m=G_{\phi}(s,K)$。该函数封装检索、推理等操作，以显化隐含逻辑。
- **记忆长度 (|m|)：** 令 $|m|$ 表示记忆序列 $m$ 以 token 计的长度。

### D.1 逐点度量：准确性与密度

#### D.1.1 通过逐点互信息（PMI）量化信息增益

我们将智能体的决策建模为动作上的条件概率分布，并区分智能体在可访问与不可访问记忆模块时的策略。

- **基线策略（先验）：** 在仅给定当前状态 $s$、无记忆增强的情况下，选择最优动作 $a^{*}$ 的概率：

$$P_{\mathrm{base}}(a^{*} | s)$$

- **记忆增强策略（后验）：** 在给定当前状态 $s$ 与生成记忆 $m$ 的情况下，选择最优动作 $a^{*}$ 的概率：

$$P_{\mathrm{mem}}(a^{*} | s,m)$$

为量化所生成记忆 $m$ 对选择正确动作 $a^{*}$ 的具体贡献，我们采用逐点互信息 (PMI)。

我们将决策信息增益定义为：

$$\mathrm{PMI}(a^{*};m | s) = \log_2 \frac{P_{\mathrm{mem}}(a^{*} | s,m)}{P_{\mathrm{base}}(a^{*} | s)}.$$

**解释。**

- 若 PMI > 0：记忆 $m$ 提供了正信息，提高了选择正确动作 $a^{*}$ 的概率。
- 若 PMI = 0：记忆 $m$ 与该决策无关。
- 若 PMI < 0：记忆 $m$ 具有误导性，降低了选择 $a^{*}$ 的概率。

#### D.1.2 记忆信息密度（每 token 的逐点增益）

我们将总信息增益按处理记忆的成本（其 token 长度）归一化。记忆信息密度定义为：

$$\rho(a^{*},m) = \frac{\mathrm{PMI}(a^{*};m | s)}{|m|}.$$

代入 PMI 定义：

$$\rho(a^{*},m) = \frac{1}{|m|} \cdot \log_2 \left( \frac{P_{\mathrm{mem}}(a^{*} | s,m)}{P_{\mathrm{base}}(a^{*} | s)} \right).$$

**单位。** 每 Token 比特（bits/token）。

#### D.1.3 该度量的小结

该框架不仅从有效性（它有帮助吗？）评估记忆模块，也从简洁性（它是否高效地提供了帮助？）评估。

- **目标：** 最大化 $\rho(a^{*},m)$。
- **优化目标：** 生成能最大化 $a^{*}$ 的似然、同时最小化 token 数 $|m|$ 的记忆内容。

#### D.1.4 聚合评估：全局记忆效率

为评估记忆模块在数据集

$$
D=\{(s_i,a_i^*,m_i)\}_{i=1}^{N}
$$

上的性能，其中包含 $N$ 个决策实例，我们计算全局记忆信息密度，记为 $\rho_{\mathrm{global}}$。

与各实例效率的算术平均值不同（对较短记忆序列而言，后者在数值上可能不稳定），$\rho_{\mathrm{global}}$ 表示整个评估集上每消耗一个 token 所获得的摊销信息增益。它回答的问题是：“系统每生成一个记忆 token，平均能获得多少 bit 与最优动作相关的决策信息？”

我们将 $\rho_{\mathrm{global}}$ 定义为累计逐点互信息总量与累计记忆长度总量之比：

$$
\rho_{\mathrm{global}}
=\frac{\sum_{i=1}^{N}\operatorname{PMI}(a_i^*;m_i\mid s_i)}
{\sum_{i=1}^{N}|m_i|}。
$$

展开 PMI 项，可计算形式为：

$$
\rho_{\mathrm{global}}
=\frac{
\sum_{i=1}^{N}\log_2\left(
\frac{P_{\mathrm{mem}}(a_i^*\mid s_i,m_i)}
{P_{\mathrm{base}}(a_i^*\mid s_i)}
\right)}
{\sum_{i=1}^{N}|m_i|}。
$$

**该指标的性质**

1. **Token 层加权：** 通过在分母中对长度求和，该指标隐式地赋予较长记忆序列更大的权重，从而确保成本高昂的生成必须以成比例的更高信息增益来证明其成本合理。
2. **稳健性：** 它对“小分母问题”具有稳健性；否则，一个记忆上下文极短（例如 $|m_i|=1$）的实例可能产生虚高的效率分数，进而扭曲数据集平均值。
3. **系统层解释：** $\rho_{\mathrm{global}}=0.5$ 表示，从总体上看，系统需要两个记忆上下文 token，才能获得关于最优动作的 1 bit 信息。

#### D.1.5 控制与筛选：界定评估范围

为确保该指标刻画的是记忆模块的边际效用，而非任务本身的难度，我们引入筛选机制。该机制隔离出智能体有望从外部信息中获益的实例。

**冗余筛选器（高先验置信度）。** 我们排除基线策略对最优动作已有较高置信度的实例，因为在这些情况下生成记忆在功能上是冗余的。

我们把主动评估子集记为 $D_{\mathrm{active}}\subseteq D$，定义为：

$$
D_{\mathrm{active}}
=\{(s_i,a_i^*,m_i)\in D\mid P_{\mathrm{base}}(a_i^*\mid s_i)<\tau_{\mathrm{conf}}\}，
$$

其中，$\tau_{\mathrm{conf}}$ 是预定义的置信度阈值（例如 $\tau_{\mathrm{conf}}=0.8$ 或 $0.9$）。

**理由。** 若 $P_{\mathrm{base}}\geq\tau_{\mathrm{conf}}$，则智能体已经“知道”答案。任何生成的记忆 $m$ 即使相关，也只会产生可忽略的信息增益（$\operatorname{PMI}\rightarrow0$）。纳入这些实例会不公平地降低 $\rho_{\mathrm{global}}$。

**生成空记忆的情况。** 我们必须考虑记忆模块决定不生成任何信息，或返回空序列（$|m|=0$）的情况。

- **处理方式：** 若 $|m_i|=0$，则从密度计算中排除该实例，因为分母为零。
- **说明：** 这些样本虽被排除在密度指标（效率）之外，但仍应单独追踪其召回率（有效性）。

**精炼后的全局指标。** 将“总和之比”方法与“冗余筛选器”结合，最终的操作性指标只在主动子集上计算：

$$
\rho_{\mathrm{final}}
=\frac{\sum_{i\in D_{\mathrm{active}}}\operatorname{PMI}(a_i^*;m_i\mid s_i)}
{\sum_{i\in D_{\mathrm{active}}}|m_i|}。
$$

这一精炼形式确保我们衡量的是必要记忆的效率，从而更纯净地反映模块对复杂推理的贡献。

### D.2 几何分析：效用-成本景观

为理解效率增益背后的机制，我们从几何视角分析记忆模块。构建如下效用-成本坐标系：

- **横轴（成本）：** 记忆长度，$L=|m|$。
- **纵轴（效用）：** 信息增益，$I(L)=\operatorname{PMI}(a^*;m\mid s)$。

标准直觉可能认为，效用随上下文长度单调地以对数形式增长。然而，在包含噪声的真实智能体场景中，我们提出，该曲线遵循单峰（有峰值）分布。

#### D.2.1 峰值现象与噪声毒性

我们并非仅将有效信息增益 $I(L)$ 建模为信号累积，而是将其建模为信号抽取与注意力稀释的叠加：

$$
I(L)=S(L)-\eta(L)，
$$

其中，$S(L)$ 为对数信号累积函数（单调递增），$\eta(L)$ 表示噪声毒性或认知负荷惩罚。

我们在曲线上识别出三个定性区域：

1. **欠拟合区域（$dI/dL>0, d^2I/dL^2>0$）：** 边际效用高；关键语义信息正在生成。
2. **甜点区（$dI/dL\approx0$）：** 峰值 $I_{\max}$；信号相对于噪声达到最大。
3. **毒性区域（$dI/dL<0$）：** 边际效用为负的区域。延长记忆长度 $L$ 所引入的无关 token（HTML 标签、题外信息）多于有效信号，从而稀释智能体的注意力机制。因此，尽管拥有“更多上下文”，正确动作的概率 $P(a^*)$ 反而下降。

#### D.2.2 效率（$\rho$）的几何解释

在该坐标系中，核心指标——全局记忆密度 $\rho$——具有精确的几何意义：

$$
\rho(L)=\frac{I(L)}{L}，
$$

即从原点 $(0,0)$ 到运行点 $(L,I(L))$ 的割线斜率。

由此可见两项最优性标准之间的权衡：

- **点 A：最大性能点。** 曲线峰值，此处 $dI/dL=0$。这是与成本无关的性能上限。
- **点 B：最大效率点。** 从原点出发的割线恰好成为曲线切线的点。几何上，该点使斜率 $\rho=I(L)/L$ 最大。

**洞见：效率-性能差距。** 通常，点 B 位于点 A 左侧。追求最高性能（从 B 向 A 移动）需要不成比例地增加 token，并产生递减、甚至为负的回报。

**“顿悟”阈值（点 C）。** 从理论上说，识别位于点 B 左侧的拐点（$I''(L)=0$）也具有启发意义。在经济学意义上，它代表最大边际效用点（$\max dI/dL$）。在记忆生成语境中，点 C 对应于获得“关键语义质量”——即首次实现上下文突破的特定 token 或短语（例如检索到正确的实体名称）。然而，在点 C 停止通常并非最优（欠拟合），因为智能体虽已获得关键信号，却缺少稳健执行决策所必需的上下文（在 C 与 B 之间积累）。

#### D.2.3 增益的向量分解

为量化所提出记忆模块相对于其他基线的改进，我们在 $L$-$I$ 平面中执行向量分解。

令基线向量为

$$
\mathbf v_{\mathrm{baseline}}=(L_{\mathrm{baseline}},I_{\mathrm{baseline}})，
$$

改进方法为

$$
\mathbf v_{\mathrm{ours}}=(L_{\mathrm{opt}},I_{\mathrm{opt}})。
$$

转换向量为：

$$
\Delta\mathbf v=\mathbf v_{\mathrm{ours}}-\mathbf v_{\mathrm{baseline}}=(\Delta L,\Delta I)。
$$

我们将改进方向分类如下：

- **水平移动（$\Delta L<0, \Delta I\approx0$）：纯压缩。** 增益完全来自成本降低。
- **垂直移动（$\Delta L\approx0, \Delta I>0$）：纯增强。** 在预算相同的情况下获得更好的推理。
- **向西北移动（$\Delta L<0, \Delta I>0$）：混合增益。** 这表明基线运行在毒性区域。通过精炼记忆，我们移除了损害决策的噪声，从而同时降低成本、提高准确率。

#### D.2.4 帕累托支配

我们认为，所提出的模块并非只是在朴素基线的同一条权衡曲线上移动。相反，它会使帕累托前沿发生移动。

令朴素基线曲线为 $C_{\mathrm{base}}$，我们的方法曲线为 $C_{\mathrm{ours}}$。我们声称，在相关定义域内：

$$
\forall L,quad I_{\mathrm{ours}}(L)\geq I_{\mathrm{base}}(L)。
$$

这意味着，我们的方法提高了记忆系统的理论上限，使智能体能够处理以往由于 $C_{\mathrm{base}}$ 的噪声约束而不可行的更复杂任务（所需 $I$ 更高）。

**备注：基线不变性与坐标偏移。** 值得注意的是，对于固定智能体，朴素基线置信度 $\sum\log P_{\mathrm{base}}(a^*\mid s)$ 为常数，因此效用轴 $I(L)$ 在函数上只是原始后验对数似然 $\sum\log P_{\mathrm{mem}}$ 的垂直平移。尽管该常数偏移会移动原点 $(0,0)$ 的绝对位置，从而缩放密度斜率 $\rho$ 的绝对值，但它保留了景观的拓扑特征。因此，方法之间的相对比较（例如是否存在“毒性下降”，或“甜点区”的位置）不随基线性能改变。

### D.3 分布式指标：确定性与校准

PMI 衡量记忆相对于真实答案 $a^*$ 的准确性，却无法刻画记忆对智能体不确定性的整体影响。一段记忆序列可能略微提高 $P(a^*)$（正 PMI），同时仍让智能体在许多其他次优动作之间犹豫不决。

为了衡量记忆剪枝搜索空间并锐化决策边界的能力，我们将该框架扩展到评估分布信息密度。

#### D.3.1 通过不确定性降低量化动作空间压缩

我们采用香农熵来量化智能体策略中固有的不确定性（或「认知负荷」）。

- 先验不确定性（$H_{\text{base}}$）：仅给定状态 $s$ 时智能体的不确定性。

$$H_{\text{base}}(s) = -\sum_{a \in A} P_{\text{base}}(a \mid s)\log_2 P_{\text{base}}(a \mid s)$$

- 后验不确定性（$H_{\text{mem}}$）：处理记忆 $m$ 之后的不确定性。

$$H_{\text{mem}}(s,m) = -\sum_{a \in A} P_{\text{mem}}(a \mid s,m)\log_2 P_{\text{mem}}(a \mid s,m)$$

动作空间压缩（或不确定性降低），记为 $\Delta H$，表示记忆为消解歧义所贡献的信息量（以比特计）：

$$\Delta H(m \mid s) = H_{\text{base}}(s) - H_{\text{mem}}(s,m).$$

**解释。**

- $\Delta H > 0$：锐化。记忆减小了搜索空间的有效规模（剪枝无效选项）。
- $\Delta H < 0$：混淆。记忆引入了冲突信息，使分布变平并增加了不确定性。

#### D.3.2 分布信息密度（每 token 的分布增益）

与逐点度量类似，我们将分布信息密度 $\rho_{\text{dist}}$ 定义为每单位处理成本的不确定性降低速率：

$$\rho_{\text{dist}}(m) = \frac{\Delta H(m \mid s)}{|m|} = \frac{H_{\text{base}}(s) - H_{\text{mem}}(s,m)}{|m|}.$$

**单位。** 每 token 移除的不确定性比特（bits/token）。

**安全性分析：置信度–有效性象限** 为严格评估记忆，我们将每个实例投影到由以下轴定义的二维平面上：

- X 轴（确定性）：动作空间压缩，$\Delta H(m \mid s)$。
- Y 轴（准确性）：决策信息增益，$\operatorname{PMI}(a^{*};m \mid s)$。

该投影将记忆交互归入以下四种机制之一：

1. 象限 I：高效推理（$\Delta H > 0$，$\operatorname{PMI} > 0$）
「更锐且正确。」
记忆确认了正确动作并排除了干扰项。智能体从不确定性走向了正确的确定性。这是理想的运行状态。
2. 象限 II：纠正性校准（$\Delta H < 0$，$\operatorname{PMI} > 0$）
「打破虚假置信。」
此处，智能体很可能一开始对错误动作具有高置信度（低先验熵）。记忆引入了必要的怀疑，使分布变平，但提高了真正最优动作 $a^{*}$ 的概率。
**意义：** 这表示一种「救援」机制，记忆在此修正智能体的内部错误观念。
3. 象限 IV：幻觉陷阱（$\Delta H > 0$，$\operatorname{PMI} < 0$）
「自信但错误。」
记忆降低了不确定性，但指向了偏离真值的方向。智能体变得「教条式地错误」。
**风险：** 这是「有毒确定性」，是检索增强生成（RAG）中最危险的失败模式。
4. 象限 III：破坏性噪声（$\Delta H < 0$，$\operatorname{PMI} < 0$）
「困惑且被误导。」
记忆不仅未能指向正确答案，还增加了整体困惑（熵），实际上起到了干扰作用。

#### D.3.3 有效性调整的分布信息密度

为将象限分析综合为单一标量度量——在奖励高效推理的同时惩罚幻觉——我们提出有效性调整的分布信息密度（$\rho_{\Phi}$）。该度量将分布偏移的幅度与准确性增益的方向性相结合。

**度量定义** 我们将 $\rho_{\Phi}$ 定义为有效性符号与归一化分布功的乘积：

$$\rho_{\Phi}(m) = \underbrace{\operatorname{sgn}(\operatorname{PMI}(a^{*};m \mid s))}_{\text{方向有效性}} \cdot \frac{|\Delta H(m \mid s)|}{|m|}$$

其中：

- $\operatorname{sgn}(\cdot)$ 是符号函数（改进为 $+1$，损害为 $-1$）。
- $|\Delta H(m \mid s)|$ 是不确定性变化的绝对幅度（比特）。
- $|m|$ 是记忆长度（tokens）。

**性质与解释** 该表述在全部四种认知机制上提供了统一评估：

1. 对效率的奖励（象限 I）：
当智能体对正确答案变得更加确定时（$\Delta H > 0$，$\operatorname{PMI} > 0$），该度为正。更高的密度表明更快收敛到真相。
2. 对纠正的奖励（象限 II）：
当记忆纠正了一个自信的错误时（$\Delta H < 0$，$\operatorname{PMI} > 0$），该度保持为正。尽管熵增加（智能体变得不那么教条），项 $|\Delta H|$ 捕捉了为打破虚假置信所执行的显著「认知功」，而 $\operatorname{sgn}(\operatorname{PMI})$ 将该偏移验证为有益。
3. 对幻觉的惩罚（象限 IV）：
当智能体对错误答案变得自信时（$\Delta H > 0$，$\operatorname{PMI} < 0$），该度变为负。幻觉越「令人信服」（更高的 $\Delta H$），惩罚严重。这起到软安全约束的作用。
4. 对破坏性噪声的惩罚（象限 III）：
当记忆使智能体困惑并降低准确性时（$\Delta H < 0$，$\operatorname{PMI} < 0$），该度为负。此处，$|\Delta H|$ 表示所引入困惑的幅度。由于记忆未能支持正确动作，这种「负功」受到惩罚，反映了处理干扰信息的成本。

**小结。** 度量 $\rho_{\Phi}$ 回答的是：「记忆模块在每 token 的成本下提供了多少比特的有效分布重塑（熵偏移）？」

## D.4 理论统一：Oracle–散度原理

至此，我们已通过特定度量分析了记忆效率：准确性（PMI）与确定性（熵）。在本节中，我们将这些度量置于更广阔的信息论框架之中。我们提出，记忆模块的根本目标是最小化智能体策略与理想策略之间的信息几何距离。

我们称之为神谕–散度原理。

#### D.4.1 一般目标：散度降低

令 $Q$ 表示神谕分布（或「上帝视角」），代表当前状态 $s$ 的理想策略。记忆生成的目标是将智能体的先验信念 $P_{\text{base}}$ 转化为在统计上更接近 $Q$ 的后验 $P_{\text{mem}}$。

我们用 Kullback-Leibler（KL）散度的降低来量化这一改进。广义神谕信息增益 $\Delta_{\text{div}}$ 定义为：

$$\Delta_{\text{div}}(m) = D_{\mathrm{KL}}(Q \parallel P_{\text{base}}) - D_{\mathrm{KL}}(Q \parallel P_{\text{mem}}).$$

由于对给定实例而言神谕分布 $Q$ 与基线先验 $P_{\text{base}}$ 是固定的，最大化该增益等价于最小化后验的散度：

$$\operatorname*{argmax}_{m} \Delta_{\text{div}}(m) \equiv \operatorname*{argmin}_{m} D_{\mathrm{KL}}(Q \parallel P_{\text{mem}}).$$

理想情况下，若记忆是完美的，$P_{\text{mem}}$ 收敛到 $Q$，散度变为零。

#### D.4.2 实际实例化：作为特例的 PMI

尽管神谕 $Q$ 理论上可以建模软标签或多峰分布，但在绝大多数离散智能体任务中（例如工具选择、多步推理），目标按规范由单一唯一真值定义。在这一标准确定性设定下，神谕分布从一般概率向量坍缩为狄拉克 $\delta$（独热）分布：

$$Q(a) = \begin{cases} 1 & \text{if } a = a^{*} \\ 0 & \text{otherwise} \end{cases}$$

在这一特定的独热假设下，KL 散度项显著简化：

$$D_{\mathrm{KL}}(Q \parallel P) = \sum_{a \in A} Q(a)\log_2 \frac{Q(a)}{P(a)} = 1 \cdot \log_2 \frac{1}{P(a^{*})} = -\log_2 P(a^{*}).$$

将其代回广义增益方程得到：

$$\begin{aligned}
\Delta_{\text{div}}(m) &= [-\log_2 P_{\text{base}}(a^{*})] - [-\log_2 P_{\text{mem}}(a^{*} \mid m)] \\
&= \log_2 P_{\text{mem}}(a^{*} \mid m) - \log_2 P_{\text{base}}(a^{*}) \\
&= \operatorname{PMI}(a^{*};m).
\end{aligned}$$

**含义：可处理性与充分性。** 该推导表明，逐点互信息（PMI）不仅仅是一种启发式，而是广义散度降低在确定性假设下的代数坍缩。这将 PMI 确立为最优工程度量，因为它：

- 计算上可处理：它只需要跟踪真值的概率 $P(a^{*})$，避免了建模完整分布距离的计算成本。

- **理论充分性：** 在独热设定下，最大化 PMI 与最小化 KL 散度在数学上同构。

因此，PMI 是一项代理指标，有效地连接了抽象信息几何与实际的低成本评估。

#### D.4.3 熵推论

这一广义视角也解释了熵减（$\Delta H$）的作用。
由于 Oracle 分布 $Q$（独热分布）的熵为零（$H(Q) = 0$），任何成功最小化散度 $D_{\mathrm{KL}}(Q \parallel P_{\mathrm{mem}})$ 的策略 $P_{\mathrm{mem}}$ 都必然降低其自身的熵。
$P_{\mathrm{mem}} \rightarrow Q \implies H(P_{\mathrm{mem}}) \rightarrow 0$。
因此，准确率（PMI）与确定性（熵）并非相互独立的目标。它们是同一优化过程的耦合特征：
- PMI 衡量概率质量与 $Q$ 的峰值之间的对齐程度。
- 熵衡量分布的紧致性，这是逼近 $Q$ 的前提。

#### D.4.4 小结

在这一统一视角下，我们的框架对理论轨迹 $\Delta_{\mathrm{div}}$ 提供了双层近似：
1. **准确率增益效率代理（$\rho$）：** 在假定标准独热 Oracle 的条件下，点互信息 $\rho$ 作为主要计算指标。它能以较低成本精确衡量智能体向最优动作 $a^{*}$ 的输运程度，因此足以用于大规模性能基准测试。
2. **经验证的确定性增益代理（$\rho_{\Phi}$）：** 分布密度 $\rho_{\Phi}$ 作为诊断性补充，近似 Oracle 散度对熵最小化的要求，并显式防范“脱靶”置信度，即智能体朝错误分布 $Q' \neq Q$ 最小化熵。
因此，未来的记忆研究与提示设计工作可用 $\rho$ 衡量额外 token（例如记忆 token）对决策的帮助程度，并用 $\rho_{\Phi}$ 诊断这种帮助有多确定。

## 附录 E 补充实验细节

### E.1 LongMemEval

**设置。** 为评估智能体基于历史用户–智能体对话回答问题的能力，我们使用 LongMemEval（Wu et al., 2024）。具体地，我们采用 LongMemEvalS 子集，该子集的每个测试用例具有 115K token 的对话上下文，与智能体记忆研究中的既定标准一致（Li et al., 2025；Rasmussen et al., 2025）。对于 PlugMem 框架，我们采用 Qwen2.5-32B-Instruct 进行结构化模块与检索模块的推理，并采用 Qwen2.5-72B-Instruct 作为推理模块。所有基线方法由 GPT-4o 驱动。NV-Embed-v2 用于生成嵌入，且对所有方法使用 Qwen2.5-72B-Instruct 作为基于所生成记忆来回答问题的基座智能体。在检索阶段，我们设置 $k=10$，提取最相关的前 10 个记忆节点供下游推理使用。

**基线。** 我们考虑三类基线：i）朴素方法，其提示中不包含历史对话，仅依赖骨干模型的参数化知识作答，或直接回答“我不知道”；ii）任务无关方法，即并非专门为智能体–用户对话问答基准设计的方法，例如在轮次级执行基于嵌入检索的标准稠密 RAG 流水线，以及 A-Mem（Xu et al., 2025）等智能体记忆方法；A-Mem 从情景记忆中生成笔记，并通过基于图的组织机制维护和更新笔记；iii）任务特定方法，即专为基于历史对话的问答定制的方法，包括基于知识图谱的 Zep（Rasmussen et al., 2025）和面向认知科学的结构化检索方法 LiCoMemory（Huang et al., 2026）。对于 Zep，我们采用其论文报告的结果。对于 A-Mem 与 LiCoMemory，我们随机打乱 LongMemEval 原始文件 `longmemeval_s_cleaned.json` 中的测试用例，并在相同的前 100 个用例上评估它们。

**指标。** 我们报告准确率（表 3 中的 Acc.），该指标基于 LongMemEval（Wu et al., 2024）的 LLM-as-a-Judge 方法评估，使用 GPT-4o 作为评估器。此外，我们还报告依赖于第 4.1 节中引入的信息论度量的全局密度。具体地，我们使用逐实例的答案重叠来实例化式 (1) 中的似然项：$P_{\mathrm{mem}}$ 与 $P_{\mathrm{base}}$ 在评估器判定答案正确时设为 1.0，否则设为 0.0。为避免在 $P_{\mathrm{mem}}$ 或 $P_{\mathrm{base}}$ 等于 0.0 时计算 $\log(P_{\mathrm{mem}}/P_{\mathrm{base}})$ 出现数学错误，我们在计算 PMI 时应用加性平滑并使用
$$\log\left(\frac{P_{\mathrm{mem}}+\epsilon}{P_{\mathrm{base}}+\epsilon}\right)$$
其中 $\epsilon$ 为无记忆智能体在 LongMemEval 上成功率的 1%。

**主要结果。** 实验结果汇总于表 3。PlugMem 在 LongMemEval 上优于所有基线，并取得最高的全局信息增益密度。如表 15 所示，相较若干任务特定方法，PlugMem 的性能也很有竞争力。值得注意的是，PlugMem 在多会话子集上表现最佳。该子集尤其困难，因为它同时要求准确检索和精确抽取记忆；智能体必须检索多条“黄金”记忆并加以区分，才能准确计数。这说明 PlugMem 的架构成功弥合了大规模数据检索与精确认知抽取之间的鸿沟，也证明集成外部记忆模块能够显著增强智能体在动态、多轮环境中的长期一致性。

**表 15：LongMemEval 上的子集性能。**（S-S-U：单会话-用户，S-S-A：单会话-助手，S-S-P：单会话-偏好，K-U：知识更新，T-R：时序推理，M-S：多会话）

| 方法 | S-S-U | S-S-A | S-S-P | K-U | T-R | M-S | 平均值 |
|---|---|---|---|---|---|---|---|
| Zep | 92.9 | 80.4 | 56.7 | 83.3 | 62.4 | 57.9 | 71.2 |
| LiCoMemory | 92.9 | 90.9 | 50.0 | 81.2 | 65.4 | 63.0 | 73.0 |
| PlugMem | 94.3 | 98.2 | 60.0 | 79.5 | 66.2 | 64.7 | 75.1 |

**消融。** 我们进一步对 PlugMem 的三个组件进行消融研究。在“无结构化”设置中，我们以轮次级简单分块替代对原始用户–智能体对话的结构化索引。在“无检索”设置中，我们直接拼接结构化模块抽取的全部语义记忆，不经检索便送入推理模块。如表 6 所示，移除任一模块都会使任务性能与全局信息增益密度持续下降，凸显各模块对有效利用记忆的重要性。具体而言，移除检索模块会得到最差性能，说明有效利用记忆必须以检索为基础；移除结构化模块会使性能降至与朴素检索相当的水平；移除推理模块虽只使准确率略微下降，却会大幅增加输入记忆 token 数。

**Token 成本分析。** 表 16 报告了不同方法在 LongMemEval 上每个样本的 token 消耗。由于 Zep 仅部分开源，我们使用 Graphiti（Zep 所基于的开源图模块）随机评估若干样本，以估计其 token 用量。

可以预期 PlugMem 会产生相对更高的 token 计数，因为它超越了语义记忆抽取，还额外标准化情景记忆并抽取程序性记忆。这些步骤对于任务无关的记忆组织至关重要，并能够在多样化任务上实现更好的泛化，而这是仅抽取语义记忆的任务特定方法（例如 LiCoMemory）难以轻易实现的。

重要的是，尽管 PlugMem 的 token 消耗与竞争方法处于相似的数量级，实际部署成本却有显著差异。PlugMem 在推理时完全依赖开源模型，可以离线执行或以显著更低的每 token 成本运行。相比之下，若干竞争方法依赖闭源模型（如 GPT-4o）进行推理，其每 token 定价要高得多。因此，当考虑模型定价而非仅仅 token 计数时，尽管 token 用量相当，PlugMem 在实践中预期会显著更具成本效率。

**表 16：LongMemEval 上的 Token 成本统计。**（每个样本 $\tilde{k}$ tokens）

| 方法 | NVE | Q32 输入/输出 | Q72 输入/输出 | 4o 输入/输出 |
|---|---|---|---|---|
| **任务无关** |  |  |  |  |
| Vanilla RAG | 107 | - | - | - |
| A-Mem | 332 | - | - | 786/177 |
| **任务特定** |  |  |  |  |
| Zep（Graphiti） | 194 | - | - | 2545/1189 |
| LiCoMemory | 75 | - | - | 585/217 |
| **本文方法** |  |  |  |  |
| PlugMem | 197 | 1604/418 | 9/0.4 | - |

### E.2 HotpotQA

**设置。** HotpotQA（Yang et al., 2018）是一个多跳问答基准，被广泛用于评估 RAG 系统与智能体记忆框架中的多步检索与推理。遵循 HippoRAG2（Gutiérrez et al., 2025）的评估协议，我们使用其包含 1,000 个样例的预处理子集。对所有方法，我们采用 Qwen2.5-32B-Instruct（Qwen et al., 2025）作为骨干 LLM，并采用 NV-Embed-v2（Lee et al., 2025）作为嵌入模型；解码参数为 `max-tokens = 2048`、`temperature = 0.0`、`top-p = 1.0`。检索时，我们设置 top-$k=10$，返回最相关的 10 个记忆节点供下游推理使用。

**情景记忆标准化。** PlugMem 将智能体轨迹标准化为受 RL 启发的情景元组 $\langle o_t,a_t,s_t,r_t,i_t\rangle$（观察、动作、状态、奖励、意图/子目标）。对于 HotpotQA，语料库是被动索引的，而非由执行中的智能体生成。因此，我们将每个语料文本单元视为单步“轨迹”（即 $T=1$）：每个情景项包含一个观察 $o_1$（正在被索引的文本单元），而与动作相关的字段并非由执行实例化。具体而言，我们将 $a_1$、$s_1$、$r_1$ 和 $i_1$ 设为空值（或 N/A）占位符，以保持统一的元组接口。语义抽取与检索主要作用于观察内容。

**HotpotQA 多跳检索控制。** 在每一跳 $t$，检索器通过双通道更新收集候选池：i）经由抽象节点进行基于链接的扩展：检索器从当前查询 $Q_t$ 推断一小组抽象概念，将其与记忆图中的高层概念节点匹配，再经由成员关系边扩展到相邻的低层命题节点；ii）直接依据查询 $Q_t$ 进行基于嵌入的检索。

随后，我们调用 LLM 控制器，评估当前检索到的候选是否足以回答问题。若控制器返回 `enough=true`，则提前终止，并将当前事实集传给下游问答模型；否则，控制器选择一小组最有希望的候选来驱动下一跳。在实验中，我们将该子集限制为 top-2 候选。为了形成下一跳查询，我们将所选候选与前一查询整合。该整合可由基于 LLM 的重写器完成；但我们发现，简单拼接查询与所选候选在实践中表现相当，因此出于效率考虑，默认采用拼接。

**基线。** 我们考虑三类基线：i）朴素方法，包括仅依赖骨干模型参数化知识的无上下文推理，以及向模型提供黄金支持上下文、作为近似上界的 Oracle（gold-context）设置；ii）任务无关方法，即并非专为知识密集型问答基准设计的方法，例如标准稠密 RAG 流水线——它将输入视为一维文本流，按固定块大小和重叠窗口切分，并执行基于嵌入的检索——以及 A-Mem（Xu et al., 2025）等智能体记忆方法；A-Mem 将文本流结构化为笔记，并通过基于图的组织机制维护和更新笔记；iii）任务特定方法，即专为知识密集型问答和多跳检索定制的方法，包括层次化/树结构检索框架 RAPTOR（Sarthi et al., 2024），以及面向图的 GraphRAG（Edge et al., 2025）、HippoRAG2（Gutiérrez et al., 2025）和 PropRAG（Wang & Han, 2025）。

**指标。** 我们报告模型生成答案与参考答案之间的 token 级 Exact Match 与 F1 分数（表 4 中的 EM 与 F1）。除 EM 与 F1 等标准端任务指标外，我们还采用第 4.1 节与附录 D 中引入的信息论度量，以量化记忆模块带来的信息增益。对于来自基准的一个样本，点互信息（PMI）计算如下：

$$\mathrm{PMI}(a^{*};m \mid s) = \log_{2}\frac{P_{\mathrm{mem}}(a^{*}\mid s,m)}{P_{\mathrm{base}}(a^{*}\mid s)}\tag{4}$$
在一个数据集上，我们通过求和比报告一个全局的、摊销的密度：
$$\rho_{\mathrm{global}}=\frac{\sum_{i}\mathrm{PMI}(a^{*}_{i};m_{i}\mid s_{i})}{\sum_{i}|m_{i}|}\tag{5}$$
具体而言，我们使用逐实例的答案重叠来实例化式 (4) 中的似然项：$P_{\mathrm{mem}}$ 根据预测答案与参考答案之间的 F1 分数计算；类似地，$P_{\mathrm{base}}$ 中的预测答案来自不带记忆模块的基础智能体。按照求和比聚合方式，我们将各实例的 PMI 之和除以记忆 token 数之和，得到全局密度。
值得注意的是，对于某些实例，$P_{\mathrm{base}}$ 或 $P_{\mathrm{mem}}$ 可能为零，这会使式 (1) 失去定义（除以零和/或 $\log 0$）。为了稳定计算，我们采用加性平滑，在计算 PMI 时使用 $\log\left(\frac{P_{\mathrm{mem}}+\epsilon}{P_{\mathrm{base}}+\epsilon}\right)$。我们发现，当 $P_{\mathrm{base}}=0$、$P_{\mathrm{mem}}>0$ 时，过小的 $\epsilon$ 会产生异常大的 PMI，从而对聚合密度造成不成比例的影响。因此，在实现中，我们把 $\epsilon$ 设为不带记忆的基础智能体 F1 分数的 1%，并在分子和分母中使用相同的 $\epsilon$。
**消融实验。** 我们进一步对 PlugMem 的三个组件进行消融研究。在“无结构化”设置中，我们以基于文本块的流水线（固定块大小和重叠范围）替代结构化索引，用于语料预处理和评估；在“无检索”设置中，我们用随机抽取的结构化记忆条目填充检索候选集。如表 7 所示，移除任一组件都会持续降低任务性能和全局信息增益密度。尤其是，移除检索导致最显著的性能下降，而移除推理模块会显著增加注入上下文的记忆 token 数，从而降低每 token 的信息增益。

**Token 成本分析。** 此外，我们在 HotpotQA 子集上分析 PlugMem 的 token 成本，记录记忆构建（索引），以及评估时检索与推理期间的 token 用量，并与竞争基线比较。
如表 17 所示，PlugMem 的 token 用量与其他强基线相当，处于同一数量级。值得注意的是，尽管 PlugMem 提供了更丰富的记忆表示，它却不是所比较方法中 token 用量最大的方法。PlugMem 执行更细粒度、更具结构的记忆构建过程，使下游阶段能够统一开展记忆编辑、检索和推理。尤其是，PlugMem 在一条流水线中联合构建情景、语义和程序性三类互补记忆，而大多数基线只单独抽取其中一类或两类。
尽管 HotpotQA 主要评估语义知识，但情景/语义/程序性（E/S/P）记忆联合构建的一次性成本可跨任务、跨时间复用，使智能体无需重新运行相互独立的抽取流水线即可支持异构记忆操作。因此，这项前期投入可在多任务、持续学习或长时程设置中自然摊销，并可能降低实际部署的 token 总成本。

### E.3 WebArena

**设置。** WebArena（Zhou et al., 2024）是一个真实且可复现的网页导航基准，共有 812 项任务，横跨五个站点域（例如 Shopping、GitLab 等）；每项任务均由精心整理的 241 个意图模板之一实例化而来。本文关注 Shopping、GitLab 和 Multi-site 子集，它们共同覆盖：i）特定领域内程序密集的交互（Shopping、GitLab）；ii）组合式跨站点工作流（Multi-site）。为直接检验记忆的演化与复用，我们采用与 WebArena 任务构造方式一致的在线/离线划分：WebArena 的意图被写成各有多个实例的模板；同一模板下的任务共享高层语义，但可能需要不同的具体执行轨迹。具体而言，对每个模板（例如含 5 个实例），我们将其中一个实例划入在线集，其余实例划入离线集。为进一步检验跨模板知识适应能力，仅含一个任务 ID 的模板被保留在离线集中。精确的数据划分存放于我们的代码仓库。

**表 17：HotpotQA 上的 Token 成本统计。**（$\tilde{k}$ tokens）

| 方法 | 输入 | 输出 | 总计 |
|---|---|---|---|
| **任务特定方法** | | | |
| RAPTOR | 331 | 40 | 371 |
| HippoRAG2 | 1350 | 490 | 1840 |
| PropRAG | 1651 | 643 | 2294 |
| **任务无关方法** | | | |
| Vanilla RAG | 110 | 20 | 130 |
| A-Mem | 1778 | 261 | 2039 |
| **本文方法** | | | |
| PlugMem | 1919 | 272 | 2191 |

我们首先在在线集上评估智能体，然后用少量高质量人工演示增强记忆模块，最后使用增强后的记忆模块在离线集上评估，以衡量智能体对新获得的程序性知识和语义知识的泛化能力。在在线集评估期间，智能体可以插入和检索记忆；在离线集上则只允许检索。对于 PlugMem，我们将基础智能体设为采用 GPT-4o 的 AgentOccam（Yang et al., 2025a）。AgentOccam 是一个强基线，工作在 WebArena 定义的动作空间内，且不依赖专门的记忆机制。对所有方法，我们采用 GPT-4o 和 Qwen2.5-32B-Instruct（Qwen et al., 2025）作为骨干 LLM，并采用 NV-Embed-v2（Lee et al., 2025）作为嵌入模型；解码参数为 `max-tokens = 2048`、`temperature = 0.0`、`top-p = 1.0`。对于 PlugMem 内部的所有 LLM 调用，我们先使用 Qwen2.5-32B-Instruct；仅当其输出困惑度高于预设阈值时，才把查询交给 GPT-4o。对于 PlugMem，我们记录网页智能体在最大允许步数内的完整动作轨迹（所有实验均为 20 步）。检索时设置 top-$k=2$，向下游推理返回两个最相关的记忆节点。

**PlugMem 与 AgentOccam 的集成。** 在实验中，PlugMem 与基础网页导航智能体集成。给定目标后，PlugMem 先检索记忆，再将推理模块的响应传给 AgentOccam。AgentOccam 执行动作（例如 `click [1234]`）后，PlugMem 为该动作生成简短的自然语言描述（例如“点击‘我的账户’”），并将该描述与动作后的新观察共同组成一条新的情景记忆。每项任务结束时，系统按照方法部分和附录 C 所述过程，将情景记忆序列插入 PlugMem。我们对 AgentOccam 的提示做了轻微修改，加入一条遵循 PlugMem 检索所得记忆条目的指令；新增指令见清单 15。

**PlugMem 纳入人工演示。** PlugMem 能将人工演示轨迹纳入记忆图，以实现热启动。在实验中，我们记录在线集中失败任务的人工演示，再将这些记录注入系统用于离线评估。具体做法是“重放”演示轨迹，并重建用于插入的情景记忆序列。离线集评估时，我们禁用记忆图插入，以检验现有记忆图的质量。通常，对于特定任务域，我们会在同一任务域内记录并插入人工演示。对于 Multi-site 任务域，除了 Multi-site 任务的演示，我们还额外加入其他任务域的轨迹，以进一步强化记忆图。借此，我们测试 PlugMem 的跨域检索和推理能力。在开发过程中，我们共收集 23 条 Shopping 演示、18 条 GitLab 演示、10 条 Reddit 演示、10 条 Map 演示和 5 条 Multi-site 演示。这些人工演示可从我们的代码仓库获得。
**基线。** 我们考虑三类基线：i）朴素方法，即不依赖检索增强的基础智能体，用于衡量骨干 LLM 的性能；这里使用 AgentOccam（Yang et al., 2025a）。ii）任务无关方法，即采用并非专为网页导航智能体设计的检索增强基线。我们纳入一条存储并检索过往交互轨迹的朴素稠密检索流水线，也评估 A-Mem（Xu et al., 2025）：该智能体记忆系统将过往交互概括为笔记，并以类图结构组织这些笔记，以便检索相关轨迹。朴素检索设置 top-$k=1$；A-Mem 设置 top-$k=3$，以充分利用其检索相关笔记的能力。两种方法对每项任务最多存储前 10 步。iii）任务特定方法，即使用专门记忆插入和检索机制的开源智能体系统。我们纳入 AWM（Wang et al., 2024b），它将过往交互概括为工作流。为了考察和比较各方法利用高质量人工演示的能力，我们在所有任务无关和任务特定基线的在线评估与离线评估之间，都插入相同的人工演示轨迹。

**表 18：WebArena 上的 Token 开销统计。**（每个样本 $\tilde{k}$ tokens）

| 方法 | Q32 输入/输出 | 4o 输入/输出 |
|---|---|---|
| 任务无关方法 | | |
| Vanilla RAG | - | - |
| A-Mem | - | 7431/1758 |
| 任务特定方法 | | |
| AWM | - | 4918/1202 |
| 本文方法 | | |
| PlugMem | 10587/1377 | 3128/371 |

**指标。** WebArena 使用功能正确性验证器评估任务完成情况，即通过程序化检查判定执行是否达成预期目标。我们使用 WebArena 的评估器，并报告在线集与离线集上的成功率。我们按每个任务样本计算 PMI。计算 PMI 时采用加性平滑，并使用

$$\log\left(\frac{P_{\mathrm{mem}}+\epsilon}{P_{\mathrm{base}}+\epsilon}\right)$$

。我们将 $\epsilon$ 设为 AgentOccam 在 WebArena 上平均成功率的 1%。

**消融实验。** 我们进一步对 PlugMem 的三个组件进行消融研究。对于“无结构化”，用朴素检索过程替换记忆图构建流水线，同时保留推理模块；对于“无检索”，使用随机抽取的结构化记忆条目填充检索候选集。此外，对于“无人工演示”，我们省略在线与离线评估之间插入高质量人工演示的中间步骤。根据表 8，移除任一组件都会降低智能体成功率或全局信息密度。我们发现，移除检索组件对所有测量指标造成的负面影响最大。

**Token 成本分析。** 如表 18 所示，PlugMem 在 WebArena 上的平均 token 用量高于任务无关与任务特定基线。这一差异反映的是从记忆到知识的抽象所需的额外计算，而不是冗余处理。

在 WebArena 中，可复用知识主要是程序性知识，并且每个任务包含多个必须记录、切分和抽象的交互步骤。因此，PlugMem 使用额外 token 来标准化较长的情景轨迹，并将其组织为与子目标对齐的程序性记忆单元。与只存储原始轨迹或单一静态工作流的方法相比，该过程自然需要更多计算。

值得注意的是，PlugMem 消耗的额外 token 大多由开源模型生成，而若干基线依赖少量闭源模型调用。这些方法看似节省 token，但实际每 token 成本要高得多。相比之下，PlugMem 的 token 用量保持在实用范围内，同时支持更丰富、可复用的程序性表示。

此外，这些成本主要发生在在线阶段，并可在多个具有相似用户意图的任务实例之间摊销。程序性记忆图一旦构建，便可直接复用，无需重新运行完整的抽象流水线；因此，随着任务时程延长、任务多样性增加，其有效成本会逐渐下降。

**清单 15：供 AgentOccam 纳入检索记忆的补充指令。**

```
============================================================
Prompt AgentOccam Retrieved Memory Instruction
============================================================
检索到的记忆包含过往经验中的相关信息，可能有助于你完成当前任务。使用这些记忆来：
- 学习相似过去任务及其成功策略
- 避免重复先前错误
- 应用在相似情形中奏效的、经过验证的方法
- 理解能够指导你行动的模式与关系
当提供检索记忆时，仔细考虑它与当前任务的关系，并把相关洞见纳入决策过程。
有些任务可能需要访问 Wikipedia；你访问 Wikipedia 的唯一方式是通过 URL
"http://<BASE_URL>:8888/wikipedia_en_all_maxi_2022-05/A/User:The_other_Kiwix_guy/Landing"
BASE_URL: localhost
```

## 附录 F 在 PlugMem 之上集成任务特定启发式规则进行任务适配

### F.1 LongMemEval

**反思式记忆管理。** RMM（Tan et al., 2025）引入了一种新颖的长期记忆管理反思机制。总体而言，RMM 采用前瞻性反思进行知识抽取与记忆更新，并采用回顾性反思，以在线强化学习方式优化记忆检索。

受这些想法启发，我们面向 LongMemEval 对 PlugMem 进行任务适配，把 RMM 的前瞻性反思机制用于语义记忆抽取和语义节点更新。就记忆抽取而言，我们采用 RMM 提出的抽取提示，为 PlugMem 提炼语义记忆。与 PlugMem 原先的语义记忆抽取提示（清单 5）相比，任务适配版更强调抽取特定于用户的个人信息，并加入上下文示例，以更好地契合 LongMemEval 设定。就记忆更新而言，我们遵循 RMM 提出的反思式更新策略，在记忆图内更新语义记忆。与原版 PlugMem 直接插入新抽取记忆不同，任务适配版先计算语义记忆嵌入之间的余弦相似度并据此排序，从记忆图中检索 top-$K$ 个最相似的现有语义记忆。随后查询 LLM，判断新记忆应作为新节点加入，还是与某个相似的现有节点合并。记忆抽取和记忆更新提示分别见清单 16 与清单 17。

**实验。** 我们在 LongMemEval 的多会话子集上进行实验，结果如表 19 所示。多会话子集是 LongMemEval 中最具挑战性的部分之一，因为它要求智能体回答的问题主要涉及跨多个会话对一系列相似事件出现次数进行计数。这一设定不仅对记忆模块的检索模块提出了同时检索多条黄金记忆的显著要求，也对结构化与推理模块提出了生成精确语义记忆并对其执行准确计数的挑战。任务适配版本优于原方法，主要有两个原因：第一，任务适配的记忆提取提示能产生更精确的语义记忆；第二，反思式更新策略防止冗余和重复的语义记忆被插入记忆图，从而使检索更加高效。

**表 19：LongMemEval 多会话子集上的任务适配。**

| 方法 | 准确率（多会话） | PMI | #TokAvg. | 信息密度 |
| --- | --- | --- | --- | --- |
| PlugMem | 64.7 | 5.06 | 311.9 | 1.62e-2 |
| PlugMem 任务适配版 | 70.7 | 5.62 | 327.5 | 1.71e-2 |

**清单 16：用于任务适配的语义记忆抽取提示模板。**

```
============================================================
Prompt Extract_Memory
============================================================
任务描述：给定用户与智能体之间的一轮对话会话，抽取用户的个人摘要。确保输出遵循以下规则：
按 OUTPUT 格式输出结果。顶层标题为 "### Memory"。其值应为字典列表，其中每个字典具有键 "Summary"：
- summary：一份简洁的个人摘要，捕捉跨多个轮次中与用户经历、偏好和背景相关的信息。
若无法提取任何个人摘要，则返回 NO_TRAIT。
示例：
INPUT:
Turn 0:
- User: 你去看过城里那家新健身房吗？
- Agent: 去了。不过我不太确定自己喜不喜欢那儿的氛围。
Turn 1:
- User: 那儿有什么问题？
- Agent: 那儿的人似乎更在意自己的外表，而不是锻炼。对我来说有点太时髦了。我比较朴素。
Turn 2:
- User: 啊，明白了。那也许那些老一点的健身房会更适合你-或者我猜你也可以买你之前说的那台跑步机。你现在更倾向哪一边？
- Agent: 我更倾向于跑步机。我觉得它更适合我的生活方式。我只是不知道该买哪种。外面选择太多了。你在健身房用跑步机吗？对家用的有什么建议吗？
Turn 3:
- User: 老实说，我通常只在那儿举重。不过我想我听说过 NordicTrack 的评价不错？
- Agent: 是的，我也听说过不错的评价。我喜欢多功能健身器材这个想法。只要天气不太糟，我就更喜欢出去跑步。但因为这儿经常下雨，我喜欢有个室内选择。新英格兰的天气怎么样？
OUTPUT:
### Memory:
1. **Summary:** User 询问了城里一家新健身房，并建议把老健身房或跑步机作为替代方案。
2. **Summary:** User 通常在健身房举重，而不是使用跑步机。
3. **Summary:** User 听说过 NordicTrack 跑步机的评价不错。
任务：遵循上述示例所展示的 OUTPUT 格式，从以下对话会话中抽取用户的个人摘要。
输入：{episodic_memory}
输出：
```

**清单 17：用于任务适配的语义记忆更新提示模板。**

```
============================================================
Prompt Update_Memory
============================================================
任务描述：给定某一特定用户的历史个人摘要列表，以及来自同一用户的一条新的、相似的个人摘要，按以下说明更新用户的历史摘要：
输入格式：历史个人摘要与新的个人摘要均以 JSON 格式提供，顶层键为 "history_summaries" 和 "new_summary"。
可选更新动作：
- Add：若新个人摘要与任何历史个人摘要都不相关，则添加它。
格式：Add()
- Merge：若新个人摘要与某条历史个人摘要相关，则将二者合并成一条更新后的摘要。
格式：Merge(index, merged_summary)
注意：index 是相关历史摘要在列表中的位置。merged_summary 是新摘要与相关历史摘要的合并结果。如果两条摘要讨论用户个人信息或经历的同一方面，则视为相关。
不要在输出中包含额外解释或示例-仅返回所需的动作函数。
示例：
INPUT:
History Personal Summaries:
- {{"history_summaries": ["SPEAKER_1 会锻炼，尽管他并不特别喜欢锻炼。"]}}
New Personal Summary:
- {{"new_summary": "SPEAKER_1 每周一和周四锻炼。"}}
OUTPUT ACTION:
Merge(0, SPEAKER_1 每周一和周四锻炼，尽管他并不特别喜欢锻炼。)
任务：遵循上述示例格式，针对给定案例更新个人历史。
INPUT:
历史个人摘要：{history_summaries}
新的个人摘要：{new_summary}
OUTPUT ACTION:
```

### F.2 HotpotQA

**来自 HippoRAG2 的启发。** 我们的检索设计部分受到 HippoRAG2 的启发，该工作表明，引入显式图结构以及受控的多跳遍历，可以在多跳问答上优于纯粹基于相似度的检索。特别是，HippoRAG2 将识别（过滤/选择相关的结构化单元）与扩展（基于图的传播以检索支持证据）相分离，从而减少虚假跳转并提高证据质量。PlugMem 采纳了这一高层原则，但将其推广到类型化智能体记忆。我们构建语义记忆图与程序性记忆图，并使用抽象节点（概念/意图）作为路由信号以激活特定的命题/处方节点，从而实现从抽象到具体的遍历。此外，类似于 HippoRAG2 的识别步骤，PlugMem 采用基于 LLM 的控制器来 i) 选择记忆类型（GETMODE），ii) 提出下一跳抽象信号（GETPLAN），以及 iii) 在固定预算下决定提前停止或聚焦候选（MULTIHOPCTRL）。最后，我们整合双通道候选更新——通过抽象节点进行基于链接的扩展，以及从精炼查询进行基于嵌入的检索——随后进行重排序/剪枝以及可选的基于推理的压缩。

**测试时扩展。** 除默认配置外，PlugMem 还提供两个实用的测试时“旋钮”，均可稳定提升性能。第一，可提高最大跳数 $T_{\max}$，从而在推理时扩展检索计算量。这使检索器能串联更多抽象到具体的步骤，类似于推理型方法中的测试时扩展。我们在 HotpotQA 的一个子集上开展实验；当把多跳上限 $T_{\max}$ 从 2 提高到 4 时，以精确匹配/F1 分数衡量的性能从 66.00/74.73 提升至 69.00/78.11。

第二，在构造下一跳查询 $Q_{t+1}$ 时，可用基于 LLM 的查询合成器替代对 $(Q_t,\text{所选事实})$ 的朴素拼接，由合成器将其重写为聚焦且自包含的查询。经验上，基于合成的精炼可减少词汇漂移与冗余，并帮助嵌入通道在后续跳中检索更有针对性的证据。我们把这两种机制都视为检索控制器内部的可控组件，从而可根据任务预算平滑权衡准确率与成本。

### F.3 WebArena

先前专门面向 WebArena 的工作常把人工设计的领域级指令（或“技巧”）纳入智能体的输入上下文。任务特定适配的一种常见做法——例如 AWM（Wang et al., 2024b）、SteP（Sodhi et al., 2024）和近期的 ColorBrowserAgent（Zhou et al., 2026）——是加入显式指导，使智能体偏向站点域内的特定交互模式，例如导航惯例或首选 UI 元素。与此不同，在主要实验中，我们有意避免注入这类手写指令或显式定义的动作策略；只依赖 PlugMem 自动存储、检索并重新应用智能体轨迹和人工演示轨迹，将其作为结构化记忆，从而让行为由经验涌现，而非来自提示工程。

**进一步的任务适配。** 我们还开展了一项针对性案例研究，以考察在已知此类领域特定约束时，PlugMem 能否受益于人工任务适配。我们识别出一组 PlugMem 仍难以完成的 WebArena 任务，并向推理模块加入少量人工设计的指令。具体而言，我们观察到 PlugMem 偶尔会指示智能体通过类别下拉菜单导航，而 WebArena 中的正确策略是使用搜索栏。加入清单 18 所示的补充指导后，我们通过显式约束智能体的动作选择，解决了若干此前失败的案例（见表 20）。WebArena 的成功标准严格且由程序判定，但查询较为简短；在这种情况下，原本合理的策略也可能被判定为错误。尽管 PlugMem 能成功检索与“搜索磨牙护齿套”等任务相关的程序性知识，它默认并不会推断出某些看似合理的动作会受到该基准隐含规则的禁止。该案例研究表明，PlugMem 在必要时原生支持细粒度人工条件化，同时在一般情况下无需任务特定提示也能保持有效。

**表 20：带有人工指令的 PlugMem 示例。**

| WebArena | 人工指令 | 效果 |
| --- | --- | --- |

**查询。** “One Stop Market 中磨牙护齿套的价格范围是多少？”

> **译注：** 原文查询的是“磨牙护齿套”，但下方“带有人工指令”的推理与动作却改为“无线耳机”；此处按原文保留该不一致。

**带有人工指令的 PlugMem。**

▷ 推理

用户的问题是要确定 One Stop Market 中无线耳机的价格范围。根据指南，当被问及某一种特定产品（例如“无线耳机”）时，智能体必须使用搜索栏定位该产品，而不是通过下拉菜单导航。当前观察显示搜索栏存在，但看起来被禁用。然而，根据所提供的相关经验，仍建议尝试使用搜索栏，因为其功能状态未必能由外观准确反映。

▷ 最终信息

1. 在搜索栏输入“wireless earphone”并按 Enter 键，以定位相关产品。
2. 一旦显示搜索结果，查看所列无线耳机的价格以确定价格范围。
3. 如果搜索栏无法工作，报告该问题并考虑定位该产品的替代方法，例如联系支持或探索其他导航选项。

AgentOccam Action 1: type [1913] [wireless earphone] [1]. （智能体使用搜索栏）

...

**不带人工指令的 PlugMem。**

▷ 推理

用户的问题是要查找 One Stop Market 中磨牙护齿套的价格范围。该观察未提供任何关于磨牙护齿套或其价格的直接信息。然而，相关经验表明，智能体应使用层级菜单结构导航到适当的类别（可能在 ”Beauty & Personal Care” 或 ”Health & Household” 之下），然后细化搜索以定位具体产品类型。如果有价格过滤器可用，可以应用它来确定价格范围。

▷ 最终信息

为了回答该问题，智能体应当：

1. 将鼠标悬停在层级菜单结构上，从“Beauty & Personal Care”或“Health & Household”等类别开始。
2. 寻找与口腔护理或牙科护理相关的子类别。
3. 导航到相关子类别（例如“Oral Care”或“Mouth Guards”）。
4. 查看该子类别中列出的产品，以识别磨牙护齿套并记录其价格。
5. 如果有价格过滤器可用，应用它以更高效地确定价格范围。
6. 汇总所识别产品的价格范围。

AgentOccam Action 1:

hover [1534]. （智能体操作下拉菜单（错误！））

...

**清单 18：供智能体与推理模块进一步执行任务适配的补充指令。**

```
============================================================
Prompt WebArena Task-specific Instructions
============================================================
1. 当被问及某种产品（而非一个类别）时，使用搜索栏搜索该产品，而不是使用类别下拉菜单。
2. 当被问及某一类产品时，通过 hover 动作访问下拉菜单。
```

## 来源

- `papers/agent-memory/PlugMem - A Task-Agnostic Plugin Memory Module for LLM Agents/PlugMem - A Task-Agnostic Plugin Memory Module for LLM Agents.pdf`（全文 49 页）
