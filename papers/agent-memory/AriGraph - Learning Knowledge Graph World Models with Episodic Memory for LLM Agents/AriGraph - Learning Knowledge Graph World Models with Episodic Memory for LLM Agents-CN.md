# AriGraph：面向 LLM 智能体、利用情景记忆学习知识图谱世界模型

Petr Anokhin¹，Nikita Semenov²，Artyom Sorokin¹，Dmitry Evseev²，Andrey Kravchenko⁴，Mikhail Burtsev³，Evgeny Burnaev²˒¹

¹ AIRI，俄罗斯莫斯科  
² Skoltech，俄罗斯莫斯科  
³ London Institute for Mathematical Sciences，英国伦敦  
⁴ University of Oxford，英国  
anokhin@airi.net

> 译注：本译稿对应 arXiv:2407.04363v3（2025 年 5 月 15 日）。模型名、数据集名、游戏名和方法名保留英文；术语首次出现时给出英文。公式、算法编号、图表编号和引文编号均与原文一致。参考文献保留原文，以便检索。

## 摘要

大语言模型（Large Language Models，LLM）能力的进步，为开发自主智能体奠定了颇具前景的基础。借助合适的工具，这些智能体可以通过积累和更新知识，学会在新环境中解决任务。现有基于 LLM 的智能体通常借助完整观察历史、摘要或检索增强来处理过往经验。然而，这些非结构化的记忆表示不利于复杂决策所必需的推理与规划。本研究提出 AriGraph：智能体在探索环境的过程中构建并更新一张融合语义记忆与情景记忆的记忆图。我们证明，Ariadne LLM 智能体由所提出的记忆架构以及规划和决策模块组成，能够有效处理交互式文字游戏环境中的复杂任务，而这些任务即使对人类玩家也有相当难度。结果表明，在一系列复杂度各异的问题上，我们的方法显著优于其他成熟的记忆方法和强强化学习（reinforcement learning，RL）基线。此外，在静态多跳问答中，与专为知识图谱设计的方法相比，AriGraph 也表现出有竞争力的性能。

## 1 引言

大语言模型（LLM）令人瞩目的语言生成能力，引发了人们对将其用作自主智能体核心组件的浓厚兴趣；这类智能体能够与动态环境交互并执行复杂任务。过去一年中，研究界已经探索了此类 LLM 智能体的通用架构与核心模块 [Wang et al., 2024; Sumers et al., 2024; Cheng et al., 2024]。通用认知智能体的一项关键属性，是积累并运用知识的能力。长期记忆使智能体能够存储和回忆过往经验与知识，从此前的遭遇中学习，并作出有依据的决策。然而，究竟应如何以最佳方式赋予智能体这些能力，仍是一个开放问题。尽管 Transformer 架构具有固有限制，现有方法已经能让 LLM 处理包含数百万 token 的上下文 [Ding et al., 2024b]。但对于需要与环境持续交互的智能体而言，这种做法效率不高。此类智能体必须把整个历史上下文保存在记忆中才能采取行动；这不仅成本高昂，而且难以处理隐藏在海量信息中的复杂逻辑。围绕循环记忆 Transformer（Recurrent Memory Transformer）[Bulatov et al., 2022; Bulatov et al., 2024] 和 MAMBA [Gu and Dao, 2023] 等替代框架的研究，正尝试提供长期记忆方案，不过这些模型仍处于早期阶段。

目前，为 LLM 智能体引入记忆的最常见方案是检索增强生成（Retrieval-Augmented Generation，RAG）。向量检索形式的 RAG 借助外部数据库，以相关信息增强模型的提示词。这项技术常用于 LLM 智能体的记忆架构，通常负责回忆特定观察或已学技能。然而，其非结构化特性会大幅削弱对相关信息的检索能力，因为这些信息可能散落在智能体的整段记忆中。使用知识图谱作为数据库可以克服这些局限。随着 LLM 兴起，这一思路也重新受到重视 [Pan et al., 2024]。不过，一个稳健的记忆架构必须同时整合结构化与非结构化数据。在认知科学中，这种整合对应于语义记忆（semantic memory）和情景记忆（episodic memory）两个概念。语义记忆包含有关世界的事实性知识，而情景记忆涉及个人经验，通常具有更丰富、更细致的信息。由于二者具有不同的神经表征，传统上常被视为彼此分离；但近期研究表明，这两类记忆相互关联 [Wong Gonzalez, 2018]。语义知识建立在情景记忆的基础之上，随后又为联想记忆提供结构化基础，从而能够整合记忆的不同方面，包括情景记忆本身。

在本研究中，我们开发了一种名为 Ariadne's Graph（AriGraph）的记忆架构，在记忆图框架中融合语义记忆和情景记忆。知识图谱表示相互连接的语义知识网络；情景记忆则表示为情景边，一条情景边可以连接图中的多条关系。当智能体与环境交互时，它通过更新和扩展基于知识图谱的记忆，学习一个联合的语义-情景世界模型。该架构不仅构成基础记忆框架，也有助于环境建模，并改善空间定向与探索能力。在我们称为 Ariadne 的通用 LLM 智能体框架中，采用了记忆检索、规划和决策流水线。为了评估所提出的方法，我们设计实验研究以下两个问题：

**RQ1：** 基于 LLM 的智能体能否通过与环境交互，从零开始学习有用的结构化世界模型？

**RQ2：** 结构化知识表示能否改善从记忆中检索相关事实的效果，并支持有效探索？

我们在 TextWorld 和 NetHack 环境的复杂交互任务中评估了智能体 [Côté et al., 2018; Küttler et al., 2020]。实验结果表明，Ariadne 能够有效地通过环境交互进行学习，并显著优于完整历史、摘要、RAG、Simulacra [Park et al., 2023] 和 Reflexion [Shinn et al., 2023] 等其他 LLM 记忆方法。我们还证明，本方法优于现有 RL 基线。我们也在经典 Roguelike 游戏 NetHack 上评估了本方法：仅使用局部观察的智能体，获得了与拥有真实知识的智能体相当的分数。尽管 AriGraph 最初是为与环境交互的智能体设计的，但它在多跳问答任务上也展现出有竞争力的性能。

**图 1：**（A）配备 AriGraph 记忆的 Ariadne 智能体架构。AriGraph 同时融合语义知识图谱和过往经验。以语义知识图谱为主体、再以情景顶点和情景边扩展而成的记忆，显著提升了 LLM 智能体在文字游戏中的表现。（B）本智能体在文字游戏上的平均表现，以及包括人类玩家和其他 LLM 记忆实现方案在内的多种基线。各 LLM 智能体仅记忆模块不同，决策组件在所有版本中保持一致。智能体结果取五次运行中成绩最好的三次；人类玩家同时报告所有参与者中前三名的成绩和全体参与者的平均成绩。

## 2 AriGraph 世界模型

### 记忆图结构

AriGraph 世界模型

$$
G=(V_s,E_s,V_e,E_e)
$$

由语义记忆的顶点和边 $(V_s,E_s)$，以及情景记忆的顶点和边 $(V_e,E_e)$ 构成（见图 2）。在每个时刻 $t$，智能体接收观察 $o_t$，并向环境返回动作 $a_t$。环境还返回奖励 $r_t$；奖励对 LLM 智能体不可见，但用于评估其表现。智能体通过从文本观察 $o_t$ 中提取语义三元组 $(object_1, relation, object_2)$，持续学习世界模型 $G$。

- $V_s$ 是语义顶点集合。语义顶点对应从三元组中提取的对象。
- $E_s$ 是语义边集合。语义边是元组 $(v,rel,u)$，其中 $u,v$ 是语义顶点，$rel$ 是二者之间的关系。语义边本质上表示被整合进语义记忆的三元组。
- $V_e$ 是情景顶点集合。每个情景顶点对应环境在相应时刻返回的一次观察，即 $v_e^t=o_t$。
- $E_e$ 是情景边集合。每条情景边 $e_e^t=(v_e^t,E_s^t)$ 将从 $o_t$ 中提取的所有语义三元组 $E_s^t$ 彼此连接，并将它们与对应的情景顶点 $v_e^t$ 相连。换言之，情景边表示“同时发生”这一时间关系。[^1]

[^1]: 严格来说，情景边既不能称为普通边，甚至也不能称为超边，因为它们将顶点与多条图边连接起来；为简便起见，本文仍称其为边或情景边。

### 构建 AriGraph

智能体与环境交互时，可以获得有关世界的信息，从而建立新知识或更新既有知识。给定新观察 $o_t$，LLM 智能体提取新三元组，并将其表示为语义顶点 $V_s^t$ 和语义边 $E_s^t$。为了查找与 $o_t$ 中所提对象有关的已有知识，先筛选出所有与 $V_s^t$ 中顶点关联的语义边集合 $E_s^{rel}$。随后，通过将 $E_s^{rel}$ 与 $E_s^t$ 比较，检测其中的过时边，并将其从图中删除。清除过时知识后，再用 $V_s^t$ 和 $E_s^t$ 扩展语义记忆。情景记忆的更新较为直接：加入包含 $o_t$ 的新情景顶点 $v_e^t$，以及一条把 $E_s^t$ 中所有边与 $v_e^t$ 连接起来的新情景边。情景节点保存智能体的过往历史，情景边则连接同一时刻获得的全部知识。用于提取新三元组和检测过时知识的提示词见附录 E。

### 从 AriGraph 检索

为了在部分可观察环境中成功决策，智能体需要能够检索相关知识。从 AriGraph 记忆中检索包含两个过程：（1）语义搜索返回最相关的三元组（语义边）；（2）情景搜索以已提取的三元组为输入，返回最相关的情景顶点 $V_e$。搜索伪代码见算法 1。

**算法 1：记忆图搜索**

```text
输入：查询集合 Q，Vs，Es，Ve，Ee，情景顶点数量 k，
      语义搜索深度 d 和宽度 w
输出：检索到的情景顶点 Ve^Q，检索到的语义三元组 Es^Q

Es^Q ← ∅
for each q in Q do
    Es' ← SemanticSearch(q, Vs, Es, d, w)
    Es^Q ← Es^Q ∪ Es'
end
Ve^Q ← EpisodicSearch(Es^Q, Ve, Ee, k)
return Es^Q, Ve^Q
```

语义搜索利用语义相似度和语义图结构，回忆最相关的三元组。给定一个查询，检索器（预训练 Contriever 模型 [Izacard et al., 2022]）选出最相关的语义三元组。随后，以这些三元组所关联的顶点集合为起点，从图中递归检索新边。搜索的深度和宽度分别由超参数 $d$ 和 $w$ 控制。详见附录 A。

情景搜索以语义搜索的结果为输入。情景边把输入三元组与表示过往观察的情景顶点相连。某个情景顶点所关联的输入三元组数量，用于计算该顶点的相关性：

$$
rel(v_e^i)=\frac{n_i}{\max(N_i,1)}\log\bigl(\max(N_i,1)\bigr),
\tag{1}
$$

其中，$n_i$ 是与情景边 $e_i$ 关联的输入三元组数量，$N_i$ 是与 $e_i$ 关联的三元组（语义边）总数；$\log(\max(N_i,1))$ 是加权因子，用以防止信息量低的观察获得过高分数。我们采用 $\log_2(N_i)$ 缩放，使提取出更多三元组的观察获得更高权重。此外，只包含一个三元组的观察被赋予零权重，因为它们不太可能提供超出该三元组本身的信息。最终返回相关性最高的 $k$ 个情景顶点，以及其中相应的观察。

## 3 Ariadne 认知架构

为了检验 AriGraph 世界建模方法的效用，我们提出名为 Ariadne 的智能体架构。Ariadne 与未知环境交互，以完成用户设定的目标。在此过程中，智能体在每个时间步学习世界模型、进行规划并执行动作。Ariadne 的长期记忆存储为 AriGraph；其工作记忆则包含当前规划和决策所需的信息。

给定一次观察后，智能体更新世界模型，并从 AriGraph 中检索语义知识和情景知识，放入工作记忆。工作记忆还包含最终目标描述、当前观察，以及最近的观察与动作历史。在规划阶段，Ariadne 使用工作记忆中的内容建立新计划或更新现有计划。计划由一系列与任务相关的子目标构成，每个子目标附有简要说明。规划模块还会依据环境对时刻 $t-1$ 所执行动作的反馈评估动作结果，并据此调整计划。

修改后的计划被加入工作记忆，供决策模块访问。决策模块负责选择最符合当前计划目标的动作。该模块遵循 ReAct [Yao et al., 2023] 框架，要求智能体在执行动作之前先阐明采取该动作的理由。将规划与决策分离，使 LLM 能够分别专注于不同的认知过程。在文字环境中，智能体从有效动作列表中选择动作。智能体还可以使用记忆模块提供的图专用导航函数。该函数把“前往某位置”类型的命令加入动作空间，并依据语义图中存储的空间关系，推断通往目标位置的最优路线。

**图 2：AriGraph 世界模型与 Ariadne 认知架构。**（A）AriGraph 在与未知环境交互的过程中学习情景知识和语义知识。每个时间步 $t$ 都会向情景记忆加入一个包含完整文本观察 $o_t$ 的新情景顶点。随后，LLM 解析观察 $o_t$，以三元组 $(object_1,relation,object_2)$ 的形式提取相关关系。这些三元组用于更新语义记忆图。情景记忆与语义记忆通过情景边连接：每条情景边将一个情景顶点与从相应观察中提取的全部三元组相连。（B）Ariadne 智能体借助 AriGraph 探索环境并完成任务。用户向智能体设定目标。工作记忆由近期观察和动作历史，以及从 AriGraph 世界模型检索出的相关语义知识和情景知识填充。规划 LLM 模块利用工作记忆的内容生成新计划或更新现有计划，规划结果再写回工作记忆。最后，基于 ReAct 的模块读取记忆内容，选择一个可在环境中执行的动作。每次观察都会触发学习过程，从而更新智能体的世界模型。

## 4 实验设置

### 4.1 TextWorld 交互环境

我们在一系列涉及空间导航、物体收集和工具操作的文字游戏中，将 Ariadne 与替代方法进行比较。每个游戏的详细说明（包括难度等级和环境地图）见附录 F。这些游戏均可视为部分可观察马尔可夫决策过程（Partially Observable Markov Decision Process，POMDP）。长期以来，此类游戏一直是研究智能体能否有效记忆信息并建立长期依赖的基准 [Parisotto et al., 2020; Pleines et al., 2022; Sorokin et al., 2022]。

**寻宝（Treasure Hunting）。** 主要目标是取回隐藏的宝藏；一系列房间提供通往最终目标所需的钥匙和线索。基础版本包含 12 个房间和 4 把钥匙，困难版本包含 16 个房间和 5 把钥匙，最难版本则包含 36 个房间、7 把钥匙以及每个房间中的额外干扰物品。

**清洁（Cleaning）。** 目标是识别放错位置的物品，并将其送回正确地点，从而清理房屋。环境包含 9 个房间（厨房、泳池等）和 11 件放错位置的物品（此外还有许多其他物品）。要解决该问题，智能体必须记住房间和物体的位置，并推理物品应当放置的位置。

**烹饪（Cooking）。** 目标是按照食谱、选择正确食材、使用适当工具并在多房间住宅中导航，最终制作并食用一道菜。该任务检验智能体记住相关信息并据此规划的能力。基础难度任务包含 9 个地点和 3 种食材；困难任务包含 12 个地点和 4 种食材；最难任务还加入了上锁的门和物品栏管理。

对于各项基线，我们保留 Ariadne 的规划和决策模块，但用下列一种记忆类型替代 AriGraph：完整的观察与动作历史、迭代摘要、RAG、带 Reflexion 的 RAG [Shinn et al., 2023]，以及 Simulacra（来自 [Park et al., 2023] 的记忆实现）。

完整历史会保留所有观察和动作的完整记录，并在每一步决策时提供这些记录。摘要则不保存完整历史，而只保留必要信息并丢弃其余内容。标准 RAG 基线依据与当前观察和计划的相似度得分，检索 top-$k$ 条记忆。Simulacra 具有一套融合时近性、重要性和相关性的评分机制，并会对提取出的记忆进行反思。Reflexion 基线与其他方法不同，它跨多个试次运行：某次尝试失败后，智能体反思该次轨迹，记录可能有助于后续试次解决任务的信息。AriGraph 和其他基于 LLM 的基线均以 `gpt-4-0125-preview` 作为 LLM 骨干。

此外，我们还在 [Adhikari et al., 2021] 烹饪测试的一个变体上检验本架构，以与 RL 基线比较。这些任务分为 4 个难度级别，但明显比我们的主要任务简单：地点、食材和必需动作都更少（见附录 F）。

在 RL 基线方面，我们收集 [Adhikari et al., 2021; Tuli et al., 2022; Basu et al., 2024] 针对 [Adhikari et al., 2021] 的四级烹饪任务所报告的 GATA、LTL-GATA 和 EXPLORER 架构最佳结果。

为了估计人类在相同游戏中的表现，我们开发了图形用户界面，让志愿者游玩寻宝、清洁和烹饪的基础版本。收集数据后，我们排除了未完成游戏的会话。

### 4.2 NetHack 环境

NetHack [Küttler et al., 2020] 是一款经典 Roguelike 冒险游戏，具有程序生成的多层地牢（地牢层级示例见附录图 13）。它对基于 LLM 和基于 RL 的方法都构成显著挑战，需要复杂的探索、资源管理与策略规划。

我们的实验基于 NetPlay [Jeurissen et al., 2024] 智能体；在无需微调或 RL 的 LLM 智能体中，该方法展现了当前最先进的性能。NetPlay 接收包含当前已探索地牢层全部信息的文本观察（Level obs），这些观察实际上充当了人工构造的记忆预言机（memory oracle）。

为评估 Ariadne，我们将文本观察限制为智能体当前所在房间或走廊（Room Obs），以检验 AriGraph 世界模型能否通过记住所有相关的层级信息，弥补这一限制。我们比较三个智能体：第一，接收受限文本观察的 NetPlay [Room obs]；第二，接收 Room Obs 并更新 AriGraph 的 Ariadne [Room obs]；第三，可访问已探索层级信息的 NetPlay [Level obs]。

### 4.3 多跳问答

尽管我们的记忆架构最初是为与环境交互的智能体设计的，但我们也在标准多跳问答基准 MuSiQue [Trivedi et al., 2022] 和 HotpotQA [Yang et al., 2018] 上评估其表现，以展示其在更标准检索任务中的稳健性和效率。我们对提示词做了少量调整，并用 BGE-M3 [Chen et al., 2024] 替换 Contriever，因为前者更适合通用文本编码。与 [Li et al., 2024a] 类似，我们从两个数据集中分别随机抽取 200 个样本。我们将本方法与 GraphReader [Li et al., 2024a]、ReadAgent [Lee et al., 2024]、HOLMES [Panda et al., 2024]、GraphRAG [Edge et al., 2024]，以及 [Li et al., 2024a] 提供的 RAG 基线进行比较。

## 5 结果

### 5.1 TextWorld

每个基于 LLM 的智能体都有五次尝试机会来解决每个游戏。归一化得分为 1 表示智能体完成游戏；小于 1 的得分表示中间进度。文字游戏的结果见图 3，性能随步骤变化的动态见附录 G。我们以五次运行中最佳三次的平均值估计性能。Ariadne 在三个任务中都能成功记住并利用世界状态信息。各基线智能体都无法完成寻宝任务，在“最难寻宝”中甚至找不到第二把钥匙。相比之下，Ariadne 大约用 50 步成功完成寻宝，在困难寻宝中保持稳健性能，并且还能完成最难寻宝；后者的房间数超过困难版的两倍，且具有额外钥匙和干扰项（见附录 G）。

与寻宝相比，清洁游戏构成了略有不同的挑战：相比确保任何信息都不丢失，正确过滤关于物品位置的过时信息更为重要。从 Ariadne 中情景记忆的效用降低，以及完整历史基线效果下降，都可以看出这一点，因为这两种记忆模块都侧重保留长期信息。总体而言，Ariadne 在该游戏中显著优于其他方法。此外，Ariadne 也优于在回合间可获得额外信息的 Reflexion [Shinn et al., 2023]。与 RAG 相比，Reflexion 在第二次尝试时性能显著提升，但在后续尝试中出现退化。

烹饪游戏难度最高，因为中间任一步骤出错都会导致整局任务无法完成。除具有明显额外优势的 Reflexion 2-shot 外，所有基线智能体都因信息不足或误用信息而无法完成烹饪任务。在该游戏中，情景记忆尤为重要：它让智能体能够回忆食谱内容或烹饪说明等有用的观察。各方法的 token 用量见附录 D 表 3。

**图 3：** AriGraph 世界模型使 Ariadne 能够成功解决多种文字游戏。（A）Ariadne 优于采用其他记忆类型的基线智能体。（B）具有情景记忆和语义记忆的 Ariadne 能扩展到更困难的环境，而不会损失性能。（C）Ariadne 的表现与最佳人类玩家相当。纵轴为相对于各环境最高可得分计算的归一化得分；误差条表示标准差。烹饪任务最大步数设为 60，其他游戏为 150。

与 RL 基线在烹饪任务变体上的比较见图 4。我们让 Ariadne 和采用完整历史的 GPT-4 在烹饪基准的 4 个难度级别上运行 [Adhikari et al., 2021]。Ariadne 在全部 4 个级别上均优于 RL 智能体，且在较难级别上的优势尤其明显。使用完整历史的 GPT-4 智能体只能解决前两个级别；这一结果与前文一致，因为图 3A 中的烹饪任务比第 4 级更难。

**图 4：** 与 RL 替代方案相比，Ariadne LLM 智能体表现最佳。图中比较了 Ariadne、完整历史基线（GPT-4）和烹饪基准中的 RL 基线；Ariadne 在全部 4 个难度级别上均表现更优。

**人类评估。** 与人类玩家的比较见图 3C。“Human All”是全部有效（已完成）人类试次的平均得分；“Human Top-3”是每项任务三次最佳游玩的平均得分。Ariadne 在所有任务上都优于样本中的普通人类玩家；在烹饪和寻宝中与最佳人类玩家得分相近，但在清洁任务上较差。

**图质量。** 我们测量了游戏期间 AriGraph 的增长率和更新率（见图 5）。图在探索阶段快速增长；智能体熟悉环境后，增长曲线趋于平缓。我们认为，这说明即使语义图不断更新，智能体也能够泛化到与环境的长时交互。附录 C 的额外结果表明，随着 LLM 骨干质量提升，图的增长率会下降。

总体结果清楚表明，Ariadne 相对于基于 LLM 和 RL 的基线具有优势。语义记忆使 Ariadne 能够构建并更新有关 POMDP 环境当前状态的知识；这对于交互环境中的导航、探索和捕捉相关细节至关重要。另一方面，情景记忆帮助智能体检索语义记忆可能没有捕捉到的详细长期信息，烹饪任务的结果体现了这一点。

**图 5：** AriGraph 在学习过程中以及面对更大环境时均表现出良好的扩展性。知识图谱规模会在探索和学习阶段迅速饱和。当困难版寻宝与烹饪游戏包含更多房间和物体时，知识图谱仅温和增长。

### 5.2 NetHack

结果见表 1。“Score”列显示 3 次运行的平均游戏得分；“Levels”列显示智能体平均完成的地牢层数。所有智能体都使用 GPT-4o。结果凸显了记忆在该任务中的重要性：可访问记忆预言机的 NetPlay [Level obs] 得分最高；只能观察当前房间的 NetPlay [Room obs] 表现最差。Ariadne [Room obs] 成功利用 AriGraph 世界模型，取得了与记忆预言机基线相当的性能。

**表 1：Ariadne 在仅获受限局部观察时，表现可与获得完整层级信息的 NetPlay 相比。**

| 方法 | 得分 | 完成层数 |
|---|---:|---:|
| Ariadne（Room obs） | $593.00 \pm 202.62$ | $6.33 \pm 2.31$ |
| NetPlay（Room obs） | $341.67 \pm 109.14$ | $3.67 \pm 1.15$ |
| NetPlay（Level obs） | $675.33 \pm 130.27$ | $7.33 \pm 1.15$ |

### 5.3 多跳问答

我们将 AriGraph 与最新的基于 LLM 的方法进行比较；这些方法会构建并检索知识图谱，以回答文档上的问题（表 2）。我们的记忆架构由 Ariadne TextWorld 智能体改造而来，同时使用 GPT-4 和 GPT-4o-mini；它优于 ReadAgent（GPT-4）、GPT-4 RAG、GPT-4 完整上下文和 GraphReader（GPT-4）等基线。GraphRAG 是一个强有力的 GPT-4o-mini 基线，但成本极高。AriGraph（GPT-4o-mini）在 MuSiQue 上表现较弱，却在 HotpotQA 上优于 GraphRAG。值得注意的是，本方法的成本比 GraphRAG 低 10 倍以上（附录 D 表 3）。

使用 GPT-4 时，HOLMES 取得最佳性能，而 AriGraph（GPT-4）的结果与之相当。值得注意的是，所有基线方法都是专门为问答任务设计的，包含任务专用提示调优和额外架构增强。GraphRAG 和 HOLMES 都在图中使用超关系，把源数据与提取出的实体连接起来，这与我们的方法类似。然而，这些方法缺少在动态环境中更新的机制，而这正是 AriGraph 的关键优势。

**表 2：AriGraph 记忆在多跳问答数据集上展现出有竞争力的性能。** 即使在非交互任务中，AriGraph 也可与强问答基线智能体相比。原表以粗体和下划线分别标出使用基础 GPT-4o 与 GPT-4o-mini 时的最佳结果。

| 方法 | MuSiQue EM | MuSiQue F1 | HotpotQA EM | HotpotQA F1 |
|---|---:|---:|---:|---:|
| BM25（top-3） | 25.0 | 31.1 | 45.7 | 58.5 |
| Ada-002（top-3） | 24.5 | 32.1 | 45.0 | 58.1 |
| GPT-4 完整上下文 | 33.5 | 42.7 | 53.0 | 68.4 |
| GPT-4 + 支持事实 | 45.0 | 56.0 | 57.0 | 73.8 |
| ReadAgent（GPT-4） | 35.0 | 45.1 | 48.0 | 62.0 |
| GraphReader（GPT-4） | 38.0 | 47.4 | 55.0 | 70.0 |
| HOLMES（GPT-4） | 48.0 | 58.0 | 66.0 | 78.0 |
| AriGraph（GPT-4） | 45.0 | 57.0 | **68.0** | 74.7 |
| GraphRAG（GPT-4o-mini） | **40.0** | **53.5** | 58.7 | 63.3 |
| AriGraph（GPT-4o-mini） | 36.5 | 47.9 | **60.0** | **68.6** |

## 6 相关工作

Voyager [Wang et al., 2023a]、Ghost in the Minecraft [Zhu et al., 2023] 和 Jarvis-1 [Wang et al., 2023b] 是先进的开放式 LLM 智能体，在 Minecraft 中的表现显著优于早期技术。这些智能体通过已学习技能库、成功动作摘要，以及保存成功任务执行计划的情景记忆来获得记忆能力。然而，它们不能以语义结构表示知识，并高度依赖 LLM 掌握的大量 Minecraft 知识，甚至依赖对 Minecraft Wiki 的访问。生成式智能体 [Park et al., 2023] 在多智能体环境中模仿人类行为，是最早引入高级 LLM 智能体记忆系统的工作之一；我们也将其作为基线。Reflexion [Shinn et al., 2023] 和 CLIN [Majumder et al., 2023] 使智能体能够反思过去的轨迹，并把与已完成动作有关的见解存入长期记忆，但它们既没有结构化知识表示，也没有结构化情景记忆。LARP [Yan et al., 2023] 使用了情景记忆和语义记忆的概念，却把二者视为相互独立的实例，而且缺少结构化知识表示。

相当多的研究致力于利用既有知识图谱增强问答 [Baek et al., 2023; Li et al., 2024b]，以解决 LLM 中存在的事实知识缺陷。近期在问答任务上表现最佳的研究包括 GraphReader [Li et al., 2024a]、HOLMES [Panda et al., 2024]、HippoRAG [Gutiérrez et al., 2024] 和 GraphRAG [Edge et al., 2024]；它们都采用了从文本构建知识图谱的技术。然而，这些研究没有涉及在交互环境中运行的情境，也未考虑新经验所触发的知识图谱更新。

文字环境 [Côté et al., 2018; Hausknecht et al., 2019; Shridhar et al., 2021; Wang et al., 2022] 最初用于评估强化学习（RL）智能体 [Guo et al., 2020; Yao et al., 2020; Ammanabrolu et al., 2020; Ammanabrolu and Hausknecht, 2020; Tuli et al., 2022; Adhikari et al., 2021]。已有多项实验探索 LLM 在这些复杂场景中的潜力 [Tsai et al., 2023; Tan et al., 2023; Momennejad et al., 2023; Ding et al., 2024a]。然而，如果没有适当的智能体架构和记忆，原始 LLM 在这些游戏中的表现很差。

## 7 结论

本文提出 AriGraph，一种为 LLM 智能体量身定制的新型知识图谱世界模型。AriGraph 以独特方式融合从文本观察中获得的语义记忆和情景记忆，提供结构化、动态的知识表示。我们在一系列交互式文字游戏和多跳问答基准上评估该方法，并与现有记忆架构比较。为全面检验其能力，我们开发了名为 Ariadne 的认知架构，把 AriGraph 与规划和决策组件结合起来。

结果表明，在部分可观察环境中需要长期记忆的决策、规划和探索任务上，AriGraph 显著优于其他记忆系统。AriGraph 提供的结构化知识表示支持高效检索和推理，从而加快学习与任务完成。AriGraph 还展现了良好的可扩展性：当任务复杂度提高、涉及更多物体和地点时，它仍能保持较高性能。在多跳问答基准上，AriGraph 也表现出竞争力，说明其稳健性与适应能力并不局限于交互环境。

尽管前景可观，我们的方法仍可通过引入多模态观察、程序性记忆以及更复杂的图搜索方法进一步增强。

## 参考文献

> 为保证作者、题名与检索信息准确，参考文献按原文保留，不翻译论文题名。

- [Adhikari et al., 2021] Ashutosh Adhikari, Xingdi Yuan, Marc-Alexandre Côté, Mikuláš Zelinka, Marc-Antoine Rondeau, Romain Laroche, Pascal Poupart, Jian Tang, Adam Trischler, and William L. Hamilton. *Learning dynamic belief graphs to generalize on text-based games*, 2021.
- [Ammanabrolu and Hausknecht, 2020] Prithviraj Ammanabrolu and Matthew Hausknecht. *Graph constrained reinforcement learning for natural language action spaces*, 2020.
- [Ammanabrolu et al., 2020] Prithviraj Ammanabrolu, Ethan Tien, Matthew Hausknecht, and Mark O. Riedl. *How to avoid being eaten by a grue: Structured exploration strategies for textual worlds*, 2020.
- [Baek et al., 2023] Jinheon Baek, Alham Fikri Aji, and Amir Saffari. *Knowledge-augmented language model prompting for zero-shot knowledge graph question answering*, 2023.
- [Basu et al., 2024] Kinjal Basu, Keerthiram Murugesan, Subhajit Chaudhury, Murray Campbell, Kartik Talamadupula, and Tim Klinger. *Explorer: Exploration-guided reasoning for textual reinforcement learning*, 2024.
- [Bulatov et al., 2022] Aydar Bulatov, Yuri Kuratov, and Mikhail S. Burtsev. *Recurrent memory transformer*, 2022.
- [Bulatov et al., 2024] Aydar Bulatov, Yuri Kuratov, Yermek Kapushev, and Mikhail S. Burtsev. *Scaling transformer to 1m tokens and beyond with RMT*, 2024.
- [Chen et al., 2024] Jianlv Chen, Shitao Xiao, Peitian Zhang, Kun Luo, Defu Lian, and Zheng Liu. *BGE M3-embedding: Multi-lingual, multi-functionality, multi-granularity text embeddings through self-knowledge distillation*, 2024.
- [Cheng et al., 2024] Yuheng Cheng, Ceyao Zhang, Zhengwen Zhang, Xiangrui Meng, Sirui Hong, Wenhao Li, Zihao Wang, Zekai Wang, Feng Yin, Junhua Zhao, and Xiuqiang He. *Exploring large language model based intelligent agents: Definitions, methods, and prospects*, 2024.
- [Côté et al., 2018] Marc-Alexandre Côté, Ákos Kádár, Xingdi Yuan, Ben Kybartas, Tavian Barnes, Emery Fine, James Moore, Ruo Yu Tao, Matthew Hausknecht, Layla El Asri, Mahmoud Adada, Wendy Tay, and Adam Trischler. *TextWorld: A learning environment for text-based games*. CoRR, abs/1806.11532, 2018.
- [Ding et al., 2024a] Peng Ding, Jiading Fang, Peng Li, Kangrui Wang, Xiaochen Zhou, Mo Yu, Jing Li, Matthew Walter, and Hongyuan Mei. *MANGO: A benchmark for evaluating mapping and navigation abilities of large language models*, 2024.
- [Ding et al., 2024b] Yiran Ding, Li Lyna Zhang, Chengruidong Zhang, Yuanyuan Xu, Ning Shang, Jiahang Xu, Fan Yang, and Mao Yang. *LongRoPE: Extending LLM context window beyond 2 million tokens*, 2024.
- [Edge et al., 2024] Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt, and Jonathan Larson. *From local to global: A graph RAG approach to query-focused summarization*, 2024.
- [Gu and Dao, 2023] Albert Gu and Tri Dao. *Mamba: Linear-time sequence modeling with selective state spaces*, 2023.
- [Guo et al., 2020] Xiaoxiao Guo, Mo Yu, Yupeng Gao, Chuang Gan, Murray Campbell, and Shiyu Chang. *Interactive fiction game playing as multi-paragraph reading comprehension with reinforcement learning*, 2020.
- [Gutiérrez et al., 2024] Bernal Jiménez Gutiérrez, Yiheng Shu, Yu Gu, Michihiro Yasunaga, and Yu Su. *HippoRAG: Neurobiologically inspired long-term memory for large language models*, 2024.
- [Hausknecht et al., 2019] Matthew Hausknecht, Prithviraj Ammanabrolu, Côté Marc-Alexandre, and Yuan Xingdi. *Interactive fiction games: A colossal adventure*. CoRR, abs/1909.05398, 2019.
- [Izacard et al., 2022] Gautier Izacard, Mathilde Caron, Lucas Hosseini, Sebastian Riedel, Piotr Bojanowski, Armand Joulin, and Edouard Grave. *Unsupervised dense information retrieval with contrastive learning*, 2022.
- [Jeurissen et al., 2024] Dominik Jeurissen, Diego Perez-Liebana, Jeremy Gow, Duygu Cakmak, and James Kwan. *Playing NetHack with LLMs: Potential and limitations as zero-shot agents*, 2024.
- [Küttler et al., 2020] Heinrich Küttler, Nantas Nardelli, Alexander Miller, Roberta Raileanu, Marco Selvatici, Edward Grefenstette, and Tim Rocktäschel. *The NetHack learning environment*. Advances in Neural Information Processing Systems, 33:7671-7684, 2020.
- [Lee et al., 2024] Kuang-Huei Lee, Xinyun Chen, Hiroki Furuta, John Canny, and Ian Fischer. *A human-inspired reading agent with gist memory of very long contexts*, 2024.
- [Li et al., 2024a] Shilong Li, Yancheng He, Hangyu Guo, Xingyuan Bu, Ge Bai, Jie Liu, Jiaheng Liu, Xingwei Qu, Yangguang Li, Wanli Ouyang, Wenbo Su, and Bo Zheng. *GraphReader: Building graph-based agent to enhance long-context abilities of large language models*, 2024.
- [Li et al., 2024b] Yihao Li, Ru Zhang, Jianyi Liu, and Gongshen Liu. *An enhanced prompt-based LLM reasoning scheme via knowledge graph-integrated collaboration*, 2024.
- [Majumder et al., 2023] Bodhisattwa Prasad Majumder, Bhavana Dalvi Mishra, Peter Jansen, Oyvind Tafjord, Niket Tandon, Li Zhang, Chris Callison-Burch, and Peter Clark. *CLIN: A continually learning language agent for rapid task adaptation and generalization*, 2023.
- [Momennejad et al., 2023] Ida Momennejad, Hosein Hasanbeig, Felipe Vieira Frujeri, Hiteshi Sharma, Nebojsa Jojic, Hamid Palangi, Robert Ness, and Jonathan Larson. *Evaluating cognitive maps and planning in large language models with CogEval*. In Thirty-seventh Conference on Neural Information Processing Systems, 2023.
- [Murugesan et al., 2020] Keerthiram Murugesan, Mattia Atzeni, Pavan Kapanipathi, Pushkar Shukla, Sadhana Kumaravel, Gerald Tesauro, Kartik Talamadupula, Mrinmaya Sachan, and Murray Campbell. *Text-based RL agents with commonsense knowledge: New challenges, environments and baselines*, 2020.
- [Pan et al., 2024] Shirui Pan, Linhao Luo, Yufei Wang, Chen Chen, Jiapu Wang, and Xindong Wu. *Unifying large language models and knowledge graphs: A roadmap*. IEEE Transactions on Knowledge and Data Engineering, pages 1-20, 2024.
- [Panda et al., 2024] Pranoy Panda, Ankush Agarwal, Chaitanya Devaguptapu, Manohar Kaul, and Prathosh A. P. *HOLMES: Hyper-relational knowledge graphs for multi-hop question answering using LLMs*, 2024.
- [Parisotto et al., 2020] Emilio Parisotto, Francis Song, Jack Rae, Razvan Pascanu, Caglar Gulcehre, Siddhant Jayakumar, Max Jaderberg, Raphael Lopez Kaufman, Aidan Clark, Seb Noury, et al. *Stabilizing transformers for reinforcement learning*. In International Conference on Machine Learning, pages 7487-7498. PMLR, 2020.
- [Park et al., 2023] Joon Sung Park, Joseph C. O'Brien, Carrie J. Cai, Meredith Ringel Morris, Percy Liang, and Michael S. Bernstein. *Generative agents: Interactive simulacra of human behavior*, 2023.
- [Pleines et al., 2022] Marco Pleines, Matthias Pallasch, Frank Zimmer, and Mike Preuss. *Memory Gym: Partially observable challenges to memory-based agents*. In The Eleventh International Conference on Learning Representations, 2022.
- [Shinn et al., 2023] Noah Shinn, Federico Cassano, Edward Berman, Ashwin Gopinath, Karthik Narasimhan, and Shunyu Yao. *Reflexion: Language agents with verbal reinforcement learning*, 2023.
- [Shridhar et al., 2021] Mohit Shridhar, Xingdi Yuan, Marc-Alexandre Côté, Yonatan Bisk, Adam Trischler, and Matthew Hausknecht. *ALFWorld: Aligning text and embodied environments for interactive learning*. In Proceedings of the International Conference on Learning Representations (ICLR), 2021.
- [Sorokin et al., 2022] Artyom Sorokin, Nazar Buzun, Leonid Pugachev, and Mikhail Burtsev. *Explain my surprise: Learning efficient long-term memory by predicting uncertain outcomes*. Advances in Neural Information Processing Systems, 35:36875-36888, 2022.
- [Sumers et al., 2024] Theodore R. Sumers, Shunyu Yao, Karthik Narasimhan, and Thomas L. Griffiths. *Cognitive architectures for language agents*, 2024.
- [Tan et al., 2023] Qinyue Tan, Ashkan Kazemi, and Rada Mihalcea. *Text-based games as a challenging benchmark for large language models*, 2023.
- [Trivedi et al., 2022] Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, and Ashish Sabharwal. *MuSiQue: Multihop questions via single-hop question composition*. Transactions of the Association for Computational Linguistics, 2022.
- [Tsai et al., 2023] Chen Feng Tsai, Xiaochen Zhou, Sierra S. Liu, Jing Li, Mo Yu, and Hongyuan Mei. *Can large language models play text games well? Current state-of-the-art and open questions*, 2023.
- [Tuli et al., 2022] Mathieu Tuli, Andrew C. Li, Pashootan Vaezipoor, Toryn Q. Klassen, Scott Sanner, and Sheila A. McIlraith. *Learning to follow instructions in text-based games*, 2022.
- [Wang et al., 2022] Ruoyao Wang, Peter Jansen, Marc-Alexandre Côté, and Prithviraj Ammanabrolu. *ScienceWorld: Is your agent smarter than a 5th grader?*, 2022.
- [Wang et al., 2023a] Guanzhi Wang, Yuqi Xie, Yunfan Jiang, Ajay Mandlekar, Chaowei Xiao, Yuke Zhu, Linxi Fan, and Anima Anandkumar. *Voyager: An open-ended embodied agent with large language models*, 2023.
- [Wang et al., 2023b] Zihao Wang, Shaofei Cai, Anji Liu, Yonggang Jin, Jinbing Hou, Bowei Zhang, Haowei Lin, Zhaofeng He, Zilong Zheng, Yaodong Yang, Xiaojian Ma, and Yitao Liang. *Jarvis-1: Open-world multi-task agents with memory-augmented multimodal language models*, 2023.
- [Wang et al., 2024] Lei Wang, Chen Ma, Xueyang Feng, Zeyu Zhang, Hao Yang, Jingsen Zhang, Zhiyuan Chen, Jiakai Tang, Xu Chen, Yankai Lin, Wayne Xin Zhao, Zhewei Wei, and Jirong Wen. *A survey on large language model based autonomous agents*. Frontiers of Computer Science, 18(6), March 2024.
- [Wong Gonzalez, 2018] Daniela Wong Gonzalez. *The Relationship Between Semantic and Episodic Memory: Exploring the Effect of Semantic Neighbourhood Density on Episodic Memory*. PhD thesis, Electronic Theses and Dissertations, 2018. Paper 7585.
- [Yan et al., 2023] Ming Yan, Ruihao Li, Hao Zhang, Hao Wang, Zhilan Yang, and Ji Yan. *LARP: Language-agent role play for open-world games*, 2023.
- [Yang et al., 2018] Zhilin Yang, Peng Qi, Saizheng Zhang, Yoshua Bengio, William W. Cohen, Ruslan Salakhutdinov, and Christopher D. Manning. *HotpotQA: A dataset for diverse, explainable multi-hop question answering*, 2018.
- [Yao et al., 2020] Shunyu Yao, Rohan Rao, Matthew Hausknecht, and Karthik Narasimhan. *Keep calm and explore: Language models for action generation in text-based games*, 2020.
- [Yao et al., 2023] Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik Narasimhan, and Yuan Cao. *ReAct: Synergizing reasoning and acting in language models*, 2023.
- [Zhu et al., 2023] Xizhou Zhu, Yuntao Chen, Hao Tian, Chenxin Tao, Weijie Su, Chenyu Yang, Gao Huang, Bin Li, Lewei Lu, Xiaogang Wang, Yu Qiao, Zhaoxiang Zhang, and Jifeng Dai. *Ghost in the Minecraft: Generally capable agents for open-world environments via large language models with text-based knowledge and memory*, 2023.

## 附录 A 记忆图搜索细节

AriGraph 中 `SemanticSearch` 函数的伪代码见算法 2。该算法接近广度优先搜索（BFS），主要区别在于函数 `EmbedAndRetrieve` 中使用了检索机制。函数 `EmbedAndRetrieve(E,q,w)` 使用预训练 Contriever [Izacard et al., 2022]，计算边 $E$ 和查询 $q$ 的嵌入，然后返回相似度得分最高的前 $w$ 条边。边 $e$ 与查询 $q$ 的相似度，是二者嵌入的点积。

多数情况下，如果传给 `EmbedAndRetrieve` 的查询是一个语义顶点，它会直接返回与该顶点关联的边；但它也能检索连接到与查询顶点语义相近之顶点的边。例如，语义图可能分别包含 `grill` 和 `grilling` 两个并未直接相连的顶点；此时搜索 `grill`，仍可能返回边 (`bbq`, `used for`, `grilling`)。

**算法 2：语义搜索**

```text
输入：搜索查询 q，Es，搜索深度 d，搜索宽度 w
输出：相关语义顶点 Vs^q 与边 Es^q

Es^q ← ∅
L ← ∅                         // 初始化空查询队列
Enqueue(L, q)                 // 将 q 压入队列 L
D[q] ← 0                      // 将 q 的搜索距离设为 0
while L is not empty do
    q' ← Dequeue(L)           // 从 L 移除第一个元素
    if D[q'] ≥ d then
        continue
    end
    // 用 Contriever 找到最接近 q' 的前 w 个三元组
    Es' ← EmbedAndRetrieve(Es, q', w)
    foreach ei in Es' do
        Vs' ← IncidentVertices(ei)  // 返回两个关联的语义顶点
        foreach v in Vs' do
            if v not in L then
                Enqueue(L, v)
                D[v] ← D[q'] + 1
            end
        end
    end
    Es^q ← Es^q ∪ Es'
end
return Es^q
```

## 附录 B 探索

在进行任何规划或决策之前，一个辅助智能体会依据预先建立的计划评估是否需要探索，并根据评估结果开启或关闭探索模式。此外，图智能体会提取包含出口信息的三元组，例如（`kitchen`, `has an unexplored exit`, `south`），以及描述地点间空间连接的三元组，例如（`hall`, `east of`, `kitchen`）。随后，可以使用简单的图方法，识别智能体已经发现但尚未探索的各房间出口。该信息随后加入智能体的工作记忆。算法 3 给出了寻找“可能对应未探索动作的三元组”的函数。当前实现依赖专家知识，以判断语义图中的哪些元素可以表示地点及其间的出口。

**算法 3：未探索出口检测**

```text
输入：Vs，Es，Ve，Ee，当前位置 vl
输出：包含从 vl 出发之未探索出口信息的三元组 Es^exp

E^exp ← ∅
E^out ← GetOutgoing(Es, vl)              // 取得从 vl 出发的语义边
foreach e in E^out do
    if RepresentExit(e) then
        E^exp ← E^exp ∪ {e}
E^in ← GetIncoming(Es, vl)               // 取得进入 vl 的语义边
foreach e in E^in do
    if RepresentExit(e) then
        // 移除通向已探索地点的出口
        E^exp ← E^exp \ {FindRelatedExit(E^exp, e)}
return E^exp
```

## 附录 C 图统计

使用图时，最好能评估图的构建质量。遗憾的是，由于图的构建具有多样性（例如使用同义词、对连接进行等价重构等），直接测量它与真实图之间的对应程度极其困难。此外，所构建图中出现真实图没有的信息，并不总是有害，有时甚至会产生正面作用。基于这些原因，我们没有测量图本身的质量，而是测量游戏过程中图的增长率和更新率（图 5）。

我们还进行了一项独立实验：图仍按照本流水线构建，但智能体的动作随机选择（图 6）。该设置采用清洁环境，因为与寻宝不同，它包含智能体可交互的多种物体；与烹饪不同，智能体又不会失败。在该设置中，我们同样测量了图的增长率和更新率。

结果表明，图在探索阶段增长最为活跃；当智能体在已经熟悉的环境区域中行动时，图仍会增长，但速度较慢。此外，构建图所用 LLM 的质量越高，图的增长率越低。

**图 6：** 随机游走过程中图的构建与更新统计。

## 附录 D Token 用量

**表 3：文字游戏与问答任务的 token 用量分析。**

| 方法 | 提示 token | 补全 token |
|---|---:|---:|
| **文字游戏（每一步）** |  |  |
| Ariadne 智能体 | 6,000 | 500 |
| RAG 记忆智能体 | 4,000 | 350 |
| 摘要记忆智能体 | 3,800 | 350 |
| 完整历史记忆智能体（第 150 步） | 14,000 | 350 |
| Simulacra 记忆智能体 | 7,500 | 400 |
| Reflexion 记忆智能体 | 5,800 | 350 |
| **问答基准（每项任务）** |  |  |
| AriGraph | 11,000 | 2,500 |
| GraphRAG | 115,000 | 20,000 |

## 附录 E LLM 提示词

> 下列提示词翻译自论文原文。变量占位符、分隔符和要求的输出结构保持不变；三元组示例中的实体和关系保留英文，以免改变其可执行语义。

### 三元组提取提示词

```text
构建知识图谱的准则：

创建节点与三元组：节点应当表示实体或概念，类似于 Wikipedia
节点。使用如下结构化三元组格式捕捉数据：
"subject, relation, object"。
例如，根据“Albert Einstein, born in Germany, is known for
developing the theory of relativity”，提取：
"Albert Einstein, country of birth, Germany;
Albert Einstein, developed, Theory of Relativity."

请记住，应把 "John, position, engineer in Google" 这样的复杂
三元组拆分成 "John, position, engineer" 和
"John, work at, Google" 这样的简单三元组。

三元组长度不得超过 7 个单词。只能提取具体知识；任何假设都必须
表述为假说。例如，根据 "John have scored many points and
potentially will be winner"，应提取
"John, scored many, points; John, could be, winner"，
而不应提取 "John, will be, winner"。

请记住，宾语和主语必须是原子单元，而关系可以更复杂、更长。

如果观察表明你拿起了某个物品，三元组应当是：
'item, is in, inventory'，且不要提取其他内容。

不要遗漏重要信息。如果观察是“book involves story about knight,
who needs to kill a dragon”，则三元组应当是
'book, involves, knight'、'knight, needs to kill, dragon'。
如果观察涉及某种笔记，不要忘记加入关于该笔记所包含实体的三元组。

观察中相隔较远的部分之间也可能存在联系。例如，如果观察开头提供了
你所在地点的信息，结尾又说明东边有一个出口，则应提取三元组：
'location, has exit, east'。

可以提取多个包含同一节点信息的三元组。例如：
'kitchen, contains, apple'、'kitchen, contains, table'、
'apple, is on, table'。不要遗漏这种联系。

其他三元组示例：'room z, contains, black locker';
'room x, has exit, east'; 'apple, is on, table';
'key, is in, locker'; 'apple, to be, grilled';
'potato, to be, sliced'; 'stove, used for, frying';
'recipe, requires, green apple'; 'recipe, requires, potato'。

不要加入说明智能体当前位置的三元组，例如
'you, are in, location'。

不要使用 'none' 作为实体之一。

如果有信息表明你读取了某些内容，不要忘记加入三元组，以表明
你所读取的实体包含你提取出的信息。

你之前提取的三元组示例：{example}

观察：{observation}

请记住，三元组必须按如下格式提取：
"subject_1, relation_1, object_1;
subject_2, relation_2, object_2; ..."

提取出的三元组：'''
```

### 过时三元组选择提示词

```text
这些三元组表示玩家所处环境中的事实。玩家会采取动作，环境也会
变化，因此，已有三元组列表中的某些三元组可以由一个新三元组替换。
例如，玩家从储物柜中取出一个物品后，已有三元组
"item, is in, locker" 应由新三元组
"item, is in, inventory" 替换。

有时没有需要替换的三元组：
已有三元组示例："Golden locker, state, open";
"Room K, is west of, Room I"; "Room K, has exit, east"。
新三元组示例："Room T, is north of, Room N";
"Room T, has exit, south"。
替换示例：[]。此处无需替换任何内容。

有时，一个新三元组可以替换多个三元组：
已有三元组示例："kitchen, contains, broom";
"broom, is on, floor"。
新三元组示例："broom, is in, inventory"。
替换示例：
[["kitchen, contains, broom" -> "broom, is in, inventory"],
 ["broom, is on, floor" -> "broom, is in, inventory"]]。
原因是扫帚的位置已经从厨房地板变为玩家的物品栏。

只有当三元组包含关于实体同一方面的冗余或冲突信息时，才应替换。
如果已有三元组与新三元组相比提供了不同或互补的实体信息，则不应
替换。具体而言，应比较各三元组描述的关系、属性或上下文，确认它们
指向同一方面。如果不确定是否应当替换，优先保留已有三元组。比较
已有三元组和新三元组时，如果它们涉及实体的不同方面或属性，就不要
替换。只有已有三元组和新三元组在语义上重复时，才应替换。

已有三元组示例："apple, to be, cooked"、
'knife, used for, cutting'、'apple, has been, sliced'。
新三元组示例："apple, is on, table"、
'kitchen, contains, knife'、'apple, has been, grilled'。
替换示例：[]。此处无需替换。这些三元组描述物品的不同属性，因此
不应替换。

另一个不应替换已有三元组的示例：
已有三元组示例："brush, used for, painting"。
新三元组示例："brush, is in, art class"。
替换示例：[]。此处无需替换。这些三元组描述刷子的不同属性，因此
不应替换。

再次强调：如果三元组携带关于实体的不同类型信息，就不要替换！！！
保留一个三元组，好过替换掉包含重要信息的三元组。如果没有把握，
就不要声称某个三元组需要替换！！！

如果在 Existing triplets 中发现某个三元组，在语义上与 New triplets
中的某个三元组重复，则用后者替换前者。但如果三元组指向不同事物，
则不要替换。

####
只生成替换关系，不需要任何说明。
Existing triplets: {ex_triplets}.
New triplets: {new_triplets}.
####
警告！替换关系必须严格按以下格式生成：
[[outdated_triplet_1 -> actual_triplet_1],
 [outdated_triplet_2 -> actual_triplet_2], ...]
回答中绝不能包含任何说明。
Replacing:
```

### 探索检查提示词

```text
####
INSTRUCTION:
你将获得智能体计划中的子目标及其理由。你的任务是判断这些子目标
是否需要探索环境、寻找或定位某个事物。
仅回答 True 或 False。
####
Plan:
{plan0}
```

### 规划提示词

```text
####
INSTRUCTION:
你是智能体系统中的规划器，负责在文字游戏中导航环境。
你的职责是制定一个简洁计划以实现主要目标，或根据收到的新信息修改
当前计划。

确保子目标有助于实现主要目标。如果主要目标是一个持续进行的复杂
过程，还应加入能够立即推进主要目标的子目标。

如果需要寻找某物，请把它写进子目标。

如果你希望修改或删除当前计划中的某个子目标，应先根据当前观察确认
该子目标已经完成，或确认它不再与主要目标相关。在此之前，不要更改
计划中 "sub_goal" 元素的措辞或位置。只能修改 "reason" 部分，
以跟踪子目标的完成进度。

如果某个子目标已完成，或确认它不再相关，则删除该目标，以新目标或
计划中优先级较低的子目标替换。在此之前，保持子目标的既有结构。
只有当新子目标有助于主要目标时才创建它，且不要把新目标置于当前
子目标之前。

如果任务要求取得某物，只有确认该物品已在你的物品栏中之后，才能
更改相应子目标。

计划中包含你必须完成的重要信息和目标。尚未完成时，不要更改子目标，
也不要调整它们在层级中的位置！

设置子目标时，请留意物品栏以及你正携带哪些物品；它们可能很重要。

请留意记忆模块提供的信息；这些信息很重要。

计划中始终应至少有一个子目标。

在每个子目标的 "reason" 中说明该子目标的完成进度。

严格按照以下 JSON 格式作答：
{ "main_goal": "...",
  "plan_steps": [{
      "sub_goal_1": "...",
      "reason": "..."
    },
    {
      "sub_goal_2": "...",
      "reason": "..."
    },
    {
      "sub_goal_...": "...",
      "reason": "..."
    }],
  "your_emotion":
    {
      "your_current_emotion": "emotion",
      "reason_behind_emotion": "..."
    }}

不要输出任何其他内容。
####
1. Main goal: {main_goal}
2. History of {n_prev} last observations and actions:
   {observations}
3. Your current observation: {observation}
4. Information from the memory module that can be relevant
   to current situation: {subgraph}
5. Your {topk_episodic} most relevant episodic memories
   from the past for the current situation: {top_episodic}.
6. Your previous plan: {plan0}
*if is explore* 7. Yet unexplored exits in the environment:
   {all_unexpl_exits}
```

### ReAct 决策提示词

```text
####
INSTRUCTION:
你是智能体系统中的动作选择器；该系统被设计用于在文字游戏中导航
环境。你的职责是接收有关智能体和环境状态的信息，以及一个可选动作
列表。

你的首要目标，是从可选动作列表中选出符合计划所列目标的动作，并按
目标在计划中的顺序确定优先级（主要目标优先级最高，其次依次为
sub_goal_1、sub_goal_2 等）。不过，如果某个子目标在当前情境下只需
执行一个动作即可解决（例如“拿起某物”），应优先于长期子目标。

"go to 'location'" 之类的动作会把智能体直接移至指定地点。如果目标
地点距离超过一步，应使用这种动作，而不是 "go_west" 一类的动作。

如果任务以探索或寻找某物为中心，应优先选择把智能体引向尚未探索
区域的动作。你可以根据观察历史和记忆模块的信息，推断哪些地点已经
访问过。

重复执行同一动作通常不会产生不同结果；如果卡住，请尝试其他动作，
或调整目标优先级以探索环境。

只能从可选动作列表中选择动作，而且必须严格选择一个动作。

严格按照以下 JSON 格式作答：
{
  "reason_for_action": "reason"
  "action_to_take": "selected action"
}

不要输出任何其他内容。
####
1. Main goal: {main_goal}
2. History of {n_prev} last observations and actions:
   {observations}
3. Your current observation: {observation}
4. Information from the memory module that can be relevant
   to current situation: {subgraph}
5. Your {topk_episodic} most relevant episodic memories
   from the past for the current situation: {top_episodic}.
6. Your current plan: {plan0}
7. Yet unexplored exits in the environment: {all_unexpl_exits}
Possible actions in current situation: {valid_actions}
```

### 摘要提示词

```text
####
INSTRUCTION:
你是参与文字游戏的一组智能体中的向导。你的职责是以简洁但完整的
方式，说明当前情境的全部关键方面。应聚焦相关细节、排除无关信息，
使摘要有助于提取信息和进行决策。在叙述中加入战略视角，突出制定
战术计划所不可或缺的信息。

准确传达先前尝试动作的结果，因为这对于后续选择至关重要。你的叙述
将成为决策智能体开展工作的唯一依据，因此必须清晰，并避免可能造成
混淆的表达。

谨慎作出推断，只提供证据充分且可能具有实际帮助的信息。叙述应简洁，
最多三段。
####
1. Main goal: {main_goal}
2. History of {n_prev} last observations and actions:
   {observations}
3. Your current observation: {observation}
4. Your previous summary: {summary}
Your summary:
```

## 附录 F 文字游戏

### 寻宝

主要目标是打开金色储物柜，取出藏在其中的宝藏。游戏由多个房间组成，每个房间都有一个颜色不同的储物柜。游戏开始时，玩家会在第一个房间找到一把钥匙，以及说明这把钥匙能打开哪个储物柜的指示。之后，每个储物柜都包含另一把钥匙和一张纸条，纸条指向下一把钥匙的位置，由此形成一条最终通往金色储物柜的线索与发现链。

简单版本包含 12 个房间和 4 把钥匙；困难版本包含 16 个房间和 5 把钥匙，但第二把钥匙明显更难找到；最难版本包含 36 个房间、7 把钥匙，并在每个房间加入额外物品作为噪声。智能体每捡起一把钥匙得 1 分，完成游戏再得 1 分。寻宝环境示例见图 7、8、9。

### 清洁

目标是清理一栋由 9 个不同房间组成的房屋，每个房间都有特定用途，例如泳池、厨房等。每个房间都包含物品，其中一些放错了位置，例如出现在餐厅里的牙刷。共有 11 件放错位置的物品。智能体的目标是识别这些物品，并将其送回正确位置，从而整理房屋。该任务要求智能体运用推理能力判断物品的正确位置，有效地从记忆中回忆各物品应放在哪里，同时管理多个任务。

智能体每捡起一件错放物品得 1 分，每把一件物品放到正确位置得 1 分，对本已放置正确的物品进行操作则扣 1 分。本任务在概念上类似 TextWorld Commonsense（TWC）基准 [Murugesan et al., 2020]。不过，TWC 主要关注一个或至多两个地点内的逻辑推理，而我们强调环境探索和基于过往观察的记忆测试。因此，我们大幅增加了场景中的房间和物品数量。清洁环境示例见图 10。

### 烹饪

目标是制作并食用一道菜。智能体首先需要找到一份食谱，其中包含详细说明，例如所需食材，以及切丁、切片、剁碎、烧烤、煎炸和烘烤等具体处理方式。中等难度任务包含 9 个地点和 3 种食材，困难任务包含 12 个地点和 4 种食材。正确选择食材并执行正确流程可得分。如果智能体执行任何错误动作，或对某种食材使用了不合适的工具，则游戏判负。

我们在初始观察中补充了说明，并解释了特定动作，以使任务适配 LLM，尤其说明处理食材时应如何正确使用家用厨具。例如，应使用烧烤架（BBQ）进行烧烤，使用炉灶进行煎炸（提示词见附录 E）。这可以检验智能体记住并遵循说明的能力。烹饪环境示例见图 11、12。

### RL 对比基准

为了与 RL 基线比较，我们在 [Adhikari et al., 2021] 烹饪测试的一个变体上检验本架构。这些任务分为 4 个难度级别，但明显比我们的主要任务简单，包含的地点、食材和必需动作更少：

- 第 1 级：1 种食材、1 个房间、切割。
- 第 2 级：1 种食材、1 个房间、切割 + 烹调。
- 第 3 级：1 种食材、9 个房间、无需加工。
- 第 4 级：3 种食材、6 个房间、切割 + 烹调。

在每个任务难度下，我们都用 3 个随机生成的环境，测试本智能体和使用完整历史的原始 GPT-4。我们对任务作了少量调整，使其适合 LLM：在游戏的第一条观察中，我们说明，应使用烧烤架、烤箱和炉灶等适当厨具，对食材进行烧烤、烘烤或煎炸。RL 智能体可以通过与环境交互学会这些规则，但这些规则不属于常识；没有这项说明，LLM 智能体和人类玩家都无法完成游戏。尽管这一调整应当不会影响难度级别，但需要注意，用于比较 LLM 与 RL 模型的环境并非完全相同。

## 附录 G 补充图表

本节给出不同智能体和人类玩家逐步变化的性能动态（图 14），并用表 4 汇总寻宝的三个版本（中等、困难、最难）、烹饪的三个版本（中等、困难、最难）以及清洁任务的全部实验结果。

**图 7：** 寻宝环境。

**图 8：** 困难寻宝环境。

**图 9：** 最难寻宝环境。

**图 10：** 清洁环境。

**图 11：** 烹饪环境。

**图 12：** 困难烹饪环境。

**图 13：** NetHack 层级示例。

**表 4：TextWorld 环境中全部任务的归一化得分。** “-”表示原表未报告结果。

| 方法 | 寻宝 | 困难寻宝 | 最难寻宝 | 烹饪 | 困难烹饪 | 最难烹饪 | 清洁 |
|---|---:|---:|---:|---:|---:|---:|---:|
| 完整历史 | 0.47 | - | - | 0.18 | - | - | 0.05 |
| 摘要 | 0.33 | 0.17 | - | 0.52 | 0.21 | - | 0.35 |
| RAG | 0.33 | 0.17 | - | 0.36 | 0.17 | - | 0.39 |
| Reflexion | 0.93 | - | - | 1.0 | - | - | 0.27 |
| Simulacra | 0.4 | - | - | 0.3 | - | - | 0.7 |
| AriGraph | **1.0** | **1.0** | **1.0** | **1.0** | **1.0** | **0.65** | **0.79** |
| AriGraph（无探索） | 0.87 | - | - | 0.87 | - | - | 0.76 |
| AriGraph（无情景记忆） | 1.0 | 0.67 | - | 0.64 | 0.45 | - | 0.92 |
| AriGraph LLaMA-3-70B | 0.47 | - | - | 0.67 | - | - | 0.5 |
| 人类 Top-3 | 1.0 | - | - | 1.0 | - | - | 1.0 |
| 全部人类 | 0.96 | - | - | 0.32 | - | - | 0.59 |

结果表明，采用 AriGraph 的智能体显著优于全部基线，并能很好地扩展到更大、更复杂的环境。一个重要结果是，本智能体在文字游戏中展现出接近人类的表现；此前尚未有使用 LLM 的方法达到这一水平。

**图 14：测试游戏中的性能动态。** 在寻宝中，Ariadne 的表现与顶尖玩家相当；在清洁任务中略微落后；在烹饪中，其完成速度超过顶尖人类玩家。各任务的困难版本表明，随着任务难度提高，Ariadne 的表现并未下降，同时也凸显了情景记忆的重要性。

## Sources

- [原论文 PDF](<AriGraph - Learning Knowledge Graph World Models with Episodic Memory for LLM Agents.pdf>)
