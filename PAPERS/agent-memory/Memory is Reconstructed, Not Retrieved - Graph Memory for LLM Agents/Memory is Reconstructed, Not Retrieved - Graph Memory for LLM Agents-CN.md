# 记忆是重构出来的，而非检索出来的：面向 LLM 智能体的图记忆

## 译文说明

本译文依据本地 PDF 全文翻译，保留原文的章节结构、公式与编号、定理与证明、算法、表格数值、图表编号、引用及提示词字段名。模型名、数据集名、工具名和 JSON 键名保留英文，以避免改变技术含义；参考文献保留原文，便于准确检索。图中承载方法含义的关键文字已译出。原文中个别名称或符号不一致之处按原文保留，并以译注说明。

Shuo Ji$^1$，Yibo Li$^1$，Bryan Hooi$^1$

$^1$ 新加坡国立大学，新加坡。通讯作者：Yibo Li `<liyibo@u.nus.edu>`。

第 43 届国际机器学习大会（ICML 2026），韩国首尔。PMLR 306，2026。版权归作者所有。

arXiv:2606.06036v1 [cs.AI]，2026 年 6 月 4 日

代码：https://github.com/Ji-shuo/MRAgent

## 摘要

尽管近期取得了进展，LLM 智能体在基于长交互历史进行推理时仍然举步维艰。当前的记忆增强型智能体依赖静态的“先检索、后推理”范式；这种僵化的流水线设计使其无法根据推理过程中发现的中间证据动态调整记忆访问。为弥合这一差距，我们提出 MRAgent：一个将联想记忆图与主动重构机制相结合的框架。我们将记忆表示为“线索–标签–内容”（Cue–Tag–Content）图，其中联想标签充当语义桥梁，将细粒度线索与记忆内容连接起来。在这一结构之上，我们的主动重构机制把 LLM 推理直接融入记忆访问，使智能体能够依据累积证据，迭代探索并剪除检索路径。这样既可确保记忆检索动态适应推理上下文，又能避免无约束扩展造成的组合爆炸。在 LoCoMo 和 LongMemEval 基准上的实验表明，相较于强基线，该方法取得了显著提升（最高达 23%），同时大幅降低了词元与运行时间成本，凸显了主动、联想式重构在长程记忆推理中的有效性。

## 1 引言

LLM 呈现出一种“参差不齐”的认知能力谱系（Hendrycks et al., 2025）：它们擅长数学与推理，却不善于处理需要长期记忆的任务，例如历经长时间交互的互动式辅助或决策支持系统（Gao et al., 2026）。在这类长程任务中，LLM 从根本上受限于有限的上下文窗口；该限制削弱了它们随时间保留交互历史的能力（Hatalis et al., 2023）。

为缓解这些限制，先前研究为 LLM 智能体配备了外部记忆系统。早期方法采用检索增强生成（Retrieval-Augmented Generation，RAG）（Lewis et al., 2020），通过在非结构化文本或嵌入存储上执行基于相似度的检索来访问记忆。后续工作引入了更结构化的记忆表示，包括分层存储（Fang et al., 2025；Kang et al., 2025）和知识图谱（knowledge graph，KG）（Rasmussen et al., 2025；Xu et al., 2025；Huang et al., 2025）；这些表示显式编码实体与关系，以支持更具可解释性、更加关系化的检索。然而，这些系统中的检索仍局限于固定的 top-$k$ 选择或预定义的子图遍历，既不能推断新的检索线索，也无法适应中间证据。在这种形式下，现有记忆系统实际上执行的是被动检索策略。

与之相反，认知神经科学把记忆检索理解为主动且联想式的重构过程（Rugg & Renoult, 2025），而非被动读出已存储的内容。具体而言，检索由上下文线索启动；这些线索经由中间表征传播，逐步重构出连贯的记忆体验（Frankland & Josselyn, 2019）。

> **图 1 内文字：** 左侧对比被动检索（直接检索）与主动重构（迭代推理 + 检索）。在“Caroline 四年前从哪个国家搬来？”的示例中，被动检索只得到“Caroline 四年前离开了祖国”，因而无法作答；主动重构识别“祖国”这一缺失信息并继续搜索，最终取得“Caroline 的祖国是瑞典”，回答“瑞典”。右侧对比扁平记忆（无显式结构）与联想记忆（语义关联 + 结构）。后者通过共享标签“business”把“dance studio”与“Jon's business”联系起来，从直接证据与联想推断共同得到答案。

**图 1：MRAgent 中被动检索与主动记忆重构的比较。**

如图 1 所示，这一视角凸显了基于 LLM 的记忆系统面临的两个关键挑战。**挑战 1：主动重构。** 如何将记忆访问从一次性检索转变为主动的多步重构过程，在连续推理步骤中逐渐揭示信息？**挑战 2：联想记忆结构。** 如何组织记忆，使其能够捕捉语义依赖与结构依赖，并支持沿联想条目开展有引导的探索？

基于这一认识，我们提出面向 LLM 智能体的记忆推理架构（Memory Reasoning Architecture for LLM Agents，MRAgent）。该框架通过把 LLM 推理融入记忆访问，使 LLM 智能体能够执行主动、联想式的记忆重构。MRAgent 在记忆图上探索多条候选检索路径，根据中间证据剪除无关分支，并迭代选择信息增益最高的下一步。为实现受控的记忆遍历，我们设计了“线索–标签–内容”记忆图，其中标签编码细粒度线索与特定记忆内容之间的关联。通过显式暴露这些关联，标签使 LLM 能够在访问详细记忆内容之前先选择检索路径。本文贡献可概括如下：

- 我们提出主动记忆重构这一新范式，将记忆访问纳入推理过程，使智能体能够根据中间证据动态调整搜索策略。

- 我们引入“线索–标签–内容”记忆图，其中联想标签在检索过程中充当线索与内容之间的中介，使 LLM 能够识别有希望的检索路径，同时剪除无关分支。

- 我们给出理论分析，证明主动检索策略的表达能力严格强于被动检索。

- 大量实验表明，MRAgent 超越了强基线，并显著提升了词元效率与运行时间效率。

> **图 2 内文字：** 问题为“当 Nate 赢得第二次电子游戏锦标赛时，Caroline 在做什么？”（a）基于相似度的检索依据表层相关性返回多条“锦标赛”事件，但噪声很大且没有答案；（b）基于图的检索从相似种子执行固定的一跳扩展，仍只得到额外噪声；（c）主动重构先从 Nate 的事件中推断关键时间线索“7 月”，再检索该时间的 Caroline 事件，得到“Caroline 在 7 月旅行”。

**图 2：被动检索与主动重构的示例比较。** 被动检索只依据查询，检索与 Nate 的电子游戏锦标赛有关的记忆内容；主动重构则通过 LLM 推理得出关键时间线索“7 月”，并识别出 Caroline 当时的活动。

## 2 问题设定：主动记忆访问

本节将记忆检索形式化为序贯决策过程，并分析传统的被动检索范式为何无法处理复杂的多步查询。

### 2.1 记忆检索的定义

考虑由记忆单元 $V=\{v_1,\ldots,v_N\}$ 组成的外部记忆 $\mathcal{M}$。给定查询 $x$，记忆访问在 $T$ 个步骤中依次选择记忆单元。令 $S^{(t)}=\{v^{(1)},\ldots,v^{(t)}\}$ 表示第 $t$ 步之后累积的证据。我们区分两类记忆访问策略。

**被动检索。** 被动（无状态）检索策略 $\pi_p$ 仅依据查询选择记忆单元：

$$
\{v^{(1)},\ldots,v^{(T)}\}=\pi_p(x). \tag{1}
$$

**主动重构。** 主动（有状态）策略 $\pi_a^{(t)}$ 以不断演化的证据为条件选择下一个单元：

$$
v^{(t)}=\pi_a^{(t)}\!\left(x,S^{(t-1)}\right),\qquad
S^{(t)}=S^{(t-1)}\cup\{v^{(t)}\}. \tag{2}
$$

这一形式使系统能够以自适应、多步骤的方式与记忆交互。

### 2.2 被动检索的局限

我们根据记忆组织方式与适应机制，把现有记忆系统的检索策略归为两种范式，并在图 2 中将其与本文提出的主动重构范式进行比较。

**基于相似度的检索。** 许多现有记忆系统采用基于相似度的检索，例如 MemoryBank（Zhong et al., 2024）和 Mem0（Chhikara et al., 2025）。此时，检索由相似度评分函数定义：

$$
\pi_{\mathrm{sim}}(x)=\operatorname{TopK}\!\left(\{\operatorname{sim}(x,v)\}_{v\in V},k\right). \tag{3}
$$

在这一范式中，记忆充当静态上下文提供者，而相关性函数 $\operatorname{sim}(x,v)$ 由查询固定下来。如图 2(a) 所示，这类方法依据表层相关性检索出许多与“电子游戏锦标赛”有关的事件，引入大量噪声，却未能找到正确证据。

**基于图的记忆检索。** 为利用记忆单元之间的结构关系，A-Mem（Xu et al., 2025）和 Zep（Rasmussen et al., 2025）等方法引入图结构记忆。这些方法将基于相似度的种子选择与预定义的 $N$ 跳邻居扩展相结合，从而拓展检索机制：

$$
V^{\mathrm{sim}}=\operatorname{TopK}\!\left(\{\operatorname{sim}(x,v)\}_{v\in V},k\right),
$$

$$
\pi_{\mathrm{graph}}(x)=V^{\mathrm{sim}}\cup \operatorname{Neighbor}\!\left(V^{\mathrm{sim}}\right). \tag{4}
$$

其中，$\operatorname{Neighbor}(\cdot)$ 返回记忆图中某节点集合预定义的 $N$ 跳邻居。该策略虽能缓解某些多跳检索难题，却要求相关证据通过显式图边彼此连接。此外，固定邻居扩展往往会引入大量噪声。图 2(b) 表明，基于图的扩展检索到了额外却无关的邻居，但仍无法取得 Caroline 的信息，因为该信息并未在记忆图中与“电子游戏锦标赛”事件直接相连。

**主动记忆重构。** 通过把 LLM 推理直接融入记忆访问，主动重构使智能体能够推断新的检索线索，并根据累积证据调整遍历策略。如图 2(c) 所示，中间发现可被显式转化为新的检索约束，使智能体能够取得被动策略无法触达的证据。

**小结。** 被动检索的根本局限，是无法在访问记忆的同时进行推理，由此产生三个弱点：（i）不能依据中间状态修订策略，例如无法把“7 月”识别为时间锚点；（ii）固定聚合导致噪声累积；（iii）严重依赖预先构建的结构，限制了灵活性与可扩展性。附录 A 给出了详细分析。

### 2.3 来自认知记忆系统的启发

上述局限表明，我们必须超越静态检索，转向主动记忆重构范式；认知神经科学研究对此提供了有力支持（Rugg & Renoult, 2025）。如图 3 所示，人类记忆研究表明，回忆是按顺序展开的：上下文线索会触发记忆痕迹（engram）的重新激活。记忆痕迹是由既往经历形成的紧凑内部记忆状态，其激活反过来会对后续回忆产生偏置和约束，从而逐步重构出连贯的记忆（Rashid et al., 2016）。受此观点启发，我们采用“线索–标签–内容”架构：标签充当中间联想结构，编码线索如何与记忆内容相连，并引导记忆重构过程。认知神经科学还支持把记忆内容区分为两类：针对具体事件的情节记忆，以及针对共享概念与知识的语义记忆（Manns et al., 2003）。

> **图 3 内文字：** 人类大脑侧为“上下文线索 → 记忆痕迹（由线索激活的记忆索引）→ 分布式皮层再现（情节再现与语义再现）”；MRAgent 侧为“查询线索 → 标签 → 情节内容与语义内容”。二者以对应箭头相连。

**图 3：人类记忆重构与 MRAgent 架构之间的功能对应关系。**

## 3 联想记忆系统

为实现第 2 节所述的主动重构范式，我们将智能体记忆组织为异构图。本节详述核心的“线索–标签–内容”架构，以及将其组织为多粒度层的方法。

### 3.1 “线索–标签–内容”联想记忆

出于第 2 节提出的主动记忆重构需求，我们不再把记忆组织成扁平的可检索条目集合，而是构建结构化联想图。

如图 4(a) 所示，记忆构建流水线包含两个阶段：元素生成与图构建。第一阶段使用 LLM 从对话中抽取并生成记忆元素，随后利用这些元素构建记忆图。

**统一记忆系统。** 我们把记忆系统建模为异构图 $\mathcal{M}=(C,V,R)$。图中包含两类主要节点：（1）**线索** $c\in C$，即实体、属性等细粒度关键词；（2）**内容** $v\in V$，用于存储具体记忆条目。

线索与内容之间的关联被编码为带类型的关系：$R\subseteq C\times G\times V$。其中，每个三元组 $(c,g,v)\in R$ 都通过关系属性 $g$ 将线索 $c$ 连接至内容节点 $v$；该属性称为**标签**（Tag）。

**联想标签。** 标签用于概括细粒度线索与内容单元之间的联想关系。基于这些关联，LLM 执行两阶段检索：先选择一小组相关标签，再以选定标签为条件检索内容。在庞大、复杂的记忆图中，若从内容节点朴素地扩展 $n$ 跳邻居，往往会导致组合爆炸，并纳入大量无关记忆。通过引入标签这一显式联想中介，MRAgent 可以在记忆图上进行有引导且灵活的推理。标签提供语义指引，使智能体能够评估并剪除遍历分支，避免承担处理完整情节内容的计算成本。

为实现这一两阶段检索过程，我们在记忆关系 $R$ 上定义两个诱导映射算子。具体而言，$\phi_{c\to g}$ 从给定线索激活候选联想标签，$\phi_{(c,g)\to v}$ 则同时以线索和所选标签为条件检索内容：

$$
\phi_{c\to g}(c)\triangleq\{g\mid(c,g,\cdot)\in R\},
$$

$$
\phi_{(c,g)\to v}(c,g)\triangleq\{v\mid(c,g,v)\in R\}. \tag{5}
$$

这两个算子共同将联想推理与内容级检索解耦，使大规模图中的选择性、重构式记忆访问变得可行。

> **图 4 内文字：** （a）联想记忆系统：对话经“元素生成”得到线索、标签、情节和语义元素，再经“图构建”形成情节记忆与语义记忆；（b）主动记忆重构：问题经“查询线索初始化”进入状态 $S^{(t)}$，依次执行 ① LLM 推理与动作选择、② 记忆遍历、③ LLM 路由，形成状态 $S^{(t+1)}$ 和重构后的记忆，继续探索或作答。

**图 4：MRAgent——具有 LLM 驱动的主动记忆重构机制的联想记忆系统。** （a）MRAgent 从对话构建联想记忆系统，通过“线索–标签–内容”结构组织情节记忆与语义记忆；标签显式编码语义关联和关系关联。（b）收到查询后，智能体执行主动记忆重构：LLM 迭代地基于线索和标签进行推理，以遍历记忆图并选择性地重构与任务相关的记忆。

### 3.2 多粒度记忆层

受人类记忆启发，我们把记忆内容组织为涵盖具体事件、稳定知识和高层抽象的互补类型。情节记忆保留锚定于特定上下文的事件特定信息；语义记忆捕捉从多个事件中提炼出的抽象、相对稳定的知识；主题层抽象则概括多个情节共同呈现的重复模式。依据这一区分，我们把记忆图组织为多个功能层，每一层支持不同形式的推理与检索。

**情节层（线索–标签–情节）。** 情节层存储事件特定的记忆单元 $e_i\in V^e$；每个单元对应发生在特定时间的一次具体经历。情节可通过细粒度线索集合 $C_i\subseteq C$ 检索，这些线索包括实体、动作或上下文关键词；同时，由概括线索与情节内容之间联想关系的标签 $g_i$ 负责路由。为支持时间推理，情节记忆还沿统一时间线组织，使系统可以在重构过程中施加时间约束。

**语义层（线索–标签–语义）。** 语义层捕捉相对稳定的知识 $s_i\in V^s$，例如从原始对话内容中提取的个人属性、偏好和一般事实。每个语义节点均锚定于线索 $c_i$，而标签则编码该线索在方面层面的关联，例如人格特质、长期偏好或事实属性。这一设计无需检索可能很长的情节历史，便可直接访问目标语义信息。

**抽象层（主题）。** 抽象层存储主题节点 $\tau\in V^\tau$；每个节点概括一组连贯情节共同呈现的重复模式。主题与构成它的情节相连，并充当计算抽象，以支持高效的自顶向下转换 $\phi_{\tau\to e}$：智能体可以先定位相关主题，再下钻到关联情节。

这些多粒度记忆层使 MRAgent 能够灵活组合来自情节记忆的事件级证据、来自语义记忆的抽象知识，以及来自主题抽象的高层结构，从而同时支持上下文化推理和概念推理。

### 3.3 通过 LLM 蒸馏填充记忆

记忆系统通过作用于输入流 $\mathcal{T}$ 的自动蒸馏流水线进行填充。首先把输入流切分为情节单元 $e_i$，每个单元对应锚定于特定上下文的连贯事件。对于每个情节单元，使用基于 LLM 的组件提取标签和线索：

$$
g_i=F_{\mathrm{LLM}}^{\mathrm{tag}}(e_i),\qquad
C_i=F_{\mathrm{LLM}}^{\mathrm{cue}}(e_i). \tag{6}
$$

其中，$F_{\mathrm{LLM}}^{\mathrm{tag}}$ 生成简短的联想标签 $g_i$，概括该情节的关系模式；$F_{\mathrm{LLM}}^{\mathrm{cue}}$ 则提取细粒度线索集合 $C_i$，例如实体、属性和显著描述词。随后，通过标签 $g_i$ 将 $C_i$ 中的每条线索连接至情节单元 $e_i$，形成构成记忆图情节层的“线索–标签–情节”关系。语义单元以类似方式提取，生成跨单个情节持续存在的稳定抽象知识，并通过方面层标签锚定至实体层线索。

为支持更粗粒度的推理，系统通过概括相关情节的共同主题来生成主题节点，并将每个主题节点连接至其构成情节。这种分层蒸馏把输入流组织成多粒度联想结构，使智能体能够根据重构需要，在具体事件、稳定事实或高层主题的粒度上访问记忆。更多细节见附录 B.1。

## 4 MRAgent：重构式记忆智能体

本节介绍 MRAgent，它把记忆访问形式化为主动重构过程。我们首先定义重构状态与遍历动作（第 4.1 节），然后详述记忆重构过程（第 4.2 节），最后从理论上分析其表达能力（第 4.3 节）。

### 4.1 重构式记忆框架

MRAgent 不执行预定义检索流水线，而是在结构化记忆系统内部开展主动重构。我们使用显式重构状态和一组遍历动作来形式化这一过程；二者共同刻画可能的重构轨迹。

**重构状态。** MRAgent 维护显式重构状态，用于在记忆重构期间指导后续遍历方向的选择。在步骤 $t$，重构状态定义为：

$$
S^{(t)}=\left(Z^{(t)},H^{(t)}\right), \tag{7}
$$

其中，$Z^{(t)}$ 表示记忆元素（包括线索、标签和内容）的活跃集合，充当下一遍历步骤的候选项；$H^{(t)}$ 表示由此前步骤中累积的证据构成的重构上下文，用于约束后续遍历方向。

**遍历动作。** 我们定义有限的遍历动作集合 $\mathcal{A}=\{\Pi_1,\ldots,\Pi_m\}$，用于规定 LLM 在重构期间如何探索结构化记忆图。每个遍历动作均由式（5）中引入的预定义映射算子 $\phi$ 诱导，包括“线索 $\to$ 标签”“（线索，标签）$\to$ 内容”和“内容 $\to$（线索，标签）”。

前向遍历动作沿“线索–标签–内容”关系扩展活跃集合，使智能体能够以当前线索和标签为条件检索新的记忆内容。具体而言，$\Pi_{c\to g}$ 从给定线索集合 $C^{(t)}$ 激活联想标签，而 $\Pi_{(c,g)\to v}$ 同时以选定的线索和标签 $(C^{(t)},G^{(t)})$ 为条件检索记忆内容：

$$
\Pi_{c\to g}\!\left(C^{(t)}\right)
\triangleq \bigcup_{c'\in C^{(t)}}\phi_{c\to g}(c'),
$$

$$
\Pi_{(c,g)\to v}\!\left(C^{(t)},G^{(t)}\right)
\triangleq
\bigcup_{c'\in C^{(t)}}\ \bigcup_{g'\in G^{(t)}}
\phi_{(c,g)\to v}(c',g'). \tag{8}
$$

反向遍历动作允许已检索内容激活新的线索和标签，以供后续探索；智能体由此可以根据中间证据细化或重定向重构轨迹。形式化地，$\Pi_{v\to(c,g)}$ 识别与已检索内容关联的线索和标签：

$$
\Pi_{v\to(c,g)}\!\left(V^{(t)}\right)
\triangleq
\{(c',g')\mid \exists v'\in V^{(t)},\ (c',g',v')\in R\}. \tag{9}
$$

这些前向和反向遍历动作共同定义了一个可组合动作空间，使 LLM 能够在重构期间扩展并细化记忆遍历。

### 4.2 记忆重构过程

定义重构状态和遍历动作之后，下面详述记忆重构过程。该过程通过状态更新与遍历动作，迭代探索相关记忆内容。给定查询，MRAgent 首先提取一组细粒度线索，并将其与已存储线索集合匹配，得到初始活跃集合 $Z^{(0)}$ 和初始重构状态 $S^{(0)}=(Z^{(0)},\varnothing)$。从这一状态出发，MRAgent 启动由 LLM 推理、受控记忆遍历和 LLM 引导的路由构成的迭代重构循环。

**LLM 推理与动作选择。** 以重构状态为条件，LLM 针对查询 $x$ 和累积上下文 $H^{(t)}$ 进行推理，并根据当前活跃集合 $Z^{(t)}$ 选择遍历动作子集 $A^{(t)}\subseteq\mathcal{A}$：

$$
A^{(t)}=f_{\mathrm{select}}\!\left(x,H^{(t)},Z^{(t)}\right). \tag{10}
$$

其中，$f_{\mathrm{select}}$ 表示基于 LLM 的动作选择函数，它选择有希望的扩展方向，从而减少噪声。此外，以累积证据 $H^{(t)}$ 为条件，使智能体能够发现新线索并动态调整推理轨迹。

**受控记忆遍历。** 对于每个选定的遍历动作 $\Pi_a\in A^{(t)}$，记忆系统执行相应遍历算子，以生成候选节点：

$$
\widetilde{Z}^{(t+1)}=
\bigcup_{a\in A^{(t)}}\Pi_a\!\left(Z^{(t)}\right), \tag{11}
$$

其中，$\widetilde{Z}^{(t+1)}$ 是新候选集合。该步骤在 LLM 所选动作的引导下扩展遍历轨迹，而非穷举式地扩展整张图。

**LLM 路由与状态更新。** LLM 根据生成的候选项选择最相关的内容，并剪除无关分支，以确保累积上下文始终简洁、聚焦。形式化地，下一活跃集合与重构状态更新如下：

$$
Z^{(t+1)}=f_{\mathrm{route}}\!\left(x,H^{(t)},\widetilde{Z}^{(t+1)}\right),
$$

$$
H^{(t+1)}=H^{(t)}\cup Z^{(t+1)}. \tag{12}
$$

其中，$f_{\mathrm{route}}$ 是基于 LLM 的路由函数，用于评估查询、重构上下文与新检索到的记忆内容之间的语义关联。不同于表层匹配或预定义遍历，这一路由步骤会纳入记忆图所暴露的语义关联与结构关系。状态更新后，$C_{\mathrm{LLM}}$ 对累积上下文 $H^{(t+1)}$ 进行评估，判断现有证据是否足以回答查询，抑或仍需继续探索。

通过这一过程，MRAgent 把 LLM 推理直接融入多轮记忆重构。选择性扩展证据可以减少噪声、提高效率；以中间证据为条件，则使系统能够灵活调整推理轨迹。实现细节与算法见附录 B.2。

### 4.3 理论分析：主动检索与被动检索

MRAgent 被设计为一种作用于图结构记忆的主动检索方法。本节从逼近理论视角，将这种主动检索相对于被动检索的优势形式化。为保持一般性并简化符号，我们把理论从 MRAgent 的“线索、标签、情节”形式推广到任意异构图，其中每个节点都带有文本信息。

给定检索预算 $T$，主动重构会根据已检索内容，自适应地选择下一个待检索节点；被动检索器则必须仅以查询为函数，预先确定全部 $T$ 次检索。

这些检索策略诱导出不同的假设类（即输入–输出映射集合）。具体而言，令 $\mathcal{H}^{\mathrm{LM}}_{\mathrm{active}}(T)$ 包含可由某个 LM 实现的全部预测器，该 LM 能够对图执行 $T$ 次自适应检索调用；令 $\mathcal{H}^{\mathrm{LM}}_{\mathrm{passive}}(T)$ 包含 $T$ 次检索调用均预先固定（非自适应）时可实现的预测器。于是得到主要结果：

**定理 4.1（主动检索严格强于被动检索）。** 对任意检索预算 $T\ge 2$，被动假设类严格包含于主动假设类：

$$
\mathcal{H}^{\mathrm{LM}}_{\mathrm{passive}}(T)
\subsetneq
\mathcal{H}^{\mathrm{LM}}_{\mathrm{active}}(T).
$$

直观而言，采用主动检索的 LM 可以学习采用被动检索的 LM 所能学习的任意函数，反之则不成立。完整陈述与证明见附录 C。

## 5 实验

本节开展全面实验，以评估 MRAgent 并回答以下研究问题：**RQ1：** 与现有记忆系统相比，MRAgent 表现如何？**RQ2：** 就计算成本而言，MRAgent 与基线相比如何？**RQ3：** MRAgent 的不同组件各自如何影响整体性能？**RQ4：** 多轮推理如何逐步改善记忆重构？**RQ5：** MRAgent 在真实对话场景中表现如何？

### 5.1 实验设置

**基准。** 我们在两个广泛使用的长上下文记忆评估基准上评估 MRAgent：（1）LoCoMo（Maharana et al., 2024），侧重理解长对话记忆；（2）LongMemEval（Wu et al., 2025），用于跨多个会话评估长期记忆系统，并且每个查询对应更长的交互历史。

**基线。** 我们将 MRAgent 与代表性的记忆增强基线进行比较：检索增强生成（RAG）、LangMem（LangChain, 2025）、A-Mem（Xu et al., 2025）、MemoryOS（Kang et al., 2025）和 Mem0（Chhikara et al., 2025）。

**实现。** 我们使用两种 LLM 骨干评估全部方法：Gemini-2.5-Flash 和 Claude-Sonnet-4.5。遵循先前工作，我们使用 GPT-4o-mini 报告 F1 与 LLM-Judge（J）分数，并额外报告证据召回率（Recall）以供分析。

**表 1：LoCoMo 不同问题类型上的性能。** 评估指标包括 F1 分数（F1）和 LLM-Judge 分数（J）。所有数值均为百分比。粗体表示最佳结果，下划线表示次佳结果。

| 骨干 | 方法 | 多跳 F1 ↑ | 多跳 J ↑ | 时间 F1 ↑ | 时间 J ↑ | 开放域 F1 ↑ | 开放域 J ↑ | 单跳 F1 ↑ | 单跳 J ↑ | 总体 J ↑ |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Gemini | RAG | 34.89 | 58.16±0.45 | 43.52 | 49.22±0.15 | 25.68 | 41.67±0.47 | 53.69 | 69.20±0.05 | 61.30 |
| Gemini | A-Mem | 33.81 | 53.54±0.33 | 40.22 | 49.53±0.44 | 12.49 | 33.33±0.49 | 46.39 | 61.83±0.10 | 55.97 |
| Gemini | MemoryOS | 41.42 | 63.82±0.44 | 35.91 | 47.04±0.53 | 23.43 | 41.66±0.85 | <u>54.82</u> | 71.90±0.22 | 63.35 |
| Gemini | LangMem | 40.67 | 61.34±0.79 | 44.70 | 53.58±0.25 | 20.49 | 38.54±0.33 | 48.20 | 69.68±0.12 | 62.86 |
| Gemini | Mem0 | **45.17** | <u>68.79±1.21</u> | <u>58.19</u> | <u>61.68±0.29</u> | <u>26.24</u> | 41.66±1.63 | 54.37 | <u>73.72±0.10</u> | <u>68.31</u> |
| Gemini | MRAgent | <u>43.69</u> | **75.17±0.33** | **67.66** | **80.37±0.15** | **32.51** | **68.75±0.98** | **64.08** | **90.48±0.10** | **84.21** |
| Claude | RAG | 34.53 | 57.45±0.59 | 43.39 | 48.29±0.15 | 26.56 | 43.75±0.34 | 53.66 | 69.20±0.06 | 61.10 |
| Claude | A-Mem | 42.45 | 71.67±0.22 | 47.73 | <u>55.48±0.28</u> | 22.02 | 47.57±0.46 | <u>55.19</u> | 74.71±0.05 | 68.45 |
| Claude | MemoryOS | 32.94 | 60.99±0.44 | 39.14 | 51.09±0.29 | 18.29 | 48.95±0.15 | 45.46 | 66.49±0.21 | 61.18 |
| Claude | LangMem | 44.37 | 70.92±0.25 | <u>56.64</u> | <u>80.68±0.36</u> | 22.66 | 54.71±0.55 | 54.36 | <u>83.12±0.15</u> | <u>78.61</u> |
| Claude | Mem0 | <u>48.66</u> | <u>75.88±0.67</u> | 49.50 | 53.58±0.44 | <u>28.58</u> | <u>56.25±0.49</u> | 54.43 | 74.07±0.06 | 69.02 |
| Claude | MRAgent | **56.72** | **90.19±0.29** | **69.82** | **85.34±0.25** | **34.67** | **71.57±0.10** | **68.62** | **91.10±0.15** | **88.32** |

**表 2：由 LLM-Judge ↑ 评估的 LongMemEval 性能。** Gemini 骨干方法中的最佳和次佳结果分别以粗体与下划线标出。Mul.：多会话；Sgl.：单会话–用户；Tmp.：时间推理；Pref.：单会话–偏好。MRAgent$^*$ 使用 Claude 执行检索，但使用 Gemini 构建记忆。完整表格见附录 D.5。

| 方法 | Mul. | Sgl. | Tmp. | Pref. | 总体 |
|---|---:|---:|---:|---:|---:|
| RAG | 54.89 | 85.71 | 42.86 | 33.33 | 54.65 |
| A-Mem | 42.85 | <u>90.00</u> | 45.11 | 46.43 | 52.98 |
| MemoryOS | <u>56.39</u> | 87.14 | 38.35 | **46.67** | <u>54.92</u> |
| LangMem | 52.63 | 78.57 | **45.71** | 36.67 | 53.77 |
| Mem0 | 50.38 | 78.57 | 45.11 | 40.00 | 53.01 |
| MRAgent | **68.42** | **92.85** | **68.42** | **66.67** | **72.95** |
| MRAgent$^*$ | 86.46 | 92.85 | 85.71 | 78.57 | 86.76 |

更多细节见附录 D。

### 5.2 主要结果（RQ1）

**MRAgent 在不同问题类型和骨干上始终优于全部基线。** 如表 1 所示，在 Gemini 骨干下，MRAgent 将总体 J 分数从 68.31 提高到 84.21（相对提升 23.3%）；在 Claude 骨干下则实现了 12.4% 的提升。

这些提升可归因于两个关键因素。第一，“线索–标签–内容”（CTC）记忆设计通过标签显式编码联想关系，将语义信息融入记忆图结构。与纯粹沿预定义结构边扩展的图检索方法相比，CTC 记忆使 MRAgent 能够根据语义相关性选择检索方向，从而提高定位支持证据的能力。第二，MRAgent 将 LLM 推理融入多轮记忆访问，使检索方向能够适应累积证据，并逐渐形成连贯的推理链。这种自适应重构过程使智能体可以聚焦相关证据，并遵循由证据引导的检索路径，因而优于被动检索基线。

表 2 进一步表明，MRAgent 在 LongMemEval 上也取得了一致提升，相较最强基线获得 32% 的相对增益。

### 5.3 成本分析（RQ2）

表 3 报告了 LongMemEval 上每个样本的总词元消耗与运行时间。该基准旨在跨多个对话会话评估长期记忆。

**表 3：使用 Gemini 骨干时，不同记忆系统在长期 LongMemEval 上每个样本的词元消耗和运行时间。** 结果同时涵盖记忆构建与检索。粗体表示最佳结果，下划线表示次佳结果。

| 方法 | 词元消耗 | 运行时间（秒） |
|---|---:|---:|
| A-Mem | 632k | 1,122.23 |
| MemoryOS | 273k | 3,135.54 |
| LangMem | 3,268k | 1,209.57 |
| Mem0 | 245k | **533.29** |
| MRAgent | **118k** | <u>586.11</u> |

**MRAgent 通过选择性的按需记忆访问提高信息效率。** MRAgent 将提示词元降至 118k，显著低于 A-Mem（632k）等基线。现有方法在构建阶段反复总结历史、分析错综复杂的依赖关系；与之不同，MRAgent 保持轻量的构建阶段，并把复杂关系的建立推迟到检索阶段，再以特定查询为条件执行。此外，借助联想标签在语义上引导检索方向，智能体可以在访问成本高昂的情节内容之前剪除无关路径。这种“按需”方法确保计算资源严格聚焦于与查询相关的证据。

### 5.4 消融研究（RQ3）

图 5 展示 LoCoMo 多跳问题上的消融结果，分别考察记忆结构与推理机制的贡献。我们考虑三种结构变体：使用直接索引的 CE（线索 $\to$ 情节）、通过中介访问情节的 CTE（线索–标签–情节），以及采用完整记忆结构的 CTC（线索–标签–内容）。CTE 与 CTC 变体均在无推理（绿色柱）和有推理（蓝色柱）两种设置下进行评估。

**主动多步推理是取得性能增益的首要因素。** 在所有记忆结构上，有推理变体（蓝色柱）始终优于只使用结构的对应变体（绿色柱）。这表明，多步推理与遍历对于累积证据、支持多跳推断至关重要；一次性检索不足以解决复杂的多跳查询。

**联想标签为检索提供有效的语义引导。** 在无推理设置（绿色柱）下，性能从 CE 到 CTE 再到 CTC 单调提升，表明更丰富的联想结构能够带来更可靠的检索。尤其是，标签有助于把检索引向语义相关方向，减少纳入碎片化或无关的记忆单元。

**情节记忆层与语义记忆层相辅相成。** 移除语义记忆组件会导致明显的性能下降（蓝色柱）。情节记忆保留事件特定细节，而语义记忆捕捉对多跳推理不可或缺的稳定抽象知识。

**图 5：LoCoMo 多跳查询上的消融结果；使用 Claude 骨干，以 Recall 和 LLM-Judge（J）进行评估。**

### 5.5 多轮推理分析（RQ4）

如图 6(a) 所示，我们评估 LoCoMo 数据集上证据召回率随推理轮次累积的变化。图 6(b) 汇总了达到收敛所需的平均推理轮数（Average Turns），以及能够检索到有效信息的最大轮数（Max Valid Turns）。

**多轮推理逐步找回缺失证据。** 如图 6(a) 所示，单跳查询（蓝线）和时间查询（黄线）在约三轮内达到接近完美的召回率；多跳查询则从迭代探索中显著获益，其召回率随连续步骤提高了 30% 以上（红线）。这凸显了多步重构对于解决组合依赖和长距离依赖的必要性。

**智能体自主评估累积上下文，以引导搜索与终止。** 如图 6(b) 所示，Max Valid Turns 与 Average Turns 非常接近，表明 LLM 能够有效判断何时继续搜索、何时停止。这种行为最大限度地减少了冗余探索。此外，附录 D.6 的实验表明，增加并行检索预算不能取代更深的重构深度。

> **图 6(b) 数值：** 多跳：Average Turns 3.16、Max Valid Turns 2.65；时间：2.42、2.40；开放域：2.60、1.09；单跳：2.07、1.28。

**图 6：使用 Claude 骨干时，LoCoMo 上的多轮推理分析。**

### 5.6 案例研究（RQ5）

图 7 给出一个定性案例，展示 MRAgent 针对复杂、具有时间依据的查询，如何在结构化记忆图上主动重构跨会话证据。详细分析见附录 D.8。

> **图 7 内文字：** 问题为“Joanna 的哪些剧本被制作公司拒绝了？”答案为“第一部和第三部”。从线索 Joanna 出发，智能体经由“创造性工作”等标签取得“剧本投稿”情节与关于剧本的语义信息，再通过“剧本拒绝”等高层主题取得拒稿事件，并按时间线将投稿与拒稿对齐。

**图 7：MRAgent 针对多会话查询在记忆图上的推理轨迹。** 从线索“Jonna”出发，智能体遍历多个联想标签，同时取得情节记忆（例如剧本投稿）与语义信息（例如剧本背景）。随后，它针对高层主题进行推理以找回拒稿事件，并将其与此前取得的投稿事件对齐，从而对查询给出连贯答案。这里，$D_x:x$ 表示具体事件实例。

> **译注：** 图注和图内起始线索写作 “Jonna”，正文问题与附录案例写作 “Joanna”；本译文分别按原文保留。

## 6 相关工作

**检索增强生成。** RAG（Lewis et al., 2020）通过相似度搜索把外部文档注入提示来增强 LLM；其中记忆是非结构化向量存储，而检索简化为查询时的一次性 top-$k$ 选择。该范式有两种扩展。GraphRAG（Han et al., 2025）把检索语料组织为图结构，并通过社区摘要与邻域扩展执行检索；相较扁平相似度搜索，它改善了全局推理与多跳推理。智能体式 RAG 则把检索移入推理循环。Search-o1（Li et al., 2025）将大型推理模型与智能体机制相结合：一旦发现知识缺口，便按需发出搜索查询，并在将检索文档整合到推理链之前对其进行提炼；Search-R1（Jin et al., 2025）则通过针对多轮查询生成的强化学习来训练这种行为。这些方法从开放或外部语料库中检索，以填补单次推理任务中的事实知识缺口，而非在智能体自身的持久交互历史上进行重构。

**基于图的记忆。** 基于图的记忆系统把智能体记忆组织为图，以捕捉记忆单元之间的结构依赖。A-Mem（Xu et al., 2025）通过 LLM 辅助的关系抽取构建相互连接的结构化记忆笔记，并先选择种子，再执行邻域扩展以完成检索。Zep（Rasmussen et al., 2025）维护双时间知识图谱，跟踪事实的有效时间并使过时边失效，以支持从不断演化的知识中检索。LiCoMemory（Huang et al., 2025）把记忆组织为轻量级分层图，以实体和关系作为语义索引层，并借助时间感知、层级感知的搜索实现高效检索。这些表示虽改善了关系访问和多跳访问，但其遍历由预定义算子控制，不会适应检索过程中累积的证据。

**分层与持久记忆系统。** 分层记忆系统维护跨交互持续更新的长期记忆。MemoryOS（Kang et al., 2025）把记忆组织为短期、中期和长期层级，以支持结构化访问。Mem0（Chhikara et al., 2025）通过 LLM 驱动的添加、更新和删除操作，维护一组紧凑的显著事实。SeCom（Pan et al., 2025）以主题连贯的片段为粒度构建记忆库，以实现更准确的检索。这些系统推动了记忆随时间存储与更新的方式，但其检索过程仍是被动的：它们把记忆单元选取为查询的固定函数，而不针对中间证据进行推理。

## 7 结论与讨论

我们提出了重构式记忆智能体 MRAgent，将记忆访问形式化为结构化记忆图上的主动多步重构过程。MRAgent 的一项关键设计选择，是把复杂关系依赖的建模移至检索阶段，使智能体能够通过有针对性、依赖状态的探索，以较低计算成本解决更复杂的查询。因此，我们当前的实现采用了相对简单的记忆构建策略，并未在记忆更新或遗忘机制中引入额外复杂性。这项设计选择也暴露出若干局限，并提示了未来研究方向。

第一，由于关系推理被推迟到检索阶段，重构成本会随探索深度增长；需要许多遍历步骤的查询，其延迟高于一次性检索。第二，我们的静态构建不会随时间更新或整合记忆，因而记忆图会随着交互累积而单调增长，增加长期部署中的存储开销。通过自适应构建、轻量级记忆维护和更稳健的遍历策略来解决这些局限，是把主动重构扩展到更广泛长程场景的一条有前景的路径。

## 致谢

本研究得到新加坡教育部学术研究基金 Tier 2（FY2025）的资助（项目编号 T2EP20124-0038）。

## 影响声明

本工作通过引入面向长程记忆推理的主动重构范式，旨在推进大型语言模型（LLM）智能体记忆系统的设计。潜在的积极影响包括：为个人助理、决策支持系统和长期人机交互等应用提供更可靠、更具上下文感知能力的 AI 助手；在这些应用中，对既往信息的准确回忆与推理至关重要。同时，与其他记忆增强型 AI 系统一样，存储长期交互数据并据此推理的能力，也带来了隐私、数据治理和负责任部署方面的考量。这些问题并非本文方法所独有，而是普遍适用于 AI 智能体中的持久记忆。

## 参考文献

参考文献保留原文著录形式，以便准确检索。

Anokhin, P., Semenov, N., Sorokin, A., Evseev, D., Kravchenko, A., Burtsev, M., and Burnaev, E. Arigraph: Learning knowledge graph world models with episodic memory for llm agents. arXiv preprint arXiv:2407.04363, 2024.

Chhikara, P., Khant, D., Aryan, S., Singh, T., and Yadav, D. Mem0: Building production-ready AI agents with scalable long-term memory. CoRR, abs/2504.19413, 2025. doi: 10.48550/ARXIV.2504.19413. URL https://doi.org/10.48550/arXiv.2504.19413.

Fang, J., Deng, X., Xu, H., Jiang, Z., Tang, Y., Xu, Z., Deng, S., Yao, Y., Wang, M., Qiao, S., Chen, H., and Zhang, N. Lightmem: Lightweight and efficient memory-augmented generation. CoRR, abs/2510.18866, 2025. doi: 10.48550/ARXIV.2510.18866. URL https://doi.org/10.48550/arXiv.2510.18866.

Frankland, P. W. and Josselyn, S. A. The neurobiological foundation of memory retrieval. Nature Neuroscience, 22(10):1576–1585, 2019.

Gao, H., Geng, J., Hua, W., Hu, M., Juan, X., Liu, H., Liu, S., Qiu, J., Qi, X., Ren, Q., Wu, Y., Wang, H., Xiao, H., Zhou, Y., Zhang, S., Zhang, J., Xiang, J., Fang, Y., Zhao, Q., Liu, D., Qian, C., Wang, Z., Hu, M., Wang, H., Wu, Q., Ji, H., and Wang, M. A survey of self-evolving agents: What, when, how, and where to evolve on the path to artificial super intelligence. Trans. Mach. Learn. Res., 2026, 2026.

Han, H., Wang, Y., Shomer, H., Guo, K., Ding, J., Lei, Y., Halappanavar, M., Rossi, R. A., Mukherjee, S., Tang, X., He, Q., Hua, Z., Long, B., Zhao, T., Shah, N., Javari, A., Xia, Y., and Tang, J. Retrieval-augmented generation with graphs (graphrag). CoRR, abs/2501.00309, 2025. doi: 10.48550/ARXIV.2501.00309. URL https://doi.org/10.48550/arXiv.2501.00309.

Hatalis, K., Christou, D., Myers, J., Jones, S., Lambert, K., Amos-Binks, A., Dannenhauer, Z., and Dannenhauer, D. Memory matters: The need to improve long-term memory in llm-agents. In Proceedings of the AAAI Symposium Series, volume 2, pp. 277–280, 2023.

Hendrycks, D., Song, D., Szegedy, C., Lee, H., Gal, Y., Brynjolfsson, E., Li, S., Zou, A., Levine, L., Han, B., Fu, J., Liu, Z., Shin, J., Lee, K., Mazeika, M., Phan, L., Ingebretsen, G., Khoja, A., Xie, C., Salaudeen, O., Hein, M., Zhao, K., Pan, A., Duvenaud, D., Li, B., Omohundro, S., Alfour, G., Tegmark, M., McGrew, K., Marcus, G., Tallinn, J., Schmidt, E., and Bengio, Y. A definition of AGI. CoRR, abs/2510.18212, 2025. doi: 10.48550/ARXIV.2510.18212. URL https://doi.org/10.48550/arXiv.2510.18212.

Hu, Y., Liu, S., Yue, Y., Zhang, G., Liu, B., Zhu, F., Lin, J., Guo, H., Dou, S., Xi, Z., et al. Memory in the age of ai agents. arXiv preprint arXiv:2512.13564, 2025.

Huang, Z., Tian, Z., Guo, Q., Zhang, F., Zhou, Y., Jiang, D., and Zhou, X. Licomemory: Lightweight and cognitive agentic memory for efficient long-term reasoning. CoRR, abs/2511.01448, 2025. doi: 10.48550/ARXIV.2511.01448. URL https://doi.org/10.48550/arXiv.2511.01448.

Jin, B., Zeng, H., Yue, Z., Wang, D., Zamani, H., and Han, J. Search-r1: Training llms to reason and leverage search engines with reinforcement learning. CoRR, abs/2503.09516, 2025. doi: 10.48550/ARXIV.2503.09516. URL https://doi.org/10.48550/arXiv.2503.09516.

Kang, J., Ji, M., Zhao, Z., and Bai, T. Memory OS of AI agent. CoRR, abs/2506.06326, 2025. doi: 10.48550/ARXIV.2506.06326. URL https://doi.org/10.48550/arXiv.2506.06326.

LangChain. Langmem sdk for agent long-term memory. https://blog.langchain.com/langmem-sdk-launch/, 2025. Accessed: 2025-01.

Lewis, P., Perez, E., Piktus, A., Petroni, F., Karpukhin, V., Goyal, N., Küttler, H., Lewis, M., Yih, W., Rocktäschel, T., Riedel, S., and Kiela, D. Retrieval-augmented generation for knowledge-intensive NLP tasks. In Larochelle, H., Ranzato, M., Hadsell, R., Balcan, M., and Lin, H. (eds.), Advances in Neural Information Processing Systems 33: Annual Conference on Neural Information Processing Systems 2020, NeurIPS 2020, December 6-12, 2020, virtual, 2020.

Li, X., Dong, G., Jin, J., Zhang, Y., Zhou, Y., Zhu, Y., Zhang, P., and Dou, Z. Search-o1: Agentic search-enhanced large reasoning models. In Christodoulopoulos, C., Chakraborty, T., Rose, C., and Peng, V. (eds.), Proceedings of the 2025 Conference on Empirical Methods in Natural Language Processing, EMNLP 2025, Suzhou, China, November 4-9, 2025, pp. 5420–5438. Association for Computational Linguistics, 2025. doi: 10.18653/V1/2025.EMNLP-MAIN.276. URL https://doi.org/10.18653/v1/2025.emnlp-main.276.

Maharana, A., Lee, D., Tulyakov, S., Bansal, M., Barbieri, F., and Fang, Y. Evaluating very long-term conversational memory of LLM agents. In Ku, L., Martins, A., and Srikumar, V. (eds.), Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), ACL 2024, Bangkok, Thailand, August 11-16, 2024, pp. 13851–13870. Association for Computational Linguistics, 2024. doi: 10.18653/V1/2024.ACL-LONG.747. URL https://doi.org/10.18653/v1/2024.acl-long.747.

Manns, J. R., Hopkins, R. O., and Squire, L. R. Semantic memory and the human hippocampus. Neuron, 38(1):127–133, 2003.

Pan, Z., Wu, Q., Jiang, H., Luo, X., Cheng, H., Li, D., Yang, Y., Lin, C.-Y., Zhao, H. V., Qiu, L., et al. Secom: On memory construction and retrieval for personalized conversational agents. In The Thirteenth International Conference on Learning Representations, 2025.

Rashid, A. J., Yan, C., Mercaldo, V., Hsiang, H.-L., Park, S., Cole, C. J., De Cristofaro, A., Yu, J., Ramakrishnan, C., Lee, S. Y., et al. Competition between engrams influences fear memory formation and recall. Science, 353(6297):383–387, 2016.

Rasmussen, P., Paliychuk, P., Beauvais, T., Ryan, J., and Chalef, D. Zep: A temporal knowledge graph architecture for agent memory. CoRR, abs/2501.13956, 2025. doi: 10.48550/ARXIV.2501.13956. URL https://doi.org/10.48550/arXiv.2501.13956.

Rugg, M. D. and Renoult, L. The cognitive neuroscience of memory representation. Neuroscience & Biobehavioral Reviews, 155:105450, 2025.

Tan, Z., Yan, J., Hsu, I.-H., Han, R., Wang, Z., Le, L., Song, Y., Chen, Y., Palangi, H., Lee, G., et al. In prospect and retrospect: Reflective memory management for long-term personalized dialogue agents. In Proceedings of the 63rd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pp. 8416–8439, 2025.

Wu, D., Wang, H., Yu, W., Zhang, Y., Chang, K., and Yu, D. Longmemeval: Benchmarking chat assistants on long-term interactive memory. In The Thirteenth International Conference on Learning Representations, ICLR 2025, Singapore, April 24-28, 2025. OpenReview.net, 2025. URL https://openreview.net/forum?id=pZiyCaVuti.

Xu, W., Liang, Z., Mei, K., Gao, H., Tan, J., and Zhang, Y. A-MEM: agentic memory for LLM agents. CoRR, abs/2502.12110, 2025. doi: 10.48550/ARXIV.2502.12110. URL https://doi.org/10.48550/arXiv.2502.12110.

Zhong, W., Guo, L., Gao, Q., Ye, H., and Wang, Y. Memorybank: Enhancing large language models with long-term memory. In Wooldridge, M. J., Dy, J. G., and Natarajan, S. (eds.), Thirty-Eighth AAAI Conference on Artificial Intelligence, AAAI 2024, Thirty-Sixth Conference on Innovative Applications of Artificial Intelligence, IAAI 2024, Fourteenth Symposium on Educational Advances in Artificial Intelligence, EAAI 2014, February 20-27, 2024, Vancouver, Canada, pp. 19724–19731. AAAI Press, 2024. doi: 10.1609/AAAI.V38I17.29946. URL https://doi.org/10.1609/aaai.v38i17.29946.

# 附录

## A 被动检索的详细分析

本附录统一分析 LLM 智能体代表性记忆系统中的检索机制，并把所有方法对齐至第 2 节的形式。回顾一下，记忆访问作用于外部记忆 $\mathcal{M}$，其中包含单元 $V=\{v_1,\ldots,v_N\}$。给定查询 $x$，被动检索返回固定集合 $\pi_p(x)$，主动重构则通过有状态策略 $v^{(t)}=\pi_a^{(t)}(x,S^{(t-1)})$ 依次选择单元。

**检索增强生成。** 检索增强生成（RAG）采用经典的基于相似度的检索范式：外部记忆被组织为向量存储，并由单次查询访问。给定查询 $x$，RAG 计算 $x$ 与每个记忆单元 $v\in V$ 之间的相关性分数，检索相似度最高的 top-$k$ 个条目：

$$
\pi_p^{\mathrm{RAG}}(x)=
\operatorname{TopK}\!\left(\{\operatorname{sim}(x,v)\}_{v\in V},k\right). \tag{13}
$$

**A-Mem（Xu et al., 2025）。** A-Mem 在记忆单元之上引入图结构，其中节点对应结构化记忆笔记，边则编码在记忆构建期间建立的语义关系。给定查询 $x$，A-Mem 先通过相似度选择种子记忆条目，再沿记忆图扩展其预定义邻域：

$$
V^*=\operatorname{TopK}\!\left(\{\operatorname{sim}(x,v)\}_{v\in V},k\right),
$$

$$
\pi_p^{\mathrm{A\text{-}MEM}}(x)=V^*\cup\operatorname{Neighbor}(V^*). \tag{14}
$$

其中，$\operatorname{Neighbor}(V^*)=\bigcup_{v\in V^*}\operatorname{Neighbor}(v)$ 返回记忆图中与种子集合直接相连的节点。

**MemoryOS（Kang et al., 2025）。** MemoryOS 把智能体记忆组织为三层结构，包括短期记忆（STM）、中期记忆（MTM）和长期个人记忆（LPM）。给定查询 $x$，它将近期短期上下文、相关中期主题记忆与长期人物信息纳入分层检索：

$$
R_{\mathrm{MTM}}(x)=
\operatorname{TopK}\!\left(\{F_{\mathrm{score}}(x,v)\}_{v\in V_{\mathrm{MTM}}}\right),
$$

$$
R_{\mathrm{LPM}}^{\mathrm{fact}}(x)=
\operatorname{TopK}\!\left(\{F_{\mathrm{score}}(x,v)\}_{v\in V_{\mathrm{LPM}}^{\mathrm{fact}}}\right),
$$

$$
\pi_p^{\mathrm{MEMOS}}(x)=
V_{\mathrm{STM}}\cup R_{\mathrm{MTM}}(x)\cup
V_{\mathrm{LPM}}^{\mathrm{profile}}\cup
R_{\mathrm{LPM}}^{\mathrm{fact}}(x). \tag{15}
$$

其中，$V_{\mathrm{LPM}}^{\mathrm{profile}}$ 表示人物信息，$V_{\mathrm{LPM}}^{\mathrm{fact}}$ 表示以查询为条件的长期事实记忆。

**LangMem（LangChain, 2025）。** LangMem 将对话历史压缩为一组记忆摘要，把它们存入基于向量的记忆存储，并在推理时注入模型上下文，从而维护对话记忆。每个记忆单元对应一条经摘要的对话消息，并以嵌入向量表示。给定查询 $x$，LangMem 通过基于相似度的搜索检索相关记忆：

$$
\pi_p^{\mathrm{LANGMEM}}(x)=
S_{\mathrm{LLM}}\!\left(
\operatorname{TopK}\!\left(\{\operatorname{sim}(x,v)\}_{v\in V},k\right),x
\right). \tag{16}
$$

其中，$\operatorname{Summarize}_{\mathrm{LLM}}(\cdot)$ 表示基于 LLM 的摘要函数，用于把检索到的记忆条目压缩为简洁的上下文表示。

> **译注：** 原文式（16）记作 $S_{\mathrm{LLM}}$，紧随其后的解释却写作 $\operatorname{Summarize}_{\mathrm{LLM}}$；本译文按原文分别保留。

**Mem0（Chhikara et al., 2025）。** Mem0 维护持久记忆存储，其中包含借助 LLM 驱动的记忆构建与更新过程，从对话中增量提取出的自然语言事实。每条记忆对应一项显著事实；记忆演化由 LLM 选择的操作（ADD、UPDATE、DELETE、NOOP）控制，以确保随时间保持一致。给定查询 $x$，Mem0 通过相似度搜索选择最相关的记忆条目：

$$
\pi_p^{\mathrm{MEM0}}(x)=
\operatorname{TopK}\!\left(\{\operatorname{sim}(x,v)\}_{v\in V},k\right). \tag{17}
$$

**结论。** 根据上述分析，现有记忆系统主要关注记忆表示的设计，检索过程则与底层结构紧密耦合。这些方法可大致分为基于相似度的检索和基于图的检索。尽管部分方法采用结构化记忆表示，其检索过程仍然是被动的：遍历算子预先定义且固定，不会根据检索期间的中间证据进行调整。

## B 实现与执行细节

**表 4：提供受控记忆遍历操作的记忆工具包。**

| 工具 | 映射函数 | 说明 |
|---|---|---|
| `query_tag_events` | $\phi_{(c,g)\to e}(c,g)$ | 检索与“线索–标签”对关联的情节事件。 |
| `query_conversation_time` | $\phi_{e\to t}(v_e^e)$ | 返回情节事件的对话时间戳。 |
| `query_event_keywords` | $\phi_{e\to(c,g)}(e)$ | 从情节事件中提取关联的线索与标签。 |
| `query_event_context` | $\phi_{e\to\mathrm{ctx}}(e)$ | 检索情节事件周围的上下文文本。 |
| `query_personal_information` | $\phi_{c^s\to g^s}(c^s)$ | 返回与人物实体关联的语义方面。 |
| `query_personal_aspect` | $\phi_{(c^s,g^s)\to v^s}(c^s,g^s)$ | 检索“人物、方面”对的语义内容。 |
| `query_topic_events` | $\phi_{\tau\to e}(\tau)$ | 检索与主题节点关联的情节事件。 |

### B.1 记忆构建流水线

**线索–标签–情节。** 为实例化记忆图元素并显式建模其联想关系，我们使用 LLM 提取构建“线索–标签–情节”三元组所需的组件。给定原始对话文本 $\mathcal{T}$，先对其重写、提炼，以消解上下文依赖并显式化线索，再把处理后的文本切分为连贯的情节单元：

$$
\{e_i\}\leftarrow R_{\mathrm{LLM}}(\mathcal{T}), \tag{18}
$$

其中，$R_{\mathrm{LLM}}$ 执行原始文本处理，包括代词消解、时间规范化和情节切分。对于每个情节片段 $e_i$，我们使用基于 LLM 的抽取器生成一个联想标签和一组细粒度线索：

$$
g_i\leftarrow T_{\mathrm{LLM}}(x_i),\qquad
C_i\leftarrow K_{\mathrm{LLM}}(x_i), \tag{19}
$$

其中，$T_{\mathrm{LLM}}$ 生成简短而精确的短语，概括该情节的核心语义模式或关系模式；$K_{\mathrm{LLM}}$ 提取线索集合 $C_i$，其中包括实体、属性和其他显著描述词。随后，我们为 $x_i$ 与每个 $c\in C_i$ 添加节点，并通过标签 $g_i$ 将其关联起来，形成“线索–标签–情节”关系。

> **译注：** 式（18）及段首把情节片段记为 $e_i$，式（19）和随后一句却使用 $x_i$；本译文按原文保留该符号差异。

**线索–标签–语义。** 语义记忆通过提供超越单个情节的、相对稳定的抽象，来补充情节重构（Anokhin et al., 2024；Hu et al., 2025）。每个语义单元表示为 $(c_i^s,g_i^s,s_i)$，其中 $s_i$ 表示语义内容，$c_i^s$ 是实体层线索，$g_i^s$ 是方面层标签。我们通过基于 LLM 的语义抽取函数，从输入文本 $\mathcal{T}$ 中提取这类语义单元：

$$
\{(c_i^s,g_i^s,s_i)\}\leftarrow S_{\mathrm{LLM}}(\mathcal{T}), \tag{20}
$$

其中，$S_{\mathrm{LLM}}$ 识别 $\mathcal{T}$ 中表达的稳定事实与属性，将其概括为语义内容 $s_i$，并为其分配实体层线索 $c_i^s$ 和方面层标签 $g_i^s$。提取出的 $(c_i^s,g_i^s,s_i)$ 单元被插入同一记忆图，并通过“线索–标签–语义”关联相连。

**高层主题记忆。** 标签捕捉情节层语义模式，主题则概括跨情节反复出现的模式，为多粒度推理提供更高层的组织。这一设计符合如下观点：高阶结构可从重复的情节经历中抽象出来（Pan et al., 2025；Tan et al., 2025）。具体而言，我们对情节片段集合应用基于 LLM 的抽象函数，以获得主题节点：

$$
\{\tau_j\}\leftarrow A_{\mathrm{LLM}}(\{e_i\}), \tag{21}
$$

其中，每个主题节点 $\tau_j$ 概括一组共享重复语义模式的连贯情节。随后，我们添加“主题–情节”链接，将每个 $\tau_j$ 与其关联情节相连；当不需要细粒度情节遍历时，智能体由此可以在更高层进行推理。

### B.2 可执行的记忆重构

图 8 详细展示了记忆重构过程，并给出示例图与伪代码。为实现多步推理轨迹，我们设计了专用工具包，用于执行 LLM 确定的遍历动作，并从记忆图返回新检索到的证据。

> **图 8 左侧流程：** 查询“当 Nate 赢得第三次电子游戏锦标赛时，他如何使用奖金？”先匹配线索 Nate 与“电子游戏锦标赛”。第 1 步调用 `query_tag_events` 找到 Nate 的多次锦标赛事件；第 2 步调用 `query_conversation_time` 确认第三次比赛的时间；第 3 步调用 `query_event_keywords` 取得该事件的关键词与标签；第 4 步调用 `query_tag_event`（图内如此命名）取得奖金用途事件；最终回答“Nate 把钱存了起来”。

**算法 1：主动记忆重构**

```text
输入：查询 x，记忆图 G，遍历动作 A={Π₁,…,Πₘ}，最大步数 T
输出：答案 ŷ

 1  C ← EXTRACTCUES(x)
 2  Z⁽⁰⁾ ← ACTIVESETINIT(C,G)
 3  H⁽⁰⁾ ← ∅
 4  S⁽⁰⁾ ← (Z⁽⁰⁾,H⁽⁰⁾)
 5  for t = 0 to T−1 do
 6      // 动作选择
 7      A⁽ᵗ⁾ ← f_select(x,H⁽ᵗ⁾,Z⁽ᵗ⁾)  // A⁽ᵗ⁾ ⊆ A
 8      // 受控遍历
 9      Z̃⁽ᵗ⁺¹⁾ ← ∅
10      for all a ∈ A⁽ᵗ⁾ do
11          Z̃⁽ᵗ⁺¹⁾ ← Z̃⁽ᵗ⁺¹⁾ ∪ Πₐ(Z⁽ᵗ⁾)
12      end for
13      // 路由与状态更新
14      Z⁽ᵗ⁺¹⁾ ← f_route(x,H⁽ᵗ⁾,Z̃⁽ᵗ⁺¹⁾)
15      H⁽ᵗ⁺¹⁾ ← H⁽ᵗ⁾ ∪ Z⁽ᵗ⁺¹⁾
16      if STOP(x,H⁽ᵗ⁺¹⁾) then
17          break
18      end if
19  end for
20  ŷ ← ANSWER_LLM(x,H⁽ᵗ⁺¹⁾)
21  return ŷ
```

**图 8：左：给定查询时记忆重构的示例。右：记忆重构算法的伪代码。**

MRAgent 在两种执行模式下运行：$\Psi\in\{\texttt{Navigate},\texttt{Answer}\}$。在 Navigate 模式中，智能体调用工具包，逐步探索记忆图并累积证据。收集到充分证据后，智能体转入 Answer 模式，以重构上下文为条件生成最终回答。

表 4 汇总了用于记忆遍历与证据获取的工具包。每个工具对应记忆组件之间的带类型映射，使 LLM 能够显式控制记忆访问的方向与粒度。智能体不会检索固定的记忆集合，而是把这些工具组合为串行或并行调用，逐步重构相关上下文。在每个导航步骤中，LLM 根据当前重构状态选择并调用一个或多个工具。返回结果被视为候选证据，由 LLM 评估，以决定扩展、细化还是终止某条遍历路径。这种显式的工具接口既保证记忆访问可解释、可控制，又允许智能体根据中间证据调整检索策略。

## C 异构知识图谱记忆上的主动检索与被动检索：理论分析

MRAgent 被设计为一种作用于图结构记忆的主动检索方法。本节从形式上说明，相比被动（非自适应）检索，这种主动（自适应）检索在逼近理论上的优势。

### C.1 设定

考虑查询 $x\in\mathcal{X}$、答案 $y$ 来自标签空间 $\mathcal{Y}$ 的任务。

MRAgent 使用包含线索、标签和情节的异构知识图谱（KG）记忆。为保持一般性并简化符号，我们将理论推广到任意异构知识图谱，其中每个节点都带有文本信息。

**异构知识图谱记忆。** 对于每个问题规模 $n$，记忆 $M\in\mathcal{M}_n$ 是一张异构知识图谱，其节点集合为：

$$
\mathcal{V}(M)=\{v_1,v_2,\ldots,v_n\}.
$$

每个节点 $v\in\mathcal{V}(M)$ 在某个类型集合中拥有类型 $\tau(v)$，并具有文本载荷 $p(v)\in\mathcal{P}$。该知识图谱还拥有有限的关系标签集合 $\mathcal{R}$（例如标签类型）。

**节点检索算子。** LM 通过检索算子与知识图谱交互；给定节点后，该算子同时返回（i）节点载荷与（ii）长度有界的出邻居列表。

形式化地，固定分支上界 $B\ge 1$。对于每个记忆 $M$，定义：

$$
\operatorname{Retrieve}^{M}:\mathcal{V}(M)
\to \mathcal{P}\times\left(\mathcal{R}\times\mathcal{V}(M)\right)^{\le B}.
$$

给定节点 $v$，算子返回：

$$
\operatorname{Retrieve}^{M}(v)=\bigl(p(v),\operatorname{Out}(v)\bigr),
$$

其中，$p(v)\in\mathcal{P}$ 是载荷，$\operatorname{Out}(v)$ 是至多包含 $B$ 条出边 $(r,v')$ 的列表；$r$ 是边的关系，$v'$ 是目标节点。该抽象与常见知识图谱实现相符：一次“节点获取”会返回属性/文本，以及数量有界的邻居引用。

### C.2 总体风险与逼近误差

**定义 C.1（总体风险）。** 对预测器 $\pi$ 和分布 $D$，定义 0–1 损失下的总体风险：

$$
L(\pi;D):=
\mathbb{E}_{(M,x,y)\sim D}
\left[\mathbf{1}[\pi(x,M)\ne y]\right].
$$

**定义 C.2（逼近误差）。** 对假设类 $\mathcal{H}$ 和分布 $D$，定义：

$$
\operatorname{opt}(\mathcal{H};D):=
\inf_{\pi\in\mathcal{H}}L(\pi;D).
$$

即便策略没有学到任何关于 $y$ 的信息，仍可以根据先验猜测。为刻画 0–1 损失下的这一基线，定义：

$$
\varepsilon_{\mathcal{Y}}:=1-\sup_{y\in\mathcal{Y}}P_{\mathcal{Y}}(y).
$$

直观而言，当只知道先验 $P_{\mathcal{Y}}$、没有关于 $y$ 的额外信息时，$\varepsilon_{\mathcal{Y}}$ 是可达到的最小误差。

### C.3 主动检索与被动检索

现在，给定参数为 $\theta\in\Theta$、检索预算为 $T$ 步的 LM，定义主动策略与被动策略。LM 诱导出函数 $Q_{\theta,t}$（$t\le T$），用于决定第 $t$ 步检索哪个节点；还诱导出最终答案头 $H_\theta$，用于根据检索历史输出答案。

在主动策略下，$Q_{\theta,t}$ 以截至当前观察到的全部历史为条件；在被动策略下，它仅以查询为条件。令 $v^{(t)}$ 表示 LM 在步骤 $t$ 选择检索的节点。形式化定义如下。

**定义 C.3（主动 LM + 检索策略类）。** 参数为 $\theta$ 的主动策略执行：

$$
v^{(1)}=Q_{\theta,1}(x),\qquad
z^{(1)}=\operatorname{Retrieve}^{M}(v^{(1)}),
$$

并且对 $t=2,\ldots,T$，

$$
v^{(t)}=Q_{\theta,t}\!\left(x,z^{(1)},\ldots,z^{(t-1)}\right),\qquad
z^{(t)}=\operatorname{Retrieve}^{M}(v^{(t)}).
$$

其预测为：

$$
\pi_\theta^{\mathrm{act}}(x,M)=
H_\theta\!\left(x,z^{(1)},\ldots,z^{(T)}\right).
$$

定义假设类：

$$
\mathcal{H}^{\mathrm{LM}}_{\mathrm{active}}(T)
:=\{\pi_\theta^{\mathrm{act}}:\theta\in\Theta\}.
$$

**定义 C.4（被动 LM + 检索策略类）。** 参数为 $\theta$ 的被动策略执行：

$$
v^{(t)}=Q_{\theta,t}^{\mathrm{pass}}(x),\qquad t=1,\ldots,T,
$$

随后对所有 $t$ 检索 $z^{(t)}=\operatorname{Retrieve}^{M}(v^{(t)})$，并输出：

$$
\pi_\theta^{\mathrm{pass}}(x,M)=
H_\theta\!\left(x,z^{(1)},\ldots,z^{(T)}\right).
$$

定义：

$$
\mathcal{H}^{\mathrm{LM}}_{\mathrm{passive}}(T)
:=\{\pi_\theta^{\mathrm{pass}}:\theta\in\Theta\}.
$$

### C.4 主定理：主动检索严格强于被动检索

我们证明：允许检索器根据此前检索到的内容调整查询，会产生一个严格大于任意非自适应（仅查询）检索器的预测器类。

**定理 C.5（主动检索严格强于被动检索）。** 对任意检索预算 $T\ge 2$，被动假设类严格包含于主动假设类：

$$
\mathcal{H}^{\mathrm{LM}}_{\mathrm{passive}}(T)
\subsetneq
\mathcal{H}^{\mathrm{LM}}_{\mathrm{active}}(T).
$$

首先注意到，被动策略是主动策略的子集，因此主动假设类至少与被动假设类一样强。

**引理 C.6。** 对任意检索预算 $T$，有：

$$
\mathcal{H}^{\mathrm{LM}}_{\mathrm{passive}}(T)
\subseteq
\mathcal{H}^{\mathrm{LM}}_{\mathrm{active}}(T).
$$

**证明。** 固定 $T$ 以及任意被动策略 $\pi_\theta^{\mathrm{pass}}\in\mathcal{H}^{\mathrm{LM}}_{\mathrm{passive}}(T)$。根据定义，其检索查询形式为 $v^{(t)}=Q_{\theta,t}^{\mathrm{pass}}(x)$，仅依赖查询 $x$。定义主动策略 $\widetilde{\pi}_\theta^{\mathrm{act}}$，其查询为：

$$
\widetilde{v}^{(t)}=
\widetilde{Q}_{\theta,t}\!\left(x,z^{(1)},\ldots,z^{(t-1)}\right)
:=Q_{\theta,t}^{\mathrm{pass}}(x),
\qquad t=1,\ldots,T.
$$

换言之，该主动策略忽略检索历史，发出与被动策略相同的节点序列。使用同一个答案头 $H_\theta$，所得预测函数对所有 $(x,M)$ 都满足 $\widetilde{\pi}_\theta^{\mathrm{act}}(x,M)=\pi_\theta^{\mathrm{pass}}(x,M)$。证毕。

**尚需证明的内容。** 为证明任意 $T\ge 2$ 时定理 1 的严格性，只需构造一个依赖于 $T$ 的分布 $D$，使主动检索可在预算 $T$ 下达到零误差，而相同预算下任意被动策略的误差都有一个严格大于零的下界。下一小节将证明这一点。

> **译注：** 此处原文称“定理 1”，但本附录中的对应结果编号为定理 C.5；本译文保留原文说法。

### C.5 分离任务族：二叉树“大海捞针”

下面构造一个具有所需性质的任务。直观而言，查询标识根节点。从根节点执行检索会显示两个候选子节点；根节点载荷包含一个比特，用于指明应沿哪个子节点继续。重复 $d$ 跳后可到达唯一目标叶节点，其载荷包含答案标签。主动检索能够依次遵循所显示的比特，从而达到零误差；被动检索却必须猜测叶节点，因而产生不可约误差。

**二叉树。** 构造深度为 $d=T-1$ 的完全二叉树。根节点记为 $v_\varnothing$。每个内部节点 $v_u$ 由二进制串 $u$ 索引，该二进制串表示从根到当前节点的路径：从根出发，每个比特表示选择左（0）子节点还是右（1）子节点。因此，每个内部节点 $v_u$ 有两个子节点，分别记为 $v_{u0}$ 与 $v_{u1}$。

**真实叶节点。** 从 $\{0,1\}^d$ 中均匀随机采样目标叶索引 $u^*$，并令相应叶节点 $v_{u^*}$ 为目标节点。把 $u^*=(u_1^*,\ldots,u_d^*)$ 的各比特解释为从根出发的真实路径：从 $v_\varnothing$ 出发，在深度 $t$ 处沿 $u_t^*\in\{0,1\}$ 所索引的子节点前进，故唯一的根–叶路径为：

$$
v_\varnothing\to v_{u_1^*}\to v_{u_1^*u_2^*}
\to\cdots\to v_{u_1^*\cdots u_d^*}=v_{u^*}.
$$

为使主动检索能够发现这条路径，我们把目标路径的下一个比特编码在路径节点的载荷中：对每个满足 $|u|=t<d$ 的前缀 $u=u_1^*\cdots u_t^*$，

$$
p(v_u)\text{ 编码下一个比特 }u_{t+1}^*.
$$

**答案标签。** 采样 $y\sim P_{\mathcal{Y}}$。答案标签仅编码在目标叶节点的载荷中：

$$
p(v_{u^*})\text{ 编码 }y.
$$

返回的其他全部信息（非目标节点的载荷与出边列表）均独立于 $y$。特别地，除非策略检索到 $v_{u^*}$，否则除了先验之外，它得不到任何关于 $y$ 的信息。

**查询。** 查询 $x$ 标识根节点 $v_\varnothing$，并询问标签 $y$。

**定义 C.7（二叉树“大海捞针”分布 $D_{n,d}$）。** 固定 $d\ge 1$，并设：

$$
n=1+2+\cdots+2^d=2^{d+1}-1.
$$

三元组 $(M,x,y)\in\mathcal{M}_n\times\mathcal{X}_n\times\mathcal{Y}$ 上的分布 $D_{n,d}$ 定义如下：

1. 采样目标叶索引 $u^*\sim\operatorname{Unif}(\{0,1\}^d)$。

2. 采样 $y\sim P_{\mathcal{Y}}$。

3. 按上述二叉树构建异构知识图谱记忆 $M\in\mathcal{M}_n$：

   - 对每个满足 $t<d$ 的前缀 $u=u_1^*\cdots u_t^*$，载荷 $p(v_u)$ 编码下一个比特 $u_{t+1}^*$；

   - 载荷 $p(v_{u^*})$ 编码 $y$；

   - 其他全部节点载荷均独立于 $y$。

4. 输出查询 $x$；它标识 $v_\varnothing$，并询问存储在 $v_{u^*}$ 中的标签。

### C.6 主动检索达到零误差

**引理 C.8。** 对分布 $D_{n,d}$，存在参数设置 $\theta$ 和预算 $T=d+1$，使得：

$$
L\!\left(\pi_\theta^{\mathrm{act}};D_{n,d}\right)=0.
$$

因此，

$$
\operatorname{opt}\!\left(
\mathcal{H}^{\mathrm{LM}}_{\mathrm{active}}(d+1);D_{n,d}
\right)=0.
$$

**证明。** 考虑如下主动策略。查询 $x$ 标识根节点 $v_\varnothing$。初始化 $u:=\varnothing$。对 $t=0,1,\ldots,d-1$：

1. 检索当前节点 $v_u$，得到 $\operatorname{Retrieve}^{M}(v_u)=(p(v_u),\operatorname{Out}(v_u))$。从 $p(v_u)$ 读取下一比特值 $b\in\{0,1\}$（当 $u$ 是 $u^*$ 的长度为 $t$ 的前缀时，该值等于 $u_{t+1}^*$）。

2. 移动至相应子节点：令 $u\leftarrow ub$；即当 $b=0$ 时前往 $v_{u0}$，当 $b=1$ 时前往 $v_{u1}$。

经过 $d$ 步后，我们已到达目标叶节点 $v_{u^*}$。最后检索 $v_{u^*}$ 以读取其载荷，并输出 $y$。该过程至多使用 $d+1$ 次检索，并对每个样本 $(M,x,y)\sim D_{n,d}$ 均给出正确答案。证毕。

### C.7 除非预算呈指数增长，否则被动检索具有不可约误差

**引理 C.9。** 对预算为 $T$ 的任意被动策略，并且对所有参数 $\theta$ 一致成立：

$$
L\!\left(\pi_\theta^{\mathrm{pass}};D_{n,d}\right)
\ge
\varepsilon_{\mathcal{Y}}\left(1-\frac{T}{2^d}\right),
\qquad
\varepsilon_{\mathcal{Y}}:=1-\sup_{y\in\mathcal{Y}}P_{\mathcal{Y}}(y).
$$

**证明。** 固定 $\theta$。被动策略仅以 $x$ 为函数，在观察任何检索值之前便选定全部待检索节点 $v^{(1)},\ldots,v^{(T)}$。

在 $D_{n,d}$ 下，目标叶索引 $u^*\in\{0,1\}^d$ 服从均匀分布，且 $x$ 不会揭示它。令 $S:=\{v^{(1)},\ldots,v^{(T)}\}$ 为检索到的节点（多重）集合，令 $L_d:=\{v_u:u\in\{0,1\}^d\}$ 为叶节点集合。因为只有叶节点才可能等于目标 $v_{u^*}$，所以：

$$
\Pr[v_{u^*}\in S]
=\Pr[v_{u^*}\in S\cap L_d]
=\frac{|S\cap L_d|}{2^d}
\le\frac{T}{2^d}.
$$

标签 $y$ 仅编码在目标载荷 $p(v_{u^*})$ 中。因此，策略只有在检索 $v_{u^*}$ 时才能取得关于 $y$ 的信息；否则，整个检索记录都独立于 $y$。在后一种情况下，最佳预测只能依赖先验 $P_{\mathcal{Y}}$，并且必然产生至少为 $\varepsilon_{\mathcal{Y}}$ 的误差。因此：

$$
L\!\left(\pi_\theta^{\mathrm{pass}};D_{n,d}\right)
\ge
\Pr[\text{未命中目标}]\cdot\varepsilon_{\mathcal{Y}}
\ge
\left(1-\frac{T}{2^d}\right)\varepsilon_{\mathcal{Y}}.
$$

证毕。

### C.8 严格性与逼近理论分离

**定理 C.5 的证明。** 固定任意 $T\ge 2$，令 $d:=T-1$。包含关系 $\mathcal{H}^{\mathrm{LM}}_{\mathrm{passive}}(T)\subseteq\mathcal{H}^{\mathrm{LM}}_{\mathrm{active}}(T)$ 来自如下事实：可以把任意被动策略视为一个查询函数忽略历史、只依赖 $x$ 的主动策略（引理 C.6）。

为证明严格性，考虑定义 C.7 中的分离分布 $D_{n,d}$。根据引理 C.8，$\operatorname{opt}(\mathcal{H}^{\mathrm{LM}}_{\mathrm{active}}(T);D_{n,d})=0$（因为 $T=d+1$）。根据引理 C.9：

$$
\operatorname{opt}\!\left(
\mathcal{H}^{\mathrm{LM}}_{\mathrm{passive}}(T);D_{n,d}
\right)
\ge
\varepsilon_{\mathcal{Y}}\left(1-\frac{T}{2^d}\right)
=
\varepsilon_{\mathcal{Y}}\left(1-\frac{T}{2^{T-1}}\right).
$$

如果两个假设类相等，那么它们对每个分布 $D$ 都应具有相同的 $\operatorname{opt}(\cdot;D)$，这与上述严格间隔矛盾。因此，对每个 $T\ge2$，该包含关系都是严格的。证毕。

> **译注：** 按原文所取 $d=T-1$，当 $T=2$ 时，上述下界为 $\varepsilon_{\mathcal{Y}}(1-2/2)=0$，并未给出严格正的误差间隔。因此，这段证明按当前写法直接建立的是 $T\ge3$ 时的严格性；$T=2$ 的结论还需要额外论证。

## D 实验

### D.1 数据集

**LoCoMo（Maharana et al., 2024）。** LoCoMo 是一个旨在评估长对话场景中长期对话记忆的基准。该数据集包含通过人类–LLM 流水线生成的 50 段对话。每段对话最多跨越 35 个会话，平均长度约为 300 轮，并配有约 200 个问答对。LoCoMo 包含多种问题类别，涵盖单跳、多跳、时间和开放域查询；这些查询要求检索并整合对话历史中相距较远的部分所包含的信息。在实验中，我们排除了对抗性问题，因为大多数基线方法不支持这一设置，而且该任务主要评估检测不可回答查询的能力，而非记忆重构或多跳推理能力。

**LongMemEval（Wu et al., 2025）。** LongMemEval 是一个旨在评估聊天助手在持续的用户–助手交互下超长期记忆能力的基准。每个评估实例由一系列带时间戳的聊天会话构成，随后给出一个需要针对交互历史进行回忆与推理的问题。我们采用 LongMemEval-S 设置，其中包含约 500 个问题，每个问题对应一段约 115K 词元的聊天历史。这一设置对长期记忆提出了很有挑战性的评估。在实验中，我们主要关注单会话–用户、多会话、时间推理和单会话–偏好问题类型。

> **译注：** 上一句的问题类型标点按原文结构翻译；结合表 5，四类应分别为 `multi-session`、`single-user`、`temporal-reasoning` 和 `single-preference`。

### D.2 基线

**检索增强生成。** 检索增强生成（RAG）把对话历史作为文本块存储在向量数据库中。对于每个查询，系统根据嵌入相似度检索 top-$k$ 条最相似的记忆，并将其与查询拼接，形成输入上下文。

**A-Mem（Xu et al., 2025）。** A-Mem 为每个对话情节构建结构化记忆笔记；新的记忆笔记通过 LLM 辅助分析与已有笔记相连，从而形成有向图结构。在检索阶段，A-Mem 对查询进行嵌入以检索相关记忆笔记，再扩展至其邻居节点。

**MemoryOS（Kang et al., 2025）。** MemoryOS 把智能体记忆组织为三层结构，包括存储近期对话上下文的短期记忆、存储基于主题的交互历史的中期记忆，以及存储稳定用户与智能体特征的长期人物记忆。在检索阶段，它跨所有层执行分层记忆访问，再把检索到的记忆与人物信息拼接起来，形成输入上下文。

**LangMem（LangChain, 2025）。** LangMem 对每轮对话进行嵌入，并通过记忆管理工具将其维护在向量数据库中，从而存储对话记忆。在推理时，它根据嵌入相似度检索 top-$k$ 条最相关的记忆，并使用 LLM 对其进行摘要，以支持回答生成。

**Mem0（Chhikara et al., 2025）。** Mem0 从对话中增量提取显著的自然语言事实，并通过 LLM 驱动的更新操作维护紧凑的长期记忆。在检索阶段，它对已存储记忆执行基于相似度的搜索，并把检索到的事实纳入输入上下文，以生成答案。

### D.3 评估指标

**LLM-Judge。** 我们采用基于 LLM 的裁判来评估生成答案在语义上的正确性。给定模型生成的答案 $\hat{y}$ 与参考答案 $y$，裁判判断二者在语义上是否等价，允许存在释义和表层形式差异。裁判输出一个表示正确与否的二元判定。

**F1 分数。** 除 Recall 外，我们还报告根据裁判判定计算的 F1 分数。对于每个问题，生成答案被视为一次正类预测，裁判判定决定它是真阳性还是假阳性。对数据集进行聚合后，精确率与召回率计算如下：

$$
\operatorname{Precision}=
\frac{\sum_i J(\hat{y}_i,y_i)}
{\sum_i \mathbf{I}[\hat{y}_i\ne\varnothing]},
\qquad
\operatorname{Recall}=
\frac{\sum_i J(\hat{y}_i,y_i)}{N}, \tag{22}
$$

F1 分数定义为：

$$
\operatorname{F1}=
\frac{2\cdot\operatorname{Precision}\cdot\operatorname{Recall}}
{\operatorname{Precision}+\operatorname{Recall}}. \tag{23}
$$

该形式既惩罚错误答案，也惩罚未能生成答案的情况，尤其适合开放式问答设置。

**证据召回率（Recall）。** 证据召回率衡量在记忆重构期间成功检索到的真实支持证据所占比例。对于问题 $i$，令 $E_i^*$ 表示带标注的真实证据条目集合，$\widehat{E}_i$ 表示智能体检索到的证据条目集合。证据召回率计算为：

$$
\operatorname{Recall}=
\frac{1}{N}\sum_{i=1}^{N}
\frac{|\widehat{E}_i\cap E_i^*|}{|E_i^*|}, \tag{24}
$$

其中，$N$ 是评估问题总数。该指标独立于最终答案生成，反映检索过程找回相关支持证据的效果。

### D.4 实现

我们使用 GPT-4o-mini 作为 LLM 裁判，温度设为 0.0。每种方法独立评估三次，并报告三次运行的裁判分数均值与标准差。为确保各方法的计算预算可比，我们把智能体针对每个查询的推理限制为最多 8 轮，并允许每轮最多调用 10 次工具。如果在耗尽预算之前满足停止条件，智能体可以提前终止。

### D.5 LongMemEval 详细结果

表 5 给出了不同评估设置下 LongMemEval 的详细结果，使用 F1 和 LLM-Judge 评估。

**表 5：由 F1 与 LLM-Judge ↑ 评估的 LongMemEval 性能。** Gemini 骨干方法中的最佳和次佳结果分别以粗体和下划线标出。MRAgent$^*$ 使用 Claude 执行检索，但记忆由 Gemini 构建。

| 方法 | 多会话 F1 ↑ | 多会话 J ↑ | 单用户 F1 ↑ | 单用户 J ↑ | 时间推理 F1 ↑ | 时间推理 J ↑ | 单偏好 F1 ↑ | 单偏好 J ↑ | 总体 J ↑ |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| RAG | 45.00 | 54.89 | 77.94 | 85.71 | 43.88 | 42.86 | 5.42 | 33.33 | 54.65 |
| A-Mem | 28.26 | 42.85 | 75.30 | <u>90.00</u> | 34.18 | 45.11 | 9.09 | 46.43 | 52.98 |
| MemoryOS | <u>45.02</u> | <u>56.39</u> | <u>79.50</u> | 87.14 | <u>35.67</u> | 38.35 | <u>9.53</u> | <u>46.67</u> | <u>54.92</u> |
| LangMem | 41.14 | 52.63 | 72.43 | 78.57 | 36.80 | <u>45.71</u> | 6.32 | 36.67 | 53.77 |
| Mem0 | 37.53 | 50.38 | 72.43 | 78.57 | 35.64 | 45.11 | 5.89 | 40.00 | 53.01 |
| MRAgent | **49.92** | **68.42** | **80.99** | **92.85** | **50.16** | **68.42** | **23.96** | **66.67** | **72.95** |
| MRAgent$^*$ | 66.31 | 86.46 | 82.41 | 92.85 | 60.10 | 85.71 | 15.58 | 78.57 | 86.76 |

### D.6 多步重构的预算敏感性分析

图 9 分析了 LoCoMo 多跳问题上推理轮数与每轮并行检索之间的权衡。我们改变每轮检索预算 $K$（即单轮推理中允许的最大并行工具调用数）和最大推理轮数 $T$，同时保持其他设置不变。

**图 9：LoCoMo 多跳查询的性能随推理轮数 $T$ 和每轮检索预算 $K$ 的变化；使用 Claude 骨干，以 LLM-Judge（J）进行评估。** 图中设置为 $T\in\{2,4,8\}$，$K\in\{2,4,8\}$。

**增加并行探索不能取代重构深度。** 随着推理轮数 $T$ 增加，所有 $K$ 取值下的准确率都稳定、单调上升；重构越深，性能显著越高。与之相反，增加每轮检索预算 $K$ 只带来有限增益，并很快趋于饱和。这些结果表明，并行探索虽然增加了单轮推理中的检索广度，却无法取代多轮重构所支持的证据顺序组合。

### D.7 检索算子的证据覆盖率

为分析不同检索算子对记忆重构的贡献，我们按问题类别汇总并考察 LoCoMo 上各工具的证据覆盖率。表 6 报告了各算子的覆盖率，反映其在重构过程中的功能角色。

**不同检索算子分别擅长不同的查询结构。** 时间问题主要通过 `query_conversation_time` 解决；它覆盖了大多数具有时间依据的证据。多跳问题高度依赖 `query_tag_events` 和 `query_topic_events`，说明要找回分布在多个情节中的证据，对标签与主题执行联想扩展不可或缺。与之相反，开放域问题在多个算子上的覆盖率更加均衡，反映了整合情节信息、语义信息与上下文信息的需求。总体而言，这些结果表明 MRAgent 执行的是结构化、依赖算子的记忆重构。智能体并不依赖统一的或相似度驱动的检索，而是根据查询结构选择性激活不同算子，从而形成差异化的证据获取模式。

**表 6：使用 Claude 骨干时，不同工具在 LoCoMo 各问题类型上的证据覆盖率。**

| 工具 | 多跳 | 时间 | 开放域 | 单跳 |
|---|---:|---:|---:|---:|
| `query_tag_events` | 66.33 | 81.08 | 41.18 | 74.76 |
| `query_conversation_time` | 4.08 | 86.49 | 17.65 | 3.88 |
| `query_event_context` | 18.37 | 32.43 | 35.29 | 33.01 |
| `query_personal_aspect` | 21.43 | 2.70 | 29.41 | 5.83 |
| `query_topic_events` | 33.67 | 45.95 | 35.29 | 27.18 |

### D.8 案例研究

如图 7 所示，我们给出一个案例，说明 MRAgent 如何在结构化记忆图上执行多轮记忆重构。查询“Joanna 的哪些剧本被制作公司拒绝了？”要求把剧本投稿事件与随后发生、分布于多个对话会话中的拒稿事件联系起来。MRAgent 通过迭代式图探索和多轮推理，逐步重构相关证据。

在第一轮推理中，智能体遍历基于标签的关联，检索剧本投稿事件与拒稿事件，识别与查询直接相关的候选事件。在第二轮中，它通过查询事件级上下文与关键词来扩展这些候选项，以取得每次拒稿的详细信息。在第三轮和第四轮中，智能体查询 Joanna 的语义信息，以找回她的剧本属性，从而更清楚地刻画每次投稿。最后，在第五轮中，智能体查询时间信息，将剧本投稿与对应的拒稿事件对齐，并验证其先后顺序。

经过五步推理后，MRAgent 正确推断出 Joanna 的第一部和第三部剧本遭到拒绝。这个例子表明，在回答复杂的多会话查询时，把联想扩展与语义验证、时间验证相结合的主动多步记忆重构大有裨益。

## E 提示词

### MRAgent 问答与工具使用提示词

> 你是一个可以访问基于事件的记忆的问答智能体。如果已有充分证据，请回答问题；否则，查询记忆工具以收集更多信息。
>
> **回答规则：**
>
> - 是/否问题：输出 `Yes`、`No`、`Likely yes` 或 `Likely no`。
>
> - 地点问题：用具体的地点名称作答。
>
> - 计数问题：用相关条目的数量作答。
>
> - 其他问题：输出最简的具体实体或短语。
>
> **选择一种模式：**
>
> （1）`"answer"`——如果证据充分，输出：

```text
{
  "mode": "answer",
  "answer": "...",
  "supports": ["D1:1", "D1:2"],
  "confidence": 0.0-1.0
}
```
>
> （2）`"navigate"`——如果证据不足，直接调用所有相关的记忆工具（不作解释）。

### 关键词抽取提示词

> 你是一个信息抽取系统。只输出有效的 JSON。
>
> **任务：** 对每个输入句子，直接从原文中抽取 2–30 个关键词。
>
> - 不得杜撰、释义或泛化关键词。
>
> - 只能包含文本中明确出现的词或短语。
>
> - 不得包含推断或隐含的概念。
>
> - 如果文本中出现以下类型的显式关键词，请全部抽取：实体、主题、动词、时间、地点、任务、事件、人物。
>
> **ID 规则：** `sentence_id` 必须与 `TEXT` 中的 `id` 字段完全匹配。不得创建或修改 ID。
>
> **输出模式（单行 JSON）：**

```json
{
  "sentence": [
    {
      "sentence_id": "D1:1-1",
      "keyword": ["Coraline", "park"]
    }
  ]
}
```

### 对话处理提示词

> 你是一个对话处理器。只输出有效的 JSON。
>
> **任务：** 对对话中的每个句子：
>
> - 保留每个原始句子。
>
> - 用上下文中的显式实体或名词短语替换所有代词。
>
> - 不得修改动词、形容词或其他词语。
>
> - 分配一个简短、具体的标签（最多两个词）。
>
> - 使用 `conversation_time` 将时间规范化为 `YYYY-MM-DD`。
>
> - 如果某个问题由下一句回答，则将二者合并。
>
> **主题：** 总体上推导至少十个具体主题。分配主题 ID（`t1..tn`）；每个句子列出适用主题，若无则为 `[]`。
>
> **个人信息：** 把与人物有关的事实提取为 `personal_sentences`。如果某项事实出现在句子中，还需添加一个简洁的规范化版本。
>
> **ID 规则：** `id` 必须为 `origin:number`，其中 `origin` 与 `dia_id` 完全匹配。不得创建新的 ID。
>
> **输出模式（单行 JSON）：**

```json
{
  "conversation_time": "YYYY-MM-DD",
  "sentence": [
    {
      "id": "D1:1-1",
      "text": "sentence.",
      "tag": "short tag",
      "origin": "D1:1",
      "topic": ["t1", "t3"],
      "time": "YYYY-MM-DD"
    }
  ],
  "topics": {
    "t1": "Topic description",
    "t2": "Topic description"
  },
  "personal_sentences": [
    {
      "id": "p1",
      "text": "Normalized personal fact.",
      "tag": "preference",
      "origin": "D1:1",
      "person": "Name"
    }
  ]
}
```

## Sources

- `papers/agent-memory/Memory is Reconstructed, Not Retrieved - Graph Memory for LLM Agents/Memory is Reconstructed, Not Retrieved - Graph Memory for LLM Agents.pdf`
