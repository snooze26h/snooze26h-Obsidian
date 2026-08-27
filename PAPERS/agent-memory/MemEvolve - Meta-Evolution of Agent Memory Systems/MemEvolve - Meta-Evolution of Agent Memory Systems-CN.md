# MemEvolve：智能体记忆系统的元进化

OPPO AI Agent Team，LV-NUS lab

> 译注：本文为 arXiv:2512.18746v1（2025 年 12 月 21 日）精确中文翻译。方法名、模型名、数据集名与代码标识符保留英文；术语首次出现时酌情附英文。公式、图表编号、实验数值、作者-年份引文及参考文献均与原文一致。原 PDF 以图标表示 SmolAgent、Flash-Searcher、OWL 与 CK-Pro；译文将图标还原为框架名。原表 2 标题中的 `WebWalerQA`、`xBench-Ds` 为排版拼写，译文按正文标准名称 WebWalkerQA、xBench-DS 书写。

**日期：** 2025 年 12 月 23 日  
**代码：** https://github.com/bingreeky/MemEvolve

## 摘要

自进化记忆系统正以前所未有的方式重塑基于大语言模型（Large Language Model，LLM）智能体的进化范式。以往工作主要依赖人工设计的记忆架构来存储轨迹、提炼经验并合成可复用工具，使智能体能够在与环境交互的过程中即时进化。然而，这一范式从根本上受限于记忆系统本身的静态性：记忆虽能促进智能体层面的进化，其底层记忆架构却无法针对多样的任务情境进行元适应。为填补这一空白，我们提出 **MemEvolve**——一种联合进化智能体经验知识与记忆架构的元进化框架，使智能体系统不仅能够积累经验，还能逐步改进其从经验中学习的方式。为使 MemEvolve 扎根于既有研究并促进未来自进化系统的开放性，我们进一步提出 **EvolveLab**：一个统一的自进化记忆代码库，将 12 种代表性记忆系统归纳进由编码、存储、检索和管理组成的模块化设计空间，既提供标准化实现底座，也提供公平的实验平台。在 4 个高难度智能体基准上的广泛评估表明，MemEvolve 实现了：（I）显著的性能提升，使 SmolAgent、Flash-Searcher 等框架的性能最高提高 17.06%；（II）强大的跨任务与跨 LLM 泛化能力，其设计出的记忆架构能够在多种基准和骨干模型之间有效迁移。

**图 1：MemEvolve 与若干流行自进化智能体记忆系统在不同基准上的比较。** 底层框架为 Flash-Searcher（Qin et al., 2025）+ GPT-5-Mini。三个子图依次为 xBench-DS、WebWalkerQA 与 GAIA。各图按“无记忆、MemEvolve、Generative、Voyager、DILU、ExpeL、AWM、Mobile-E、Cheatsheet”的顺序给出准确率：xBench-DS 为 69.0、74.0、70.0、68.0、69.0、64.0、71.0、68.0、65.0；WebWalkerQA 为 71.18、74.71、72.35、73.53、72.94、69.41、72.35、71.76、72.94；GAIA 为 69.09、73.33、66.67、69.70、66.67、66.06、67.27、69.09、68.48。

**图 2：智能体自进化范式与人类学习之间存在自然类比。** 在一个极端，平庸学习者无法从经验中获益（无记忆智能体）。能力更强的熟练学习者能够从过往经验中抽取可复用技能，但所依赖的抽象方案是固定且预先定义的。相较之下，适应型学习者既会积累经验，也会动态调整经验的巩固与利用策略。最后这一形态正是 MemEvolve 的目标。图中示例任务为“计算 $(1+2i)6-3i$”，示例解答为“答案似乎是 $6-9i$”。从左到右依次为：平庸学习者丢弃既往解法；熟练学习者从既往轨迹中形成技能与工具（如 SkillWeaver、Alita），或技巧与洞见（如 ExpeL、G-Memory、ChemAgent）；适应型学习者进一步利用错误笔记、解题模板、既往轨迹、错误反思和可复用洞见来调整学习方式。图中把 MemoryBank、JARVIS-1、MobileGPT、Voyager 与 $\mathrm{Mem}^p$ 列作以往轨迹型记忆示例。

## 1 引言

得益于能力日益增强的基础模型（Team et al., 2025a,b）和日趋复杂的脚手架（Wang et al., 2024a; LangChain, 2023），语言智能体与智能体系统发展迅速，在深度研究（Chen et al., 2025）、科学发现（Bai et al., 2025; Wei et al., 2025b）和工业报告生成（Zhang et al., 2025g）等复杂任务上展现出前所未有的性能。推动这一成功的一项关键力量是智能体记忆系统（Zhang et al., 2024b; Hu et al., 2025c）：它持续捕获智能体与环境之间的交互，将其提炼为多种形式的知识与技能，从而使基于大语言模型（LLM）的智能体能够在任务求解和世界探索中持续进化（Wu et al., 2025c）。

自然地，记忆范式的选择对智能体即时自进化能力具有决定性影响。早期设计以原始轨迹存储和少样本提示为中心（Zhong et al., 2024; Wen et al., 2024），后来逐渐被技巧、捷径和推理模板等抽象程度更高的文本产物取代（Ouyang et al., 2025; Zhang et al., 2025b; Ye et al., 2025; Tang et al., 2025）。近期进展还探索了以结构化工具接口——例如 API（Zheng et al., 2025）和 MCP（Qiu et al., 2025b,a; Zhang et al., 2025h）——以及代码级仓库（Zhang et al., 2025e; Wang et al., 2025a）作为记忆载体。面对愈发多样的选择，求知欲强的实践者自然会问：**哪一种记忆架构最能有效驱动智能体的自我改进？**

我们认为，不存在普遍最优的记忆架构。例如，从既往轨迹中提炼可复用 API 的记忆系统可能在网页浏览等任务上表现出色，却很难为数学与科学推理提供同等帮助。反过来，以自我批评为基础的记忆虽然在推理密集型领域中很强（Cai et al., 2025），在编码和工具使用场景中的效力却会下降，这一点已由 Zhang et al.（2025d）的实证讨论所表明。我们认为，这些权衡源于当前记忆系统的静态本质。研究者通常设计一条固定的记忆流水线——即记忆摄取、抽象与检索（Zhang et al., 2025i）——并将其嵌入智能体，假定只要持续接触新经验，它就能支撑长期进化。然而，这忽视了一项关键事实：不同任务对应不同的记忆可供性。一个无法针对当前任务自我适应的记忆系统，从根本上违背了开放式智能体进化的前提。

为说明这一困境，可以考虑人类学习的类比。成绩优秀和成绩不佳的学生都难免犯错，二者的区别在于他们用何种元认知策略从错误中学习。表现不佳的学生可能诉诸死记硬背，只在表面上记下错误，却没有真正理解（Zhong et al., 2024; Orhan, 2023）。更熟练的学生则会进行高阶学习：他们不仅记录错误，还通过反思提炼可迁移的洞见（Shinn et al., 2023; Zhao et al., 2024），或推导可复用的图式（Zheng et al., 2025; Qiu et al., 2025b）。当前记忆系统实际上建模的是熟练学习者。关键缺口正在于此：最高效的人类学习者不只是熟练，而且具有适应性。他们会根据学科动态改变学习策略，例如文学分析更重记忆，数学学习则更重解题模板的抽象。我们主张，智能体记忆系统必须完成的，正是从熟练学习者向适应型学习者的转变（图 2）。更形式化地说：

> 记忆系统如何既促进智能体系统进化，又对自身架构进行元进化，在保持泛化能力的同时，为具体任务领域带来更大的性能增益？

为应对这一挑战，我们提出 MemEvolve，以促进智能体经验及其记忆架构的双重进化。从概念上看，MemEvolve 是一个双层优化过程：内循环执行一阶进化，智能体在固定记忆系统的引导下，通过填充经验库来适应连续到来的新任务；外循环驱动二阶进化，通过元学习得到更有效的记忆架构，加速未来学习。由此，智能体不仅能够进化，而且会随时间推移而进化得更加高效、更加智能。

然而，记忆系统庞大而异构的设计空间——例如知识图谱、技能库和向量数据库——给可控优化带来重大挑战。为使该优化可处理，我们引入模块化设计，将任意记忆架构分解为 4 个关键组件：♣ **编码（Encode）**，负责感知经验并格式化；♦ **存储（Store）**，负责写入信息；♥ **检索（Retrieve）**，负责上下文感知的回忆；♠ **管理（Manage）**，负责巩固与遗忘。MemEvolve 以模型驱动方式进化这些模块的程序化实现，并使用智能体在内循环中的性能反馈。该过程形成良性循环：外循环改进后的记忆架构提升智能体的学习效率；能力更强的智能体又能产生质量更高的轨迹，为外循环提供更精确的适应度信号，驱动下一轮架构进化。

为使框架立足于现有自我改进型智能体记忆的多样图景，我们在统一模块化设计空间中系统重实现了 12 种代表性架构，包括 ExpeL（Zhao et al., 2024）、Agent Workflow Memory（Wang et al., 2024b）和 Dynamic Cheatsheet（Suzgun et al., 2025）。所得框架称为 EvolveLab，既是 MemEvolve 进化过程的实证基础，也是促进未来自进化智能体研究的标准化代码库。本文贡献如下：

1. **统一代码库：** 我们提出 EvolveLab，为自我改进型智能体记忆系统提供一个涵盖编码、存储、检索和管理 4 个关键组件的模块化设计空间，并为多种主流智能体记忆系统提供统一实现和基准支持。
2. **元进化框架：** 我们提出 MemEvolve，联合进化智能体的经验知识及其底层记忆架构，使智能体系统不仅积累经验，还能逐步改进从经验中学习的机制。
3. **实验评估：** 在 4 个高难度智能体基准上的广泛实验表明，MemEvolve 能够实现：（I）显著性能提升，使 SmolAgent、Flash-Searcher 等框架最高提高 17.06%；（II）跨领域、跨框架和跨 LLM 泛化，在未见基准和骨干模型上，TaskCraft 进化出的记忆系统带来 $2.0\%-9.09\%$ 的增益。

## 2 相关工作

**LLM 智能体系统。** 过去两年，基于 LLM 的智能体系统在多个维度上迅速发展（Tran et al., 2025; Fang et al., 2025a）。从系统复杂度看，研究已从工作流由人工定义、工具配置有限的早期单智能体设置（Wu et al., 2023; Significant-Gravitas, 2023），发展到集成多种 MCP 并具备自动编排能力的复杂多智能体架构（Zhang et al., 2024a, 2025a; Wang et al., 2025b; Zhang et al., 2025c）。从任务领域看，其能力已从编码和数学推理等相对受限的领域（Hong et al., 2024; Yin et al., 2023），扩展到深度研究与科学发现等更具挑战性的领域（Du et al., 2025; Ghareeb et al., 2025）。如今，许多开源多智能体系统已在 GAIA（Mialon et al., 2023）、HLE（Phan et al., 2025）、BrowseComp（Wei et al., 2025a）和 xBench（Chen et al., 2025）等高难度基准上展现出有竞争力的性能，包括 CAMEL 的 OWL（Hu et al., 2025a）、腾讯的 CK-Pro（Fang et al., 2025c）、Skywork 的 AgentOrchestra（Zhang et al., 2025f）以及字节跳动的 AIME（Shi et al., 2025b）等。

**智能体记忆架构。** 按目标划分，智能体记忆系统大体可分为个性化记忆和自我改进型记忆（Zhang et al., 2024b; Hu et al., 2025c）。前者使智能体聊天机器人能够动态捕获用户特定的信息与偏好，后者则专注于从与环境的持续交互中提炼知识与技能以提升性能；本文采用后一重点。自我改进型记忆的主要差异在于其存储模态。早期系统将原始智能体轨迹作为少样本示例存储（Wang et al., 2023; Zhong et al., 2024; Packer et al., 2023）；后续设计则把这些经验抽象为更高层的教训与洞见（Yang et al., 2025; Sun and Zeng, 2025; Wu et al., 2025b）、程序性技巧（Wang et al., 2025c; Zheng et al., 2025; Fang et al., 2025b），以及最近出现的可复用工具和结构化仓库（Zhao et al., 2025; Qiu et al., 2025a,b; Zhang et al., 2025e）。尽管表示形式不同，这些方法有着相同愿景：使智能体以近似人类的方式学习、适应并改进。

## 3 EvolveLab：统一的自进化记忆代码库

本节首先形式化基于 LLM 的智能体系统及其记忆架构，随后介绍 EvolveLab 的模块化设计空间——它全面涵盖现有自进化智能体记忆的特征——最后介绍统一代码库 EvolveLab。

### 3.1 预备知识

我们将基于 LLM 的智能体系统形式化为 $\mathcal{M}=\langle\mathcal{I},\mathcal{S},\mathcal{A},\Psi,\Omega\rangle$，其中 $\mathcal{I}$ 为 $\{1,\ldots,N\}$ 个智能体的索引，$\mathcal{S}$ 表示共享状态空间，$\mathcal{A}=\bigcup_{i\in\mathcal{I}}\mathcal{A}_i$ 表示联合动作空间；$\Psi(s_{t+1}\mid s_t,a_t,\mu(t))$ 描述环境动态，其中 $\mu(t)\in\mathcal{I}$ 表示时间步 $t$ 的活动智能体。系统使用记忆模块 $\Omega$，该模块维护持续进化的记忆状态 $M_t$。每一步中，活动智能体观察当前状态 $s_t$，考虑任务特定查询 $Q$，并根据交互历史 $H_t$ 与 $\Omega$ 交互，以检索与上下文相关的记忆 $c_t$。随后，智能体 $\mu_t$ 的策略 $\pi_{\mu_t}$ 给出动作：

$$
a_t=\pi_{\mu(t)}(s_t,H_t,Q,c_t),\qquad c_t\sim\Omega(M_t,s_t,H_t,Q).
$$

任务执行后，系统记录轨迹 $\tau=(s_0,a_0,\ldots,s_T)$，并通过终止奖励 $R(\tau)$ 评估总体性能。记忆系统吸收新的经验单元 $\epsilon$；其粒度可从单个状态-动作转移，到聚合片段或完整轨迹不等。记忆状态更新为

$$
M_{t+1}=\Omega(M_t,\epsilon),
$$

其中 $\Omega$ 抽象了记忆整合与组织新经验或新知识的机制。

### 3.2 记忆系统的模块化设计空间

自我改进型智能体记忆异构且快速演变，给系统分析与受控实验带来挑战。为此，我们提出模块化设计空间，将任意记忆系统 $\Omega$ 分解为 4 个功能不同但彼此依赖的组件：$\Omega=(\mathcal{E},\mathcal{U},\mathcal{R},\mathcal{G})$，分别表示编码、存储、检索与管理操作。

- **编码（$\mathcal{E}$）：** 将轨迹片段 $\tau_t=(s_t,a_t,s_{t+1})$、工具输出或自我批评等原始经验转换为结构化表示 $e_t=\mathcal{E}(\epsilon_t)$。编码既可以像压缩原始轨迹那样简单（Zheng et al., 2023），也可以像抽取可泛化教训那样复杂（Zheng et al., 2025）。
- **存储（$\mathcal{U}$）：** 将编码后的经验整合进持久记忆 $M_t$，得到 $M_{t+1}=\mathcal{U}(M_t,e_t)$。存储介质可以是向量数据库（Zhao et al., 2024）、知识图谱（Zhang et al., 2025b; Rasmussen et al., 2025）或其他形式。
- **检索（$\mathcal{R}$）：** 提供任务相关的记忆内容，形式化为 $c_t=\mathcal{R}(M_t,s_t,Q)$，以辅助智能体作出策略决策 $a_t$。检索内容可以包括可复用工具（Zhang et al., 2025f）、规划经验（Tang et al., 2025），或提炼后的程序性知识（Wu et al., 2025b; Yang et al., 2025; Fang et al., 2025b）。
- **管理（$\mathcal{G}$）：** 执行巩固、抽象或选择性遗忘等离线异步操作，以维持长期记忆的质量和效率，记作 $M_t'=\mathcal{G}(M_t)$。

借助这一模块化抽象，我们可以把每种记忆系统表示为 $(\mathcal{E},\mathcal{U},\mathcal{R},\mathcal{G})$ 程序化实现的一种特定组合，形成便于 MemEvolve 元进化的“基因型”。

**表 1：EvolveLab 所实现的自我改进型智能体记忆系统分类。** “多智能体”列中，单表示支持单智能体设置，双表示兼容多智能体系统。“粒度”表示提供记忆的粒度（逐步或逐轨迹），“在线性”表示记忆是即时更新还是作为离线经验仓库维护。

| 方法 | 日期 | 多智能体 | 粒度 | 在线性 | 编码 | 存储 | 检索 | 管理 |
|---|---:|:---:|:---:|:---:|---|---|---|---|
| I. Voyager | 2023.5 | 单 | 轨迹 | 在线 | 轨迹与技巧 | 向量数据库 | 语义搜索 | 不适用 |
| II. ExpeL | 2023.8 | 单 | 轨迹 | 在线 | 轨迹与洞见 | 向量数据库 | 对比比较 | 不适用 |
| III. Generative | 2023.10 | 多 | 轨迹 | 在线 | 轨迹与洞见 | 向量数据库 | 语义搜索 | 不适用 |
| IV. DILU | 2024.2 | 单 | 轨迹 | 在线 | 轨迹 | 向量数据库 | 语义搜索 | 不适用 |
| V. AWM | 2024.9 | 单 | 轨迹 | 在线/离线 | 工作流 | 向量数据库 | 语义搜索 | 不适用 |
| VI. Mobile-E | 2025.1 | 单 | 步 | 离线 | 技巧与捷径 | 向量数据库 | 语义搜索 | 不适用 |
| VII. Cheatsheet | 2025.4 | 单 | 轨迹 | 在线 | 技巧与捷径 | JSON | 语义搜索 | 不适用 |
| VIII. SkillWeaver | 2025.4 | 单 | 轨迹 | 离线 | API | 工具库 | 函数匹配 | 技能剪枝 |
| IX. G-Memory | 2025.6 | 多 | 轨迹 | 在线 | 技巧与工作流 | 图 | 图搜索/语义搜索 | 情景巩固 |
| X. Agent-KB | 2025.7 | 多 | 步 | 离线 | 技巧与工作流 | 混合数据库 | 混合搜索 | 去重 |
| XI. Memp | 2025.8 | 单 | 步 | 在线 | 技巧与工作流 | JSON | 语义搜索 | 失败驱动调整 |
| XII. EvolveR | 2025.10 | 单 | 步 | 在线 | 技巧与工作流 | JSON | 对比比较 | 更新与剪枝 |

### 3.3 EvolveLab 代码库

基于上述设计空间，我们提出 EvolveLab：一个统一、可扩展的代码库，用于系统实现和评估自进化记忆，并为社区提供标准化资源。

**实现。** EvolveLab 的基石是模块化、层级化设计。代码库中重实现的每一种记忆架构（见表 1）都继承自唯一的抽象基类 `BaseMemoryProvider`；该基类强制实施由 ♣ 编码、♦ 存储、♥ 检索和 ♠ 管理组成的统一四组件接口。由此，多样的记忆机制可以在一致的程序结构下得到管理、修改与进化。实现细节见附录 A。

**评估。** 除统一实现外，EvolveLab 还提供标准化测试平台，用于在多样的智能体任务上严格评估记忆架构。该框架开箱即用地支持 GAIA（Mialon et al., 2023）、xBench（Chen et al., 2025）和 DeepResearchBench（Du et al., 2025）等多个高难度基准。EvolveLab 支持两种评估范式：■ **在线模式**，智能体系统处理连续任务流时即时更新经验记忆库；■ **离线模式**，记忆系统先从一组静态轨迹中积累经验，再在另一组未见任务上评估。为保证评估稳健且通用，我们支持精确字符串匹配和灵活的 LLM-as-a-Judge 等多种评估协议。

## 4 MemEvolve：元进化记忆框架

### 4.1 双重进化过程

传统自我改进型记忆系统在固定记忆架构下运行：记忆接口 $\Omega$ 预先定义且保持静态。在这一架构中，智能体通过与环境交互和经历任务，迭代填充并更新记忆状态 $M_t$。对于由查询 $Q$ 引发的轨迹 $\tau$，记忆按下式进化：

$$
M_{t+1}=\Omega(M_t,\epsilon_\tau),\qquad \epsilon_\tau\in\mathcal{E}(\tau),
$$

其中，$\mathcal{E}(\cdot)$ 表示将轨迹映射为经验单元集合的经验抽取算子，$\epsilon_\tau$ 是从该集合采样的一个元素。尽管该过程能够积累知识，却从根本上排除了架构适应，因为记忆接口 $\Omega$ 本身始终不变。

为突破这一限制，我们提出双重进化过程，联合进化：（i）智能体的记忆库；（ii）底层记忆架构（图 3）。我们不再使用单一静态 $\Omega$，而是在每个进化迭代 $k$ 中维护一组有限的候选记忆系统 $\{\Omega_j^{(k)}\}_{j\in\mathcal{J}^{(k)}}$，其中每个 $\Omega_j^{(k)}$ 都是四组件记忆接口的具体实现：

$$
\Omega_j^{(k)}\triangleq
(\mathcal{E}_j^{(k)},\mathcal{U}_j^{(k)},\mathcal{R}_j^{(k)},\mathcal{G}_j^{(k)}).
$$

初始迭代从单元素集合 $|\mathcal{J}^{(0)}|=1$ 开始，对应一个人工设计的基线记忆；之后的迭代允许多个候选相互竞争。给定使用记忆系统 $\Omega_j^{(k)}$ 执行智能体而独立生成的一批轨迹 $\mathcal{T}_j^{(k)}$，双重进化过程由两个嵌套循环组成：

- **内循环（经验进化）。** 对每个候选记忆系统 $\Omega_j^{(k)}$，其关联记忆状态 $M_{t,j}^{(k)}$ 在迭代 $k$ 开始时初始化为空，并沿轨迹 $\tau\in\mathcal{T}_j^{(k)}$ 更新：

  $$
  M_{t+1,j}^{(k)}=\Omega_j^{(k)}(M_{t,j}^{(k)},\epsilon_\tau),
  \qquad \epsilon_\tau\in\mathcal{E}_j^{(k)}(\tau).
  $$

  使用 $\Omega_j^{(k)}$ 在 $\mathcal{T}_j^{(k)}$ 上执行智能体后，每条轨迹 $\tau$ 都会产生反馈向量 $\mathbf{f}_j^{(k)}(\tau)\in\mathbb{R}^d$；$d=3$ 对应任务成功率、token 消耗和时延三项评估指标。聚合算子 $\mathcal{S}$ 将每个候选的内循环结果概括为

  $$
  \mathbf{F}_j^{(k)}=\mathcal{S}\!\left(\{\mathbf{f}_j^{(k)}(\tau)\}_{\tau\in\mathcal{T}_j^{(k)}}\right),
  \qquad j\in\mathcal{J}^{(k)}.
  $$

- **外循环（架构进化）。** 随后，根据概括集合 $\{\mathbf{F}_j^{(k)}\}_{j\in\mathcal{J}^{(k)}}$ 更新记忆架构集合。元进化算子 $\mathcal{F}$ 选择高性能候选并提出新变体，产生下一轮候选集合：

  $$
  \{\Omega_{j'}^{(k+1)}\}_{j'\in\mathcal{J}^{(k+1)}}
  =\mathcal{F}\!\left(\{\Omega_j^{(k)}\}_{j\in\mathcal{J}^{(k)}},
  \{\mathbf{F}_j^{(k)}\}_{j\in\mathcal{J}^{(k)}}\right).
  $$

  具体而言，$\mathcal{F}$ 按 $\mathbf{F}_j^{(k)}$ 对候选排序，保留 top-$K$ 个记忆系统，再通过修改或重组入选候选的 4 个组件 $(\mathcal{E},\mathcal{U},\mathcal{R},\mathcal{G})$ 生成新架构，其中 $K$ 是固定的幸存者预算。$\mathcal{F}(\cdot)$ 的实现详见第 4.2 节。

**统一视角。** 在更高层次上，每个迭代 $k$ 在两步之间交替：（i）在一组固定架构下，从空初始化开始进化记忆经验库；（ii）根据所得性能进化记忆架构本身：

$$
\left(\{\varnothing\}_{j\in\mathcal{J}^{(k)}},\{\Omega_j^{(k)}\}_{j\in\mathcal{J}^{(k)}}\right)
\xrightarrow{\text{内循环}}
\left(\{M_{t+1,j}^{(k)}\}_{j\in\mathcal{J}^{(k)}},\{\Omega_j^{(k)}\}_{j\in\mathcal{J}^{(k)}}\right)
\xrightarrow{\text{外循环}}
\left(\{M_{t+1,j}^{(k)}\}_{j\in\mathcal{J}^{(k)}},\{\Omega_{j'}^{(k+1)}\}_{j'\in\mathcal{J}^{(k+1)}}\right).
$$

通过反复执行双重进化，智能体不再只是于固定记忆系统中积累经验；记忆库及其支配架构会共同进化，随时间推移产生适应性和资源感知能力日益增强的记忆驱动行为。

**图 3：MemEvolve 概览。** 左侧搜索空间把记忆系统拆分为编码、存储、检索与管理：轨迹或轨迹片段编码为记忆单元，单元可进入向量数据库、知识图谱等存储，再按状态和任务查询进行检索，并通过巩固、更新、遗忘等操作管理。中间展示候选记忆系统跨迭代进化：基础系统在第一轮产生候选；候选可把语义搜索演化为混合搜索，或从原始轨迹存储演化为摘要、工具、捷径和多粒度抽象。右侧“诊断-设计”（D&D）过程综合候选性能、API 成本、执行时延和任务日志，以 Pareto 排序选出候选；元进化器据诊断报告识别缺少工具/技能、记忆内容过长、筛选策略含糊等问题，再提出编码层级、存储介质和检索护栏等方面的改造。

### 4.2 诊断-设计进化

下面详细介绍元进化算子 $\mathcal{F}$，它控制每次进化迭代中的架构更新。从概念上看，$\mathcal{F}$ 分解为两个协同组件：（i）架构选择，识别一组高性能记忆系统作为进化父代；（ii）诊断-设计进化，先执行结构化诊断，再在模块化记忆设计空间内进行受约束的重新设计，由每个入选父代生成新的记忆架构。

**架构选择。** 给定候选集合 $\{\Omega_j^{(k)}\}_{j\in\mathcal{J}^{(k)}}$ 及相应概括 $\{\mathbf{F}_j^{(k)}\}$，定义每个概括向量为

$$
\mathbf{F}_j^{(k)}\triangleq
(\mathrm{Perf}_j^{(k)},-\mathrm{Cost}_j^{(k)},-\mathrm{Delay}_j^{(k)}),
$$

所有维度均以数值越高越优。首先依据 $\mathbf{F}_j^{(k)}$ 进行非支配排序，得到 Pareto 等级 $\rho_j^{(k)}$。在同一 Pareto 等级内，再按主要性能指标 $\mathrm{Perf}_j^{(k)}$ 排序。选出 top-$K$ 个候选形成父代集合：

$$
\mathcal{P}^{(k)}=\underset{j\in\mathcal{J}^{(k)}}{\operatorname{Top-K}}
(\rho_j^{(k)},\mathrm{Perf}_j^{(k)}).
$$

这一步确保架构进化由在任务有效性和资源效率之间取得良好权衡的系统引导，同时在 Pareto 等价候选之间优先考虑任务性能。

**诊断-设计进化。** 对每个父代架构 $\Omega_p^{(k)}\in\mathcal{P}^{(k)}$，$\mathcal{F}$ 通过两个阶段生成一组 $S$ 个后代 $\{\Omega_{p,s}^{(k+1)}\}_{s=1}^{S}$：

- **诊断。** 使用父代自身执行批次 $\mathcal{T}_p^{(k)}$ 中的轨迹级证据检查每个父代架构。对每条轨迹，智能体提供结果统计量（如成功指示和 token 成本）以及相关任务查询的结构化描述。回放接口允许访问相应轨迹 $\tau\in\mathcal{T}_p^{(k)}$，从而定向检查记忆行为，包括检索失败、无效抽象或低效存储。诊断阶段由此产生结构化缺陷画像 $\mathcal{D}(\Omega_p^{(k)})$，刻画 4 个记忆组件 $(\mathcal{E}_p^{(k)},\mathcal{U}_p^{(k)},\mathcal{R}_p^{(k)},\mathcal{G}_p^{(k)})$ 中的架构瓶颈。
- **设计。** 以缺陷画像 $\mathcal{D}(\Omega_p^{(k)})$ 为条件，只修改模块接口内允许变更的实现位置，由此构造重新设计的架构，以保证兼容性并把架构变更限制在指定设计空间内。设计步骤通过实例化 4 个组件的不同但有效的配置，生成 $S$ 个变体：

  $$
  \Omega_{p,s}^{(k+1)}=\operatorname{Design}
  (\Omega_p^{(k)},\mathcal{D}(\Omega_p^{(k)}),s),
  \qquad s\in\{1,\ldots,S\}.
  $$

  这些变体在编码策略、存储规则、检索约束或管理策略上有所不同，但全部符合统一的记忆系统接口，并可由智能体执行。

**更新结果。** 聚合所有父代产生的后代，得到下一组候选架构：

$$
\{\Omega_{j'}^{(k+1)}\}_{j'\in\mathcal{J}^{(k+1)}}
=\bigcup_{\Omega_p^{(k)}\in\mathcal{P}^{(k)}}
\{\Omega_{p,s}^{(k+1)}\}_{s=1}^{S}.
$$

这种诊断-设计进化将 $\mathcal{F}$ 落实为可操作机制，用于产生适应性不断增强的记忆系统，并确保架构更新既有实证依据，又在结构上受到统一设计空间的约束。

## 5 实验

### 5.1 实验设置

**基准。** 我们在 4 个高难度智能体基准上评估所提框架，包括 GAIA（Mialon et al., 2023）、WebWalkerQA（Wu et al., 2025a）、xBench-DeepSearch（xBench-DS）（Chen et al., 2025）以及 TaskCraft（Shi et al., 2025a）。更多统计信息与细节见附录 B.1。

**表 2：不同智能体框架在 WebWalkerQA、xBench-DS、TaskCraft 和 GAIA 基准上的性能。** “-”表示原文未报告。

| 框架 | 模型家族 | WebWalkerQA | xBench-DS | TaskCraft | GAIA 平均 | Level 1 | Level 2 | Level 3 |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| **闭源智能体框架** |||||||||
| Langfun | Claude 3.7 等 | - | - | - | 71.52 | 83.02 | 68.60 | 57.69 |
| TraseAgent | Claude 等 | - | - | - | 70.30 | 83.02 | 69.77 | 46.15 |
| OpenAI Deep Research | o1、o3 等 | - | - | - | 67.36 | 74.29 | 69.06 | 47.60 |
| h2oGPTe | Claude-3.5 | - | - | - | 63.64 | 67.92 | 67.44 | 42.31 |
| Desearch | GPT-4o | - | - | - | 56.97 | 71.70 | 58.14 | 23.08 |
| **开源智能体框架** |||||||||
| OWL Workforce（pass@3） | GPT-4o+o3-mini | 57.64 | 55.0 | 58.33 | 60.61 | 81.14 | 58.14 | 26.92 |
| OWL RP（pass@3） | GPT-4o+o3-mini | - | - | - | 58.18 | 81.14 | 54.65 | 23.08 |
| TapeAgents | Claude 3.7 等 | - | - | - | 55.76 | 71.70 | 53.49 | 30.77 |
| AutoAgent | Claude 3.5 等 | - | - | - | 55.15 | 71.70 | 53.40 | 26.92 |
| SmolAgent | GPT-4.1 | - | - | - | 55.15 | 67.92 | 53.49 | 34.62 |
| SmolAgent | GPT-5-mini | 58.82 | 51.0 | 64.00 | 55.75 | 69.81 | 54.65 | 30.77 |
| Magnetic-1 | OpenAI o1 等 | - | - | - | 46.06 | 56.60 | 46.51 | 23.08 |
| Cognitive Kernel-Pro（pass@1） | Claude-3.7 等 | 60.64 | 56.0 | 66.00 | 60.00 | 79.25 | 56.98 | 30.77 |
| Cognitive Kernel-Pro（pass@3） | Claude-3.7 等 | - | - | - | 75.15 | 84.91 | 73.26 | 61.54 |
| OAgents | Claude-3.7 等 | 58.23 | 47.0 | - | 66.67 | 77.36 | 66.28 | 46.15 |
| JoyAgents | Claude-4、o4-mini | - | - | - | 75.2 | 86.8 | 77.9 | 42.3 |
| Agent KB（pass@1） | GPT-4.1 | 60.59 | 48.0 | 61.67 | 61.21 | 79.25 | 58.14 | 34.62 |
| Agent KB（pass@2） | GPT-4.1 | 68.82 | 58.0 | 72.67 | 67.27 | 83.02 | 67.44 | 34.62 |
| Agent KB（pass@3） | GPT-4.1 | 73.53 | 68.0 | 75.33 | 73.94 | 84.91 | 73.26 | 53.85 |
| Flash-Searcher（pass@1） | GPT-5-mini | 71.18 | 69.0 | 69.67 | 69.09 | 79.25 | 69.77 | 46.15 |
| Flash-Searcher（pass@1） | Kimi K2 | 52.35 | 66.0 | 58.00 | 52.12 | 58.49 | 52.33 | 34.62 |
| Flash-Searcher（pass@1） | DeepSeek V3.2 | 69.41 | 68.0 | 69.33 | 60.61 | 79.25 | 53.49 | 46.15 |
| **MemEvolve + SmolAgent（pass@1）** | GPT-5-mini | 61.18 | 57.0 | 67.67 | 64.24 | 83.02 | 58.14 | 46.15 |
| **MemEvolve + SmolAgent（pass@2）** | GPT-5-mini | 67.06 | 63.0 | 75.00 | 67.88 | 84.91 | 63.95 | 46.15 |
| **MemEvolve + SmolAgent（pass@3）** | GPT-5-mini | 71.18 | 68.0 | 77.00 | 72.12 | 88.68 | 68.60 | 50.00 |
| **MemEvolve + Flash-Searcher（pass@1）** | GPT-5-mini | 74.71 | 74.0 | 72.00 | 73.33 | 83.02 | 73.26 | 53.85 |
| **MemEvolve + Flash-Searcher（pass@2）** | GPT-5-mini | 79.41 | 77.0 | 75.00 | 77.58 | 92.45 | 74.42 | 57.69 |
| **MemEvolve + Flash-Searcher（pass@3）** | GPT-5-mini | 81.18 | 78.0 | 79.33 | 80.61 | 94.34 | 79.07 | 57.69 |
| **MemEvolve + Flash-Searcher（pass@1）** | Kimi K2 | 69.41 | 68.0 | 68.00 | 61.21 | 67.92 | 63.95 | 38.46 |
| **MemEvolve + Flash-Searcher（pass@1）** | DeepSeek V3.2 | 72.35 | 70.0 | 72.67 | 67.88 | 83.02 | 63.95 | 50.00 |

**方法配置。** 双重进化过程共运行 $K_{\max}=3$ 轮。在外循环中，幸存者预算设为 $K=1$；每轮只保留排名最高的架构，并扩展为 $S=3$ 个后代。在内循环中，每个候选架构 $\Omega_j^{(k)}$ 都在一批 $\mathcal{T}_j^{(k)}$、共 60 条任务轨迹上评估，其中包括 40 个新采样任务和 20 个从上一轮复用的任务，以稳定迭代间比较。

**智能体框架。** 我们将 MemEvolve 集成进两个代表性智能体框架：SmolAgent（Roucher et al., 2025），一种轻量级双智能体架构；以及 Flash-Searcher（Qin et al., 2025），一种高性能单智能体深度研究系统。为评估 MemEvolve 的泛化与即插即用能力，我们还在两个留出多智能体系统上进行评估：腾讯的 Cognitive Kernel-Pro（CK-Pro）（Fang et al., 2025c），由主智能体、文件智能体和网页智能体组成的三智能体框架；以及 OWL（Hu et al., 2025b），一个包含规划、协调、网页、文档和编码智能体的层级系统。这种架构与系统复杂度上的多样性，使我们能够全面检验 MemEvolve 对异构智能体脚手架的适应性。

**模型配置。** 我们使用 GPT-5-mini（OpenAI, 2025）实例化 MemEvolve，既作为底层智能体框架的 LLM 骨干，也用于支持元进化算子 $\mathcal{F}(\cdot)$。为进一步评估 MemEvolve 的跨 LLM 泛化能力，我们还考虑 DeepSeek V3.2（DeepSeek-AI et al., 2025）和 Kimi K2（Team et al., 2025a）等替代骨干。为清楚起见，后续实验明确报告每个智能体框架使用的具体 LLM 骨干。

**表 3：Flash-Searcher 在不同记忆设置下跨数据集的性能、成本、时延和步数。** 成本表示每条任务查询产生的平均 API 成本；时延表示每个任务的平均执行时延（秒）；步数表示完成每项任务所需的智能体交互步数。

| 记忆设置 | GAIA 性能 | 成本 | 时延 | 步数 | xBench 性能 | 成本 | 时延 | 步数 | WebWalkerQA 性能 | 成本 | 时延 | 步数 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 无记忆 | 69.09 | 0.086 | 505.46 | 10.44 | 69.00 | 0.141 | 523.05 | 14.69 | 71.18 | 0.048 | 251.57 | 6.91 |
| Generative | 66.67 | 0.061 | 436.26 | 8.87 | 70.00 | 0.131 | 818.37 | 13.45 | 72.35 | 0.045 | 268.56 | 6.64 |
| Voyager | 69.70 | 0.060 | 499.89 | 9.25 | 68.00 | 0.117 | 553.46 | 12.71 | 73.53 | 0.049 | 333.69 | 6.99 |
| DILU | 66.67 | 0.059 | 444.62 | 8.91 | 69.00 | 0.134 | 500.72 | 13.83 | 72.94 | 0.046 | 272.16 | 6.96 |
| ExpeL | 66.06 | 0.059 | 500.11 | 8.68 | 64.00 | 0.123 | 710.32 | 13.05 | 69.41 | 0.076 | 385.28 | 10.96 |
| AWM | 67.27 | 0.062 | 584.88 | 10.23 | 71.00 | 0.138 | 761.33 | 14.12 | 72.35 | 0.068 | 397.20 | 11.40 |
| Mobile-E | 69.09 | 0.065 | 321.80 | 9.35 | 68.00 | 0.120 | 537.18 | 13.16 | 71.76 | 0.059 | 296.01 | 6.52 |
| Cheatsheet | 68.48 | 0.069 | 559.81 | 9.72 | 65.00 | 0.174 | 818.07 | 15.99 | 72.94 | 0.057 | 367.13 | 7.59 |
| **MemEvolve** | **73.33** | **0.085** | **693.33** | **10.14** | **74.00** | **0.136** | **773.06** | **14.20** | **74.71** | **0.040** | **332.49** | **6.64** |

### 5.2 主要结果

表 2 报告了 MemEvolve 与 SmolAgent、Flash-Searcher 集成后的 pass@1-3 性能，以及与未见 LLM（Kimi K2、DeepSeek V3.2）配合时的泛化结果。特别地，在相对简单的 TaskCraft 基准上，我们分别使用 MemEvolve + SmolAgent 和 MemEvolve + Flash-Searcher 进化出两个不同的记忆系统。随后固定这些进化后的记忆系统，并在 WebWalkerQA 和 xBench-DS 上进行评估，即不再针对数据集执行专门的元进化。

**记忆系统对智能体系统至关重要。** 如表 2 所示，为智能体系统配备有效的记忆架构对性能十分关键。在 xBench 上，SmolAgent + GPT-5-Mini 的初始 pass@1 为 51%；集成 MemEvolve 后，pass@1 提高 6 个百分点，pass@3 则升至 68.0%。类似地，Flash-Searcher + GPT-5-Mini 在配备 MemEvolve 后，其 xBench 性能从 69% 提升至 74%。这些结果清楚表明，设计良好的记忆系统会对智能体性能产生重大影响。与此同时，记忆并非万能药，其效果仍受底层智能体框架能力的限制。在 GAIA 上，MemEvolve + SmolAgent 的 pass@3 达到 72.12%，与 AgentKB 相当，却无需构建庞大而昂贵的离线知识库。相比之下，MemEvolve + Flash-Searcher 的提升更为显著，pass@3 达到 80.61%，在同一指标下超过 OWL-Workforce 和 CK-Pro 等强大多智能体系统。

**MemEvolve 展现跨任务、跨模型和跨框架泛化。** 前文已经说明，WebWalkerQA 与 xBench 使用的记忆系统直接继承自在 TaskCraft 上进化的系统，并未进行任何任务特定的元进化。即便如此，这些迁移记忆仍在更具挑战性的基准上带来一致提升（WebWalkerQA + SmolAgent：$58.82\rightarrow61.18\%$；xBench + Flash-Searcher：$69.0\rightarrow74.0\%$），表明 MemEvolve 捕获的是与任务无关的记忆设计原则，而非过拟合单个数据集。MemEvolve 也展现出强大的跨 LLM 泛化能力。尽管元进化使用 GPT-5-Mini 完成，在 TaskCraft + Flash-Searcher 上进化出的记忆系统仍能不经人工适配地有效迁移到 Kimi K2 和 DeepSeek V3.2。尤其是，Kimi K2 + Flash-Searcher 在 WebWalkerQA 上提高 17.06%，在 TaskCraft 上提高 10.0%。最后，MemEvolve 还表现出很强的跨框架泛化。图 4 表明，把在 TaskCraft + Flash-Searcher 上进化出的记忆系统直接迁移到 OWL 和 CK-Pro 等异构智能体框架，即使架构差异显著，也能一致提升性能。这说明 MemEvolve 学到的是与框架无关、可方便插入多样智能体系统的记忆抽象。

**图 4：跨框架泛化分析。** 将在 TaskCraft + Flash-Searcher 上进化的记忆系统迁移到 OWL-Workforce 与 CK-Pro。红色百分比表示各框架集成 MemEvolve 后，相对于无记忆版本的得分增幅。OWL-Workforce 在 Web、xBench、TaskCraft、GAIA 上分别提高 7.1%、10.9%、11.9%、2.7%；CK-Pro 分别提高 4.2%、16.1%、4.8%、8.4%。

**图 5：累积准确率随问题索引的变化。** 索引 $i$ 处的累积准确率定义为前 $i$ 个问题的平均准确率。由于早期样本量有限，曲线在较小索引处波动更大；随着问题不断累积，曲线逐渐稳定。图中比较无记忆、Generative、Voyager、DILU、ExpeL、AWM、Mobile-E、Cheatsheet 与 MemEvolve 在 GAIA、xBench、WebWalkerQA 上的变化。

**图 6：从固定 AgentKB 架构逐步进化为智能体性更强、效率更高的记忆架构。** 每一阶段都反映了记忆编码、存储、检索与维护方面的结构和功能变更，最终产生 Riva、Cerebra 等高性能系统。进化路径如下：

- $\Omega_1^{(0)}$，AKB Mem：编码冻结；存储冻结；检索为“推理-检索-精炼”；无管理。
- 第一轮候选 $\Omega_1^{(1)}$，Adaptive Decision：编码为 9 种技能；JSON 存储；混合检索 + LLM 精炼；无管理。
- 第一轮获胜候选 $\Omega_3^{(1)}$，Meta Mem：编码任务/策略/操作/检查；JSON 存储；混合检索 + 元护栏；无管理。
- 第二轮候选 $\Omega_1^{(2)}$，Domain Mem：编码失败模式/指导；条目列表存储；上下文感知评分器 + LLM 引导器；LLM 排序器管理。
- 第二轮获胜候选 $\Omega_3^{(2)}$，Riva：按领域与语义编码；数据库存储；引导 + 探测 + 门控检索；无管理。
- 第三轮获胜候选 $\Omega_1^{(3)}$，Cerebra Mem：图编码；将成功轨迹存为语义内容；文本/工具双路检索；节点/边剪枝与巩固。
- 第三轮另一候选 $\Omega_3^{(3)}$，Omni-Mem：编码轨迹、标签与摘要；JSON 存储；精炼 + 推理 + 综合检索；无管理。

### 5.3 自进化记忆比较

我们进一步将 MemEvolve 自动进化出的记忆系统与主流人工设计的自我改进型记忆系统进行比较。在表 3 中，我们把 EvolveLab 实现的 7 种代表性自我改进型记忆系统集成进 Flash-Searcher，并全面报告性能、每任务成本、执行时延与执行步数。MemEvolve 的结果来自在 TaskCraft + Flash-Searcher + GPT-5-Mini 上进化出的系统。

**现有记忆系统无法带来一致增益。** 尽管我们忠实按照原始设计进行重实现，许多现有记忆系统仍无法稳定改进性能。例如，DILU 能够提升 xBench 和 WebWalkerQA 的性能，却使 GAIA 下降 2.42%。Dynamic Cheatsheet 通过技能浓缩使 WebWalkerQA 提高 1.76%，但在 GAIA 和 xBench 上表现不佳。还存在更极端的情况：ExpeL 在 3 个基准上都不如基线。仔细分析后，这并不意外，因为 ExpeL 最初面向相对简单的具身或问答设置（如 ALFWorld、HotpotQA）而设计，其提示与机制并不适合长时程、长上下文的深度研究。这些结果突显出任务感知记忆设计的必要性。

**图 7：进化记忆在 GAIA 与 xBench 真实任务中的实例化方式。** 记忆系统会自适应提供与阶段相匹配的指导：从高层规划和任务分解，到细粒度工具使用建议及显著上下文回忆，从而引导智能体高效、成功地完成任务。

左侧 GAIA 任务询问：“找出赢得英国电影和电视艺术学院游戏奖的 2019 年游戏对应的 Wikipedia 页面。截至 2022 年的最新条目，在该 Wikipedia 页面所列游戏发行月份之前，该页面共发生过多少次修订？”执行分为规划、定位 Wikipedia 页面、定位文章等步骤。Lightweight 提供的记忆包括：

- **消歧：** 先通过定向 `site:wikipedia.org` 查询定位规范 Wikipedia 条目；查询中使用游戏名以及 `BAFTA` 或 `British Academy Games Awards` 等关键词，以免产生歧义。
- **工具使用建议：** 根据相似任务，使用 MediaWiki API/历史端点列出修订，并以发行月份日期设置截止值（`rvend` 或等价方式），从而统计该月份之前的全部修订。
- **规划建议：** 将任务分解为“定位 → 抽取修订 → 统计截止值之前的数量 → 用存档快照（Wayback/Wikidata）交叉验证”，并保存精确 URL/oldid，使结果可审计。
- **工具使用建议：** 使用带 `rvend` 截止值的 MediaWiki API revisions 接口，列出并统计发行月份截止之前的修订。
- **上下文提醒：** 已定位规范 Wikipedia 条目 `Outer Wilds`（https://en.wikipedia.org/wiki/Outer_Wilds），并确认它与 2019 年 BAFTA 获奖游戏相符。正文列出的年月包括 2019 年 5 月（Windows、Xbox One）、2019 年 10 月（PS4）、2022 年 9 月（PS5/Xbox）和 2023 年 12 月（Switch）。

右侧 xBench 任务为：“有一个景点，是一个老太太花了很多年的心血，用一片片瓷片建成的。这个景点的门票上，印着一行字，请问这行字是什么？”智能体先规划，并识别景点为“瓷宫”；子任务目标是确定门票上印刷的具体文字。在定位视角、寻找文字阶段，Lightweight 给出：

- **可能来源：** 检索到的一条记忆写道：“以往任务发现，当文章省略门票文字时，图片说明和旅游列表往往会包含该信息。”这说明如果文字描述没有提及门票题字，相关信息常见于图片说明或在线旅游预订列表。

在该记忆引导下，智能体系统查询 Trip.com、去哪儿等旅游平台，重点检查景点列表和售票页中的结构化字段。

**MemEvolve 带来稳健且一致的提升。** 与既有方法不同，MemEvolve 带来稳定、稳健的性能增益。尽管底层记忆系统在 TaskCraft 上进化，它在全部 3 个评估基准上都能稳定提高 $3.54\%-5.0\%$。重要的是，这些增益并非以大幅增加每任务成本为代价。表 3 显示，MemEvolve 在所有基准上的 API 成本都与无记忆基线相当（例如 GAIA：\$0.085 对 \$0.086；xBench：\$0.136 对 \$0.141），执行时延也与其他自我改进基线处于相近量级（例如 GAIA：693.33 秒，对比 AWM 的 584.88 秒、Cheatsheet 的 559.81 秒；xBench：773.06 秒，对比 AWM 的 761.33 秒、Cheatsheet 的 818.07 秒）。图 5 进一步展示不同自进化记忆系统随任务执行推进的累积成功率。早期因样本量有限，性能方差较大；此后 MemEvolve 逐渐稳定，并收敛到始终更优的性能区间。这说明 MemEvolve 发现的是有原则且有效的记忆设计，而非依赖脆弱的任务特定启发式规则。

乍看之下，这种泛化可能与本文最初的动机冲突：既然记忆系统无法跨所有领域泛化，就需要任务特定进化。我们认为并非如此。TaskCraft 上进化出的记忆系统不太可能有效迁移到本质不同的任务家族（例如具身动作），因为环境、动作空间和工具集合存在实质差异。但在共享的任务形态内，MemEvolve 能够发现具有广泛适用性的记忆架构，并在需要时保留进一步进行任务特定适应的能力。

### 5.4 元进化动态

在确认 MemEvolve 带来的显著性能增益后，我们进一步考察元进化在实践中如何执行，以及进化过程中哪些组件得到修改或增强。如图 6 所示，MemEvolve 从 AgentKB 的预定义结构出发，迭代进化为效率越来越高的记忆架构。图 9 和图 10 展示该轨迹上发现的两个高性能记忆系统，分别称为 Riva 和 Cerebra。图 8 则展示从最简单的少样本示例记忆基线进化而来的系统 Lightweight。

**智能体自发进化出高效记忆架构。** 如图 6 所示，初始 AgentKB 记忆系统对编码和存储都采用冻结设计，无法吸收新经验。以此为基线，MemEvolve 探索一系列进化方向。一些候选相对激进，例如 $\Omega_1^{(1)}$ Adaptive Decision System 把单条智能体轨迹分解为 9 种技能粒度；另一些候选则较为保守，例如 $\Omega_3^{(1)}$ Meta Memory System 在 4 个层次存储轨迹，并在检索期间引入基于 LLM 的元护栏，过滤无关信息。后者成为第一轮进化的胜者。这一阶段的决定性特征是“智能体性”：记忆编码与解码都越来越依赖智能体驱动的决策，而非预定义流水线。第三轮进化又引入两项进展。记忆系统从 $\Omega_3^{(2)}$ Riva 进化到 $\Omega_1^{(3)}$ Cerebra 后，不仅学会从既往经验中提炼文本洞见，也会提炼可复用工具，并加入记忆数据库的周期性维护。这些增强共同为底层智能体框架提供更快的进化动量。

**进化后的记忆系统在实践中有效。** 图 7 进一步给出 Lightweight 系统在真实执行中生成的具体记忆示例。结果表明，Lightweight 能以不同粒度提供记忆内容，并根据任务阶段自适应调整。早期规划时，记忆提供任务分解策略等高层指导；执行推进后，它提供更细粒度的工具使用建议，以及一种突出前几轮显著信息的工作记忆。值得注意的是，Lightweight 还表现出预测行为：它预判目标信息可能出现在在线旅游网站的图片内容中，并成功引导智能体在 trip.com 定位证据。这些示例共同说明，MemEvolve 进化出的记忆系统在实践中确实有效。

## 6 结论

本文为快速发展的自进化智能体记忆领域提供了统一实现与设计空间，并给出名为 EvolveLab 的标准化代码库；在此基础上，我们进一步构建元进化记忆框架 MemEvolve。与人工打造单一自我改进型记忆架构、再期待它泛化到所有领域的传统范式不同，MemEvolve 采用由实证交互反馈驱动的、自适应的架构级进化。跨多样智能体基准和骨干模型的广泛实验，证明了该方法的有效性、稳健性与泛化能力。此外，对自动进化记忆系统的分析还揭示出若干具有启发性的设计原则，包括提高智能体参与程度、采用层级组织以及多层次抽象。我们希望 MemEvolve 能够推动构建持续改进的智能体智能，走向更加自动化、更有原则、也更具元进化性的路径。

## 7 贡献者

**核心贡献者**

- Guibin Zhang
- Haotian Ren

**贡献者**

- Chong Zhan
- Zhenhong Zhou
- Junhao Wang
- He Zhu

**通信作者**

- Wangchunshu Zhou
- Shuicheng Yan

如对代码、论文细节或本工作的其他方面有任何问题，欢迎通过 `guibinz@outlook.com` 联系作者，或提交 GitHub issue。

## 参考文献

> 为确保作者、题名、版本和检索信息准确，以下书目信息保留原文，不翻译。

- Bai, L., Cai, Z., Cao, Y., Cao, M., Cao, W., Chen, C., Chen, H., Chen, K., Chen, P., Chen, Y., Chen, Y., Cheng, Y., Chu, P., Chu, T., Cui, E., Cui, G., Cui, L., Cui, Z., Deng, N., Ding, N., Dong, N., Dong, P., Dou, S., Du, S., Duan, H., Fan, C., Gao, B., Gao, C., Gao, J., Gao, S., Gao, Y., Gao, Z., Ge, J., Ge, Q., Gu, L., Gu, Y., Guo, A., Guo, Q., Guo, X., He, C., He, J., Hong, Y., Hou, S., Hu, C., Hu, H., Hu, J., Hu, M., Hua, Z., Huang, H., Huang, J., Huang, X., Huang, Z., Jiang, Z., Kong, L., Li, L., Li, P., Li, P., Li, S., Li, T., Li, W., Li, Y., Lin, D., Lin, J., Lin, T., Lin, Z., Liu, H., Liu, J., Liu, J., Liu, J., Liu, K., Liu, K., Liu, K., Liu, S., Liu, S., Liu, W., Liu, X., Liu, Y., Liu, Z., Lu, Y., Lv, H., Lv, H., Lv, H., Lv, Q., Lv, Y., Lyu, C., Ma, C., Ma, J., Ma, R., Ma, R., Ma, R., Ma, X., Ma, Y., Ma, Z., Mi, S., Ning, J., Ning, W., Pang, X., Peng, J., Peng, R., Qiao, Y., Qiu, J., Qu, X., Qu, Y., Ren, Y., Shang, F., Shao, W., Shen, J., Shen, S., Song, C., Song, D., Song, D., Su, C., Su, W., Sun, W., Sun, Y., Tan, Q., Tang, C., Tang, H., Tang, K., Tang, S., Tong, J., Wang, A., Wang, B., Wang, D., Wang, L., Wang, R., Wang, W., Wang, W., Wang, J., Wang, Y., Wang, Z., Wu, L.-I., Wu, W., Wu, Y., Wu, Z., Xiao, L., Xing, S., Xu, C., Xu, H., Xu, J., Xu, R., Xu, W., Yang, G., Yang, Y., Ye, H., Ye, J., Ye, S., Yu, J., Yu, J., Yu, J., Yuan, F., Zang, Y., Zhang, B., Zhang, C., Zhang, C., Zhang, H., Zhang, J., Zhang, Q., Zhang, Q., Zhang, S., Zhang, T., Zhang, W., Zhang, W., Zhang, Y., Zhang, Z., Zhao, H., Zhao, Q., Zhao, X., Zhao, X., Zhou, B., Zhou, D., Zhou, P., Zhou, Y., Zhou, Y., Zhu, D., Zhu, L., and Zou, Y. (2025). Intern-s1: A scientific multimodal foundation model.

- Cai, Y., Cai, S., Shi, Y., Xu, Z., Chen, L., Qin, Y., Tan, X., Li, G., Li, Z., Lin, H., Mao, Y., Li, K., and Sun, X. (2025). Training-free group relative policy optimization.

- Chen, K., Ren, Y., Liu, Y., Hu, X., Tian, H., Xie, T., Liu, F., Zhang, H., Liu, H., Gong, Y., Sun, C., Hou, H., Yang, H., Pan, J., Lou, J., Mao, J., Liu, J., Li, J., Liu, K., Liu, K., Wang, R., Li, R., Niu, T., Zhang, W., Yan, W., Wang, X., Zhang, Y., Hung, Y.-H., Jiang, Y., Liu, Z., Yin, Z., Ma, Z., and Mo, Z. (2025). xbench: Tracking agents productivity scaling with profession-aligned real-world evaluations.

- DeepSeek-AI, Liu, A., Mei, A., Lin, B., Xue, B., Wang, B., Xu, B., Wu, B., Zhang, B., Lin, C., Dong, C., Lu, C., Zhao, C., Deng, C., Xu, C., Ruan, C., Dai, D., Guo, D., Yang, D., Chen, D., Li, E., Zhou, F., Lin, F., Dai, F., Hao, G., Chen, G., Li, G., Zhang, H., Xu, H., Li, H., Liang, H., Wei, H., Zhang, H., Luo, H., Ji, H., Ding, H., Tang, H., Cao, H., Gao, H., Qu, H., Zeng, H., Huang, J., Li, J., Xu, J., Hu, J., Chen, J., Xiang, J., Yuan, J., Cheng, J., Zhu, J., Ran, J., Jiang, J., Qiu, J., Li, J., Song, J., Dong, K., Gao, K., Guan, K., Huang, K., Zhou, K., Huang, K., Yu, K., Wang, L., Zhang, L., Wang, L., Zhao, L., Yin, L., Guo, L., Luo, L., Ma, L., Wang, L., Zhang, L., Di, M. S., Xu, M. Y., Zhang, M., Zhang, M., Tang, M., Zhou, M., Huang, P., Cong, P., Wang, P., Wang, Q., Zhu, Q., Li, Q., Chen, Q., Du, Q., Xu, R., Ge, R., Zhang, R., Pan, R., Wang, R., Yin, R., Xu, R., Shen, R., Zhang, R., Liu, S. H., Lu, S., Zhou, S., Chen, S., Cai, S., Chen, S., Hu, S., Liu, S., Hu, S., Ma, S., Wang, S., Yu, S., Zhou, S., Pan, S., Zhou, S., Ni, T., Yun, T., Pei, T., Ye, T., Yue, T., Zeng, W., Liu, W., Liang, W., Pang, W., Luo, W., Gao, W., Zhang, W., Gao, X., Wang, X., Bi, X., Liu, X., Wang, X., Chen, X., Zhang, X., Nie, X., Cheng, X., Liu, X., Xie, X., Liu, X., Yu, X., Li, X., Yang, X., Li, X., Chen, X., Su, X., Pan, X., Lin, X., Fu, X., Wang, Y. Q., Zhang, Y., Xu, Y., Ma, Y., Li, Y., Li, Y., Zhao, Y., Sun, Y., Wang, Y., Qian, Y., Yu, Y., Zhang, Y., Ding, Y., Shi, Y., Xiong, Y., He, Y., Zhou, Y., Zhong, Y., Piao, Y., Wang, Y., Chen, Y., Tan, Y., Wei, Y., Ma, Y., Liu, Y., Yang, Y., Guo, Y., Wu, Y., Wu, Y., Cheng, Y., Ou, Y., Xu, Y., Wang, Y., Gong, Y., Wu, Y., Zou, Y., Li, Y., Xiong, Y., Luo, Y., You, Y., Liu, Y., Zhou, Y., Wu, Z. F., Ren, Z. Z., Zhao, Z., Ren, Z., Sha, Z., Fu, Z., Xu, Z., Xie, Z., Zhang, Z., Hao, Z., Gou, Z., Ma, Z., Yan, Z., Shao, Z., Huang, Z., Wu, Z., Li, Z., Zhang, Z., Xu, Z., Wang, Z., Gu, Z., Zhu, Z., Li, Z., Zhang, Z., Xie, Z., Gao, Z., Pan, Z., Yao, Z., Feng, B., Li, H., Cai, J. L., Ni, J., Xu, L., Li, M., Tian, N., Chen, R. J., Jin, R. L., Li, S. S., Zhou, S., Sun, T., Li, X. Q., Jin, X., Shen, X., Chen, X., Song, X., Zhou, X., Zhu, Y. X., Huang, Y., Li, Y., Zheng, Y., Zhu, Y., Ma, Y., Huang, Z., Xu, Z., Zhang, Z., Ji, D., Liang, J., Guo, J., Chen, J., Xia, L., Wang, M., Li, M., Zhang, P., Chen, R., Sun, S., Wu, S., Ye, S., Wang, T., Xiao, W. L., An, W., Wang, X., Sun, X., Wang, X., Tang, Y., Zha, Y., Zhang, Z., Ju, Z., Zhang, Z., and Qu, Z. (2025). Deepseek-v3.2: Pushing the frontier of open large language models.

- Du, M., Xu, B., Zhu, C., Wang, X., and Mao, Z. (2025). Deepresearch bench: A comprehensive benchmark for deep research agents.

- Fang, J., Peng, Y., Zhang, X., Wang, Y., Yi, X., Zhang, G., Xu, Y., Wu, B., Liu, S., Li, Z., Ren, Z., Aletras, N., Wang, X., Zhou, H., and Meng, Z. (2025a). A comprehensive survey of self-evolving ai agents: A new paradigm bridging foundation models and lifelong agentic systems.

- Fang, R., Liang, Y., Wang, X., Wu, J., Qiao, S., Xie, P., Huang, F., Chen, H., and Zhang, N. (2025b). Memp: Exploring agent procedural memory.

- Fang, T., Zhang, Z., Wang, X., Wang, R., Qin, C., Wan, Y., Ma, J.-Y., Zhang, C., Chen, J., Li, X., Zhang, H., Mi, H., and Yu, D. (2025c). Cognitive kernel-pro: A framework for deep research agents and agent foundation models training.

- Ghareeb, A. E., Chang, B., Mitchener, L., Yiu, A., Szostkiewicz, C. J., Laurent, J. M., Razzak, M. T., White, A. D., Hinks, M. M., and Rodriques, S. G. (2025). Robin: A multi-agent system for automating scientific discovery.

- Hong, S., Zhuge, M., Chen, J., Zheng, X., Cheng, Y., Wang, J., Zhang, C., Wang, Z., Yau, S. K. S., Lin, Z., Zhou, L., Ran, C., Xiao, L., Wu, C., and Schmidhuber, J. (2024). MetaGPT: Meta programming for a multi-agent collaborative framework. In The Twelfth International Conference on Learning Representations.

- Hu, M., Zhou, Y., Fan, W., Nie, Y., Xia, B., Sun, T., Ye, Z., Jin, Z., Li, Y., Chen, Q., Zhang, Z., Wang, Y., Ye, Q., Ghanem, B., Luo, P., and Li, G. (2025a). Owl: Optimized workforce learning for general multi-agent assistance in real-world task automation.

- Hu, M., Zhou, Y., Fan, W., Nie, Y., Xia, B., Sun, T., Ye, Z., Jin, Z., Li, Y., Zhang, Z., Wang, Y., Ye, Q., Luo, P., and Li, G. (2025b). Owl: Optimized workforce learning for general multi-agent assistance in real-world task automation.

- Hu, Y., Liu, S., Yue, Y., Zhang, G., Liu, B., Zhu, F., Lin, J., Guo, H., Dou, S., Xi, Z., Jin, S., Tan, J., Yin, Y., Liu, J., Zhang, Z., Sun, Z., Zhu, Y., Sun, H., Peng, B., Cheng, Z., Fan, X., Guo, J., Yu, X., Zhou, Z., Hu, Z., Huo, J., Wang, J., Niu, Y., Wang, Y., Yin, Z., Hu, X., Liao, Y., Li, Q., Wang, K., Zhou, W., Liu, Y., Cheng, D., Zhang, Q., Gui, T., Pan, S., Zhang, Y., Torr, P., Dou, Z., Wen, J.-R., Huang, X., Jiang, Y.-G., and Yan, S. (2025c). Memory in the age of ai agents.

- LangChain (2023). Langchain: Build context-aware reasoning applications. [Online]. https://github.com/langchain-ai/langchain.

- Mialon, G., Fourrier, C., Wolf, T., LeCun, Y., and Scialom, T. (2023). Gaia: a benchmark for general ai assistants. In The Twelfth International Conference on Learning Representations.

- OpenAI (2025). Introducing GPT-5 — openai.com. https://openai.com/index/introducing-gpt-5/. [Accessed 16-12-2025].

- Orhan, A. E. (2023). Recognition, recall, and retention of few-shot memories in large language models.

- Ouyang, S., Yan, J., Hsu, I.-H., Chen, Y., Jiang, K., Wang, Z., Han, R., Le, L. T., Daruki, S., Tang, X., Tirumalashetty, V., Lee, G., Rofouei, M., Lin, H., Han, J., Lee, C.-Y., and Pfister, T. (2025). Reasoningbank: Scaling agent self-evolving with reasoning memory.

- Packer, C., Fang, V., Patil, S., Lin, K., Wooders, S., and Gonzalez, J. (2023). Memgpt: Towards llms as operating systems.

- Phan, L., Gatti, A., Han, Z., Li, N., Hu, J., Zhang, H., Zhang, C. B. C., Shaaban, M., Ling, J., Shi, S., et al. (2025). Humanity’s last exam. arXiv preprint arXiv:2501.14249.

- Qin, T., Chen, Q., Wang, S., Xing, H., Zhu, K., Zhu, H., Shi, D., Liu, X., Zhang, G., Liu, J., Jiang, Y. E., Gao, X., and Zhou, W. (2025). Flash-searcher: Fast and effective web agents via dag-based parallel execution.

- Qiu, J., Juan, X., Wang, Y., Yang, L., Qi, X., Zhang, T., Guo, J., Lu, Y., Yao, Z., Wang, H., Liu, S., Jiang, X., Leqi, L., and Wang, M. (2025a). Agentdistill: Training-free agent distillation with generalizable mcp boxes.

- Qiu, J., Qi, X., Zhang, T., Juan, X., Guo, J., Lu, Y., Wang, Y., Yao, Z., Ren, Q., Jiang, X., et al. (2025b). Alita: Generalist agent enabling scalable agentic reasoning with minimal predefinition and maximal self-evolution. arXiv preprint arXiv:2505.20286.

- Rasmussen, P., Paliychuk, P., Beauvais, T., Ryan, J., and Chalef, D. (2025). Zep: A temporal knowledge graph architecture for agent memory.

- Roucher, A., del Moral, A. V., Wolf, T., von Werra, L., and Kaunismäki, E. (2025). ‘smolagents‘: a smol library to build great agentic systems. https://github.com/huggingface/smolagents.

- Shi, D., Cao, J., Chen, Q., Sun, W., Li, W., Lu, H., Dong, F., Qin, T., Zhu, K., Yang, M., Yang, J., Zhang, G., Liu, J., Zhang, C., Wang, J., Jiang, Y. E., and Zhou, W. (2025a). Taskcraft: Automated generation of agentic tasks.

- Shi, Y., Wang, M., Cao, Y., Lai, H., Lan, J., Han, X., Wang, Y., Geng, J., Li, Z., Xia, Z., et al. (2025b). Aime: Towards fully-autonomous multi-agent framework. arXiv preprint arXiv:2507.11988.

- Shinn, N., Labash, B., and Gopinath, A. (2023). Reflexion: an autonomous agent with dynamic memory and self-reflection. arXiv preprint, abs/2303.11366.

- Significant-Gravitas (2023). Autogpt. [Online]. https://github.com/Significant-Gravitas/AutoGPT.

- Sun, H. and Zeng, S. (2025). Hierarchical memory for high-efficiency long-term reasoning in llm agents.

- Suzgun, M., Yuksekgonul, M., Bianchi, F., Jurafsky, D., and Zou, J. (2025). Dynamic cheatsheet: Test-time learning with adaptive memory.

- Tang, X., Hu, T., Ye, M., Shao, Y., Yin, X., Ouyang, S., Zhou, W., Lu, P., Zhang, Z., Zhao, Y., Cohan, A., and Gerstein, M. (2025). Chemagent: Self-updating library in large language models improves chemical reasoning.

- Team, K., Bai, Y., Bao, Y., Chen, G., Chen, J., Chen, N., Chen, R., Chen, Y., Chen, Y., Chen, Y., Chen, Z., Cui, J., Ding, H., Dong, M., Du, A., Du, C., Du, D., Du, Y., Fan, Y., Feng, Y., Fu, K., Gao, B., Gao, H., Gao, P., Gao, T., Gu, X., Guan, L., Guo, H., Guo, J., Hu, H., Hao, X., He, T., He, W., He, W., Hong, C., Hu, Y., Hu, Z., Huang, W., Huang, Z., Huang, Z., Jiang, T., Jiang, Z., Jin, X., Kang, Y., Lai, G., Li, C., Li, F., Li, H., Li, M., Li, W., Li, Y., Li, Y., Li, Z., Li, Z., Lin, H., Lin, X., Lin, Z., Liu, C., Liu, C., Liu, H., Liu, J., Liu, J., Liu, L., Liu, S., Liu, T. Y., Liu, T., Liu, W., Liu, Y., Liu, Y., Liu, Y., Liu, Y., Liu, Z., Lu, E., Lu, L., Ma, S., Ma, X., Ma, Y., Mao, S., Mei, J., Men, X., Miao, Y., Pan, S., Peng, Y., Qin, R., Qu, B., Shang, Z., Shi, L., Shi, S., Song, F., Su, J., Su, Z., Sun, X., Sung, F., Tang, H., Tao, J., Teng, Q., Wang, C., Wang, D., Wang, F., Wang, H., Wang, J., Wang, J., Wang, J., Wang, S., Wang, S., Wang, Y., Wang, Y., Wang, Y., Wang, Y., Wang, Y., Wang, Z., Wang, Z., Wang, Z., Wei, C., Wei, Q., Wu, W., Wu, X., Wu, Y., Xiao, C., Xie, X., Xiong, W., Xu, B., Xu, J., Xu, J., Xu, L. H., Xu, L., Xu, S., Xu, W., Xu, X., Xu, Y., Xu, Z., Yan, J., Yan, Y., Yang, X., Yang, Y., Yang, Z., Yang, Z., Yang, Z., Yao, H., Yao, X., Ye, W., Ye, Z., Yin, B., Yu, L., Yuan, E., Yuan, H., Yuan, M., Zhan, H., Zhang, D., Zhang, H., Zhang, W., Zhang, X., Zhang, Y., Zhang, Y., Zhang, Y., Zhang, Y., Zhang, Y., Zhang, Y., Zhang, Z., Zhao, H., Zhao, Y., Zheng, H., Zheng, S., Zhou, J., Zhou, X., Zhou, Z., Zhu, Z., Zhuang, W., and Zu, X. (2025a). Kimi k2: Open agentic intelligence.

- Team, M. L., Bayan, Li, B., Lei, B., Wang, B., Rong, B., Wang, C., Zhang, C., Gao, C., Zhang, C., Sun, C., Han, C., Xi, C., Zhang, C., Peng, C., Qin, C., Zhang, C., Chen, C., Wang, C., Ma, D., Pan, D., Bu, D., Zhao, D., Kong, D., Liu, D., Huo, F., Li, F., Zhang, F., Dong, G., Liu, G., Xu, G., Li, G., Tan, G., Lin, G., Jing, H., Fu, H., Yan, H., Wen, H., Zhao, H., Liu, H., Shi, H., Hao, H., Tang, H., Lv, H., Su, H., Li, J., Liu, J., Li, J., Yang, J., Wang, J., Yang, J., Tan, J., Sun, J., Zhang, J., Fu, J., Yang, J., Hu, J., Qin, J., Wang, J., He, J., Kuang, J., Mei, J., Liang, K., He, K., Zhang, K., Wang, K., He, K., Gao, L., Shi, L., Ma, L., Qiu, L., Kong, L., Si, L., Lyu, L., Guo, L., Yang, L., Yan, L., Xia, M., Gao, M., Zhang, M., Zhou, M., Shen, M., Tuo, M., Zhu, M., Li, P., Pei, P., Zhao, P., Jia, P., Sun, P., Gu, Q., Li, Q., Li, Q., Huang, Q., Duan, Q., Meng, R., Weng, R., Shao, R., Li, R., Wu, S., Liang, S., Wang, S., Dang, S., Fang, T., Li, T., Chen, T., Bai, T., Zhou, T., Xie, T., He, W., Huang, W., Liu, W., Shi, W., Wang, W., Wu, W., Zhao, W., Zan, W., Shi, W., Nan, X., Su, X., Li, X., Mei, X., Ji, X., Xi, X., Huang, X., Li, X., Fu, X., Liu, X., Wei, X., Cai, X., Chen, X., Liu, X., Li, X., Shi, X., Li, X., Wang, X., Chen, X., Hu, X., Miao, X., He, X., Zhang, X., Hao, X., Cao, X., Cai, X., Yang, X., Feng, Y., Bai, Y., Chen, Y., Yang, Y., Huo, Y., Sun, Y., Lu, Y., Zhang, Y., Zang, Y., Zhai, Y., Li, Y., Yin, Y., Lv, Y., Zhou, Y., Yang, Y., Xie, Y., Sun, Y., Zheng, Y., Wei, Y., Qian, Y., Liang, Y., Tai, Y., Zhao, Y., Yu, Z., Zhang, Z., Yang, Z., Zhang, Z., Xia, Z., Zou, Z., Zeng, Z., Su, Z., Chen, Z., Zhang, Z., Wang, Z., Jiang, Z., Zhao, Z., Wang, Z., and Su, Z. (2025b). Longcat-flash technical report.

- Tran, K.-T., Dao, D., Nguyen, M.-D., Pham, Q.-V., O’Sullivan, B., and Nguyen, H. D. (2025). Multi-agent collaboration mechanisms: A survey of llms.

- Wang, G., Xie, Y., Jiang, Y., Mandlekar, A., Xiao, C., Zhu, Y., Fan, L., and Anandkumar, A. (2023). Voyager: An Open-Ended Embodied Agent with Large Language Models. arXiv e-prints, page arXiv:2305.16291.

- Wang, W., Piękos, P., Nanbo, L., Laakom, F., Chen, Y., Ostaszewski, M., Zhuge, M., and Schmidhuber, J. (2025a). Huxley-gödel machine: Human-level coding agent development by an approximation of the optimal self-improving machine.

- Wang, X., Li, B., Song, Y., Xu, F. F., Tang, X., Zhuge, M., Pan, J., Song, Y., Li, B., Singh, J., Tran, H. H., Li, F., Ma, R., Zheng, M., Qian, B., Shao, Y., Muennighoff, N., Zhang, Y., Hui, B., Lin, J., Brennan, R., Peng, H., Ji, H., and Neubig, G. (2024a). OpenHands: An Open Platform for AI Software Developers as Generalist Agents.

- Wang, Y., Yang, L., Li, G., Wang, M., and Aragam, B. (2025b). Scoreflow: Mastering llm agent workflows via score-based preference optimization.

- Wang, Z., Xu, H., Wang, J., Zhang, X., Yan, M., Zhang, J., Huang, F., and Ji, H. (2025c). Mobile-agent-e: Self-evolving mobile assistant for complex tasks.

- Wang, Z. Z., Mao, J., Fried, D., and Neubig, G. (2024b). Agent workflow memory.

- Wei, J., Sun, Z., Papay, S., McKinney, S., Han, J., Fulford, I., Chung, H. W., Passos, A. T., Fedus, W., and Glaese, A. (2025a). Browsecomp: A simple yet challenging benchmark for browsing agents. arXiv preprint arXiv:2504.12516.

- Wei, J., Yang, Y., Zhang, X., Chen, Y., Zhuang, X., Gao, Z., Zhou, D., Wang, G., Gao, Z., Cao, J., Qiu, Z., He, X., Zhang, Q., You, C., Zheng, S., Ding, N., Ouyang, W., Dong, N., Cheng, Y., Sun, S., Bai, L., and Zhou, B. (2025b). From ai for science to agentic science: A survey on autonomous scientific discovery.

- Wen, L., Fu, D., Li, X., Cai, X., Ma, T., Cai, P., Dou, M., Shi, B., He, L., and Qiao, Y. (2024). Dilu: A knowledge-driven approach to autonomous driving with large language models.

- Wu, J., Yin, W., Jiang, Y., Wang, Z., Xi, Z., Fang, R., Zhang, L., He, Y., Zhou, D., Xie, P., and Huang, F. (2025a). Webwalker: Benchmarking llms in web traversal.

- Wu, Q., Bansal, G., Zhang, J., Wu, Y., Zhang, S., Zhu, E., Li, B., Jiang, L., Zhang, X., and Wang, C. (2023). Autogen: Enabling next-gen llm applications via multi-agent conversation framework.

- Wu, R., Wang, X., Mei, J., Cai, P., Fu, D., Yang, C., Wen, L., Yang, X., Shen, Y., Wang, Y., and Shi, B. (2025b). Evolver: Self-evolving llm agents through an experience-driven lifecycle.

- Wu, Y., Liang, S., Zhang, C., Wang, Y., Zhang, Y., Guo, H., Tang, R., and Liu, Y. (2025c). From human memory to ai memory: A survey on memory mechanisms in the era of llms.

- Yang, C., Yang, X., Wen, L., Fu, D., Mei, J., Wu, R., Cai, P., Shen, Y., Deng, N., Shi, B., Qiao, Y., and Li, H. (2025). Learning on the job: An experience-driven self-evolving agent for long-horizon tasks.

- Ye, S., Yu, C., Ke, K., Xu, C., and Wei, Y. (2025). H2r: Hierarchical hindsight reflection for multi-task llm agents. arXiv preprint arXiv:2509.12810.

- Yin, Z., Sun, Q., Chang, C., Guo, Q., Dai, J., Huang, X., and Qiu, X. (2023). Exchange-of-thought: Enhancing large language model capabilities through cross-model communication.

- Zhang, G., Chen, K., Wan, G., Chang, H., Cheng, H., Wang, K., Hu, S., and Bai, L. (2025a). Evoflow: Evolving diverse agentic workflows on the fly. arXiv preprint arXiv:2502.07373.

- Zhang, G., Fu, M., Wan, G., Yu, M., Wang, K., and Yan, S. (2025b). G-memory: Tracing hierarchical memory for multi-agent systems.

- Zhang, G., Niu, L., Fang, J., Wang, K., Bai, L., and Wang, X. (2025c). Multi-agent architecture search via agentic supernet. arXiv preprint arXiv:2502.04180.

- Zhang, G., Wang, J., Chen, J., Zhou, W., Wang, K., and Yan, S. (2025d). Agentracer: Who is inducing failure in the llm agentic systems?

- Zhang, J., Hu, S., Lu, C., Lange, R., and Clune, J. (2025e). Darwin godel machine: Open-ended evolution of self-improving agents.

- Zhang, J., Xiang, J., Yu, Z., Teng, F., Chen, X., Chen, J., Zhuge, M., Cheng, X., Hong, S., Wang, J., Zheng, B., Liu, B., Luo, Y., and Wu, C. (2024a). AFlow: Automating Agentic Workflow Generation. arXiv:2410.10762.

- Zhang, W., Cui, C., Zhao, Y., Hu, R., Liu, Y., Zhou, Y., and An, B. (2025f). Agentorchestra: A hierarchical multi-agent framework for general-purpose task solving.

- Zhang, W., Li, X., Zhang, Y., Jia, P., Wang, Y., Guo, H., Liu, Y., and Zhao, X. (2025g). Deep research: A survey of autonomous research agents.

- Zhang, W., Zeng, L., Xiao, Y., Li, Y., Cui, C., Zhao, Y., Hu, R., Liu, Y., Zhou, Y., and An, B. (2025h). Agentorchestra: Orchestrating hierarchical multi-agent intelligence with the tool-environment-agent(tea) protocol.

- Zhang, Z., Bo, X., Ma, C., Li, R., Chen, X., Dai, Q., Zhu, J., Dong, Z., and Wen, J.-R. (2024b). A survey on the memory mechanism of large language model based agents. arXiv preprint arXiv:2404.13501.

- Zhang, Z., Dai, Q., Chen, X., Li, R., Li, Z., and Dong, Z. (2025i). Memengine: A unified and modular library for developing advanced memory of llm-based agents.

- Zhao, A., Huang, D., Xu, Q., Lin, M., Liu, Y.-J., and Huang, G. (2024). Expel: Llm agents are experiential learners. In Proceedings of the AAAI Conference on Artificial Intelligence, volume 38, pages 19632–19642.

- Zhao, S., Zhang, H., Lin, S., Li, M., Wu, Q., Zhang, K., and Wei, C. (2025). Pyvision: Agentic vision with dynamic tooling.

- Zheng, B., Fatemi, M. Y., Jin, X., Wang, Z. Z., Gandhi, A., Song, Y., Gu, Y., Srinivasa, J., Liu, G., Neubig, G., and Su, Y. (2025). Skillweaver: Web agents can self-improve by discovering and honing skills.

- Zheng, L., Wang, R., Wang, X., and An, B. (2023). Synapse: Trajectory-as-exemplar prompting with memory for computer control. arXiv preprint arXiv:2306.07863.

- Zhong, W., Guo, L., Gao, Q., Ye, H., and Wang, Y. (2024). Memorybank: Enhancing large language models with long-term memory. In Proceedings of the AAAI Conference on Artificial Intelligence, volume 38, pages 19724–19731.

## 附录 A EvolveLab 实现

EvolveLab 被设计为模块化、可扩展的代码库，以支持对自进化智能体记忆系统的系统研究。它提供统一接口，抽象不同记忆架构的复杂性，从而支持标准化实现、评估与元进化。

### A.1 统一接口与抽象基类

EvolveLab 的基石是 `BaseMemoryProvider` 抽象基类（Abstract Base Class，ABC），它定义所有记忆系统的基本协议。如下方代码所示，该接口强制实现两个映射到模块化设计空间（编码、存储、检索、管理）的主要操作：

- **检索（`provide_memory`）：** 处理上下文感知的记忆回忆。它接收一个 `MemoryRequest`，其中包含当前任务查询、执行上下文和系统状态；返回一个 `MemoryResponse`，其中包含相关 `MemoryItem` 列表。
- **编码与存储（`take_in_memory`）：** 编排新经验的摄取。该方法处理一个封装完整任务执行历史的 `TrajectoryData` 对象，从中抽取结构化洞见或工具（编码），再将其持久化到底层存储介质（存储）。

`take_in_memory` 主要整合编码与存储阶段；负责离线巩固或选择性遗忘的管理功能，通常作为提供器类中的辅助方法实现，或在特定生命周期事件中调用。

```python
class BaseMemoryProvider(ABC):
    """记忆提供器抽象基类。"""

    def __init__(self, memory_type: MemoryType, config: Optional[dict] = None):
        self.memory_type = memory_type
        self.config = config or {}

    @abstractmethod
    def provide_memory(self, request: MemoryRequest) -> MemoryResponse:
        """
        根据查询、上下文和状态检索相关记忆。

        参数：
            request：包含查询、上下文、状态及可选参数的 MemoryRequest。
        返回：
            包含相关记忆的 MemoryResponse。
        """
        pass

    @abstractmethod
    def take_in_memory(self, trajectory_data: TrajectoryData) -> tuple[bool, str]:
        """
        从轨迹数据中存储/摄取新记忆。

        参数：
            trajectory_data：包含查询、轨迹和元数据的 TrajectoryData。
        返回：
            tuple[bool, str]：（记忆摄取是否成功，所吸收记忆的说明）。
        """
        pass

    @abstractmethod
    def initialize(self) -> bool:
        """
        初始化记忆提供器（加载现有数据、建立索引等）。

        返回：
            bool：初始化是否成功。
        """
        pass

    def get_memory_type(self) -> MemoryType:
        """获取该记忆提供器的类型。"""
        return self.memory_type

    def get_config(self) -> dict:
        """获取该记忆提供器的配置。"""
        return self.config.copy()
```

**代码清单 1：记忆提供器的抽象基类。**

### A.2 标准化数据载体

为确保异构记忆设计与智能体框架之间无缝互操作，EvolveLab 使用标准化记忆数据载体。这些结构充当框架的“通用语言”：

- **`MemoryItem`：** 基本信息单元，可表示原始文本、提炼后的洞见或可执行代码（API）。每个条目都包含创建时间戳、置信度分数和来源标识符等元数据。
- **`TrajectoryData`：** 任务执行历史的综合容器，包括初始查询、完整交互轨迹（状态-动作对）和终止奖励，是记忆进化的原始基础材料。
- **`MemoryRequest/Response`：** 检索查询与结果的标准化封装结构，确保任意智能体系统都能与任意记忆提供器交互，无需针对架构进行特定修改。

### A.3 实现示例：ExpeL 与 SkillWeaver

我们对 12 种不同记忆系统的实现体现了 EvolveLab 接口的通用性。两个代表性示例如下：

- **`ExpeLProvider`：** 实现基于对比学习的记忆。其 `take_in_memory` 函数识别成功与失败轨迹，将高层“洞见”提炼为文本。这些洞见存入向量数据库，并在 `provide_memory` 期间通过语义相似度检索，引导智能体避开以往错误。
- **`SkillWeaverProvider`：** 在以工具为中心的设计空间中运行。其 `take_in_memory` 逻辑使用 LLM 从成功轨迹中合成可复用 Python 函数（技能）。这些技能作为可执行的代码级仓库存储，并通过统一 `MemoryItem` 接口动态检索、注入智能体动作空间。

## 附录 B 实验细节

### B.1 数据集细节

本研究使用的 4 个数据集如下：

- **GAIA**（Mialon et al., 2023）包含 165 个任务，其中 Level 1、Level 2、Level 3 问题分别为 53、86、26 个。评估 MemEvolve + SmolAgent 和 MemEvolve + Flash-Searcher 在 GAIA 上的表现时，记忆系统使用 GAIA Level 1 任务及 67 条 TaskCraft 查询进行进化。元进化共进行 3 轮，每轮 40 条轨迹。
- **WebWalkerQA**（Wu et al., 2025a）评估智能体处理复杂多轮网页交互的能力，包含来自 4 个领域、覆盖 1,373 个以上网页的 680 条真实查询。我们采样其中 170 条查询进行评估，采样脚本已在代码库中发布。WebWalkerQA 使用的全部记忆系统均在 TaskCraft 上完成元进化。
- **xBench-DeepSearch（xBench-DS）**（Chen et al., 2025）包含 100 个任务，用于评估智能体规划、工具使用与推理。与 WebWalkerQA 类似，xBench-DS 评估使用的记忆系统全部在 TaskCraft 上完成元进化。
- **TaskCraft**（Shi et al., 2025a）是通过自主数据流水线生成的合成基准。我们收集 300 条查询作为工作子集，并使用其中 120 条进行 3 轮元进化，每轮 40 条查询。SmolAgent 与 Flash-Searcher 的元进化彼此独立进行。

## 附录 C 记忆系统展示

为具体、直观地说明 MemEvolve 进化出的记忆架构，我们在图 8-10 中可视化沿不同进化轨迹发现的 3 个代表性系统。这些示例突出表明：通过修改记忆编码、检索和管理策略，MemEvolve 如何逐步把简单、静态的记忆机制变为表达能力更强、适应性更高的架构。它们共同展示同一元进化框架能够产生的多样记忆设计。

**图 8：MemEvolve 进化出的 Lightweight 记忆系统。** 进化起点是类似 MemoryBank 的最小少样本轨迹记忆，其中每条已完成轨迹都逐字存储。面对新任务，智能体通过向量相似度检索 top-$k$ 条最相似轨迹，并直接以它们作为条件。MemEvolve 逐步将该基线改进为结构更强、具备阶段感知能力的记忆系统。流程图包括摄取与提供两条主路径：按任务是否成功处理轨迹、抽取摘要和工具使用模式、保存成功/失败/工具记忆、生成嵌入与 TF-IDF 索引；检索时进行混合相似度排序、按阶段和上下文筛选，并在需要时生成阶段特定指导。

**图 9：MemEvolve 进化出的 Riva 记忆系统。** 其进化初始化遵循 AgentKB 风格架构，但不继承庞大且昂贵的离线知识库。通过元进化，Riva 在保持轻量、完全在线的同时，发展出更加以智能体为中心的编码与检索策略。摄取路径会在任务成功时抽取策略与操作指导、抽取领域与探测项、形成 `RivaMemoryRecord` 并加入 `RivaStore`；可选地抽取工具函数写入注册表。检索路径先抽取查询领域，使用多信号混合搜索与领域对齐排序，再按阶段门控；之后可生成领域感知的引导与探测，并按阈值决定返回空响应还是携带检索条目的响应。

**图 10：MemEvolve 进化出的 Cerebra 记忆系统。** 它从相同的 AgentKB 风格初始化出发（不含离线知识库），进一步进化为既能从经验中提炼可复用工具，也能提炼抽象知识，并加入工作记忆维护机制，以支持长时程智能体进化。系统初始化图数据库、嵌入与工具，并以核心模式播种初始记忆、建立 TF-IDF 与语义嵌入索引。成功轨迹会经 LLM 摘要并抽取模式，形成文本记忆、节点和边；可选工具函数进入注册表。系统建立语义边，持久化数据库，并按快速/慢速巩固周期执行节点合并、低性能类型剪枝、边权优化与索引重建。检索端使用 TF-IDF + 语义 + 图扩展的混合搜索、自适应阈值过滤，并按需用 LLM 综合文本指导；如启用工具记忆，还会执行工具语义搜索与函数注册表/API 包装，最终组装记忆条目并跟踪成功率。

## Sources

- `papers/agent-memory/MemEvolve - Meta-Evolution of Agent Memory Systems/MemEvolve - Meta-Evolution of Agent Memory Systems.pdf`
