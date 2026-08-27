发表为 ICLR 2026 会议论文

# 通过增量式多轮交互评估 LLM 智能体中的记忆

Yuanzhe Hu\*†, Yu Wang\*†, Julian McAuley

University of California, San Diego

{yuh127, yuw164, jmcauley}@ucsd.edu

数据集  源代码

\* 这些作者对本工作贡献相同。

† 共同通讯作者。

## 摘要

近期针对大语言模型（LLM）智能体的基准主要聚焦于评估推理、规划与执行能力，而另一个关键组件——记忆，涵盖智能体如何记忆、更新与检索长期信息——由于缺乏基准而评估不足。我们将具备记忆机制的智能体称为记忆智能体。本文基于记忆科学与认知科学的经典理论，识别出记忆智能体所需的四项核心能力：准确检索、测试时学习、长程理解与选择性遗忘。现有基准要么依赖有限的上下文长度，要么专为基于书籍的问答等静态长上下文设定而设计，无法反映增量式积累信息的记忆智能体所具有的交互式、多轮本质。此外，尚无现有基准覆盖全部四项能力。我们提出 MemoryAgentBench，一个专为记忆智能体设计的新基准。我们的基准将现有长上下文数据集进行转换，并将新构建的数据集纳入多轮格式，从而有效模拟记忆智能体的增量式信息处理特征。通过精心选择与整理数据集，我们的基准全面覆盖上述四项核心记忆能力，从而为评估记忆质量提供系统且具有挑战性的测试平台。我们评估了多样化的记忆智能体，涵盖从简单的基于上下文与检索增强生成（RAG）系统，到配备外部记忆模块与工具集成的先进智能体。实证结果表明，现有方法尚未掌握全部四项能力，凸显了对 LLM 智能体综合性记忆机制开展进一步研究的必要性。

## 1 引言

大语言模型（LLM）智能体已迅速从概念验证聊天机器人转变为能够编写软件 (Wang et al., 2024c)、控制浏览器 (Müller & Žunič, 2024) 并对多模态输入进行推理的端到端系统。Manus、OWL (Hu et al., 2025)、OpenHands (Wang et al., 2024c) 和 Codex 等框架已能常规地解决复杂的、富工具任务，并在 GAIA (Mialon et al., 2023) 和 SWE-Bench (Jimenez et al., 2023) 等智能体基准上取得最先进的结果。然而这些评估几乎完全聚焦于推理（规划、工具使用、代码合成），而将同等重要的记忆问题（抽象、存储、更新、检索）在很大程度上留待探索。近期以记忆为中心的架构——从 MemoryLLM (Wang et al., 2024d)、SELF-PARAM (Wang et al.) 和 M+ (Wang et al., 2025) 等参数化记忆系统，到 MemGPT (Packer et al., 2023; Lin et al., 2025)、Mem0 (Chhikara et al., 2025)、Cognee (Markovic et al., 2025)、Zep (Rasmussen et al., 2025) 和 MIRIX (Wang & Chen, 2025) 等商业化 token 级记忆方案——采用多样化的策略来存储与检索过去的信息。尽管兴趣日益增长，它们的现实有效性在很大程度上仍属轶事，目前尚无统一基准用于系统评估智能体中的记忆质量。本文将配备记忆机制的智能体称为记忆智能体，其中记忆可以采取多种形式，包括参数、向量、文本历史或外部数据库。本文主要关注利用文本历史与外部数据库的记忆智能体，因为这些方法在现实应用中部署最为普遍。相比之下，编码在模型参数中的记忆 (Wang et al., 2024d; 2025; Yin et al., 2024) 仍主要停留在学术研究范围内，且通常不如闭源 API 模型所配备的专有记忆系统强大。

**图 1：记忆智能体应具备的四项互补能力。**

- **准确检索：** “我去了动物园，看到了大象。”……（很长很长的对话）……“我在动物园看到了什么？”
- **测试时学习：** “$A_1$ 属于类别 1；$B_1$ 属于类别 2；$A_2$ 属于类别 1；$B_2$ 属于类别 2；”……（很长很长的对话）……“$A_5$ 属于哪个类别？$B_5$ 属于哪个类别？”
- **长程理解：** “我正在读这本书：Harry Potter 挥舞魔杖，准备迎接接下来发生的任何事情，……”……（很长很长的对话）……“请帮我概括这个故事。”
- **选择性遗忘：** “我喜欢梨。”……（很长很长的对话）……“我说过我喜欢梨吗？那可能是笔误。我不喜欢水果。我喜欢豌豆。”……“我喜欢梨吗？”

基于记忆与认知科学中的一些经典理论 (James, 1890; McClelland et al., 1995; Anderson & Neely, 1996; Wimber et al., 2015)，我们识别出评估记忆智能体的四项互补能力（示例见图 1）：（1）准确检索（AR）：根据查询提取正确片段的能力。这可以涉及单跳或多跳检索，只要相关信息可以通过一次查询访问。（2）测试时学习（TTL）：在部署期间纳入新行为或获取新技能的能力，无需额外训练。（3）长程理解（LRU）：整合分布在扩展上下文（≥ 100k tokens）中的信息，并回答需要对整个序列有全局理解的问题的能力。（4）选择性遗忘（SF）：在面对矛盾证据时修订、覆盖或删除先前存储信息的技能，与模型编辑和知识遗忘任务中的目标相一致 (Meng et al., 2023; Wang et al., 2024e)。对于这四项能力，我们在附录 B 中提供了更详细的定义。

先前为评估语言模型记忆而开发的数据集存在显著局限。早期基准如 LOCOMO (Maharana et al., 2024)（∼9k tokens）、LooGLE (Li et al., 2023)（∼24k tokens）和 LongBench (Bai et al., 2023)（∼ 20k tokens）的上下文相对较短，已不再能挑战当前模型。更新近的数据集如 NovelQA (Wang et al., 2024a)（∼200k tokens）、NOCHA (Karpinska et al., 2024)（∼127k tokens）、Loong (Wang et al., 2024b)（∼100k tokens）和 ∞-Bench (Zhang et al., 2024)（∼150k tokens）扩展了上下文长度以评估全局推理与检索能力。然而，这些数据集主要为评估长上下文语言模型而非记忆智能体而设计。长上下文基准不能直接用于评估记忆智能体的原因如下。记忆与长上下文之间存在根本区别：记忆是对过去信息的压缩与提炼表示。记忆并非逐字存储所有历史内容，而是选择性地提取显著细节、去除无关信息，并常常纳入从先前经验中得出的新推断。因此，记忆智能体被设计为**增量式**处理上下文——逐块吸收输入，随时间抽象并整合信息，生成新推断，并从累积历史中学习新颖规则。因此，以单一块提供全部上下文的数据集不能直接用于评估记忆智能体。更新近的一项工作 LongMemEval (Wu et al., 2025) 试图通过使用合成长篇对话来解决这一局限，这些对话可以逐会话逐渐注入记忆。尽管如此，其评估框架仍受限于有限的主题多样性和不够真实的交互模式，降低了其对现实世界记忆智能体场景的适用性。

为解决这些局限，我们提出统一基准 MemoryAgentBench，专门用于评估智能体系统中广泛的记忆机制。我们还提供了一个记忆智能体评估框架。在该框架中，向智能体呈现模拟与用户多轮交互的文本输入序列。我们将原先为长上下文 LLM 评估开发的现有数据集进行重构，将输入切分并重建为多个对话块，并按时间顺序增量式地提供给智能体。然而，由于这些数据集并未完全覆盖全部四项目标记忆能力，我们还引入了两个新数据集：EventQA 和 FactConsolidation，分别用于评估准确检索和选择性遗忘。我们的基准包括对最先进商业记忆智能体（如 MIRIX 和 MemGPT）、将全部输入视为记忆的长上下文智能体，以及通过检索方法扩展其记忆的 RAG 智能体的评估。我们考察为长上下文模型和 RAG 开发的技术如何迁移到记忆智能体设定。通过在多样化智能体架构与数据集上提供一致的评估协议，MemoryAgentBench 就四项核心记忆能力提供了对智能体性能的全面洞察。在表 1 中，我们在多个维度上将 MemoryAgentBench 与先前代表性基准进行比较。

我们的贡献总结如下：

- **数据集：** 我们重构现有数据集并创建两个新数据集，以构建一个全面的基准，覆盖四项不同的记忆能力。
- **框架：** 我们提供统一的评估框架，并开源代码库与数据集以鼓励可复现性与进一步研究。
- **实证研究：** 我们实现了具有多样化记忆机制的各种简单智能体，采用商业智能体，并在我们提出的基准上评估这些智能体。我们的结果表明，现有记忆智能体虽然在某些任务上有效，但在某些方面仍面临显著挑战。

## 2 相关工作

### 2.1 长上下文与记忆基准

本节回顾评估基准方面的先前工作，将其分为三个领域：长上下文理解、检索增强生成与记忆智能体。

**长上下文 LLM 基准。** 早期为长上下文评估设计的基准包括 LongBench (Bai et al., 2023) 和 LooGLE (Li et al., 2023)，平均输入长度分别约为 20k 和 24k tokens。更新近的基准——如 ∞-Bench (Zhang et al., 2024)、HELMET (Yen et al., 2024)、RULER (Hsieh et al., 2024)、NOCHA (Karpinska et al., 2024)、NoLiMa (Modarressi et al., 2025) 和 LongBench V2 (Bai et al., 2024)——将上下文长度扩展到超过 100k tokens。虽然这些基准有效评估了模型在单次通过中处理海量信息的能力，但它们主要面向静态长上下文阅读理解，并未反映记忆智能体增量式、多轮的本质。

**检索增强生成基准。** 除纯长上下文评估外，一系列基准针对检索增强生成（RAG），面向开放域 QA、事实核查以及在固定语料库上的文档排序等知识密集型任务，例如 KILT (Petroni et al., 2021) 和 BEIR (Thakur et al., 2021)。更新近的工作在长上下文或特定应用场景下显式评估端到端 RAG 系统，包括 LaRA (Li et al.)、LONG2 RAG (Qi et al., 2024)、FRAMES (Krishna et al., 2025) 和 CRUD-RAG (Lyu et al., 2025)。RAGBench (Friel et al., 2024)、RAGTruth (Niu et al., 2024)、FreshLLM (Vu et al., 2024) 和 T²-RAGBench (Strich et al., 2025) 等大规模基准进一步将评估空间分别扩展到工业手册、幻觉检测、时间敏感的网络 QA，以及文本与表格财务报告。然而，现有 RAG 基准通常假设静态或缓慢变化的知识库以及短时交互，强调检索准确性与依据，而非对记忆智能体至关重要的信息持续更新与选择性遗忘。

**记忆智能体基准。** 更近地，LOCOMO (Maharana et al., 2024)、LongMemEval (Wu et al., 2025)、RealTalk (Lee et al., 2025) 和 StoryBench (Wan & Ma, 2025) 等基准被专门提出用于评估记忆智能体。尽管前景可观，LOCOMO 的对话仍然相对较短（∼9k），而 LongMemEval 使用主题多样性有限的合成对话，使得对话不够真实，可能较难代表现实世界的记忆使用场景。同时，上述基准的评估范围不足以全面评估稳健记忆智能体所必需的四项核心能力——准确检索、测试时学习、长程理解与选择性遗忘。

**表 1：MemoryAgentBench 与现有长期记忆问答基准的比较。#Q 表示问题总数。上下文深度定义为历史中的 token 数。\*论文中未报告，基于我们的近似。StoryBench 的上下文深度在论文中未报告。我们从这些数据集能否全面且有效地评估所提出的各项能力维度进行比较。我们还比较先前工作对记忆智能体的评估覆盖范围，具体而言，即它们是否对不同类别的记忆方法——长上下文智能体（LCA）、RAG 智能体和智能体式记忆（AM）——提供全面评估。**

| 基准 | #Q | 上下文深度 | AR | TTL | LRU | SF | LCA | RAG | AM |
|---|---:|---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| MemoryBank (Zhong et al., 2023a) | 194 | 5k | ✓ | ✓ | ✗ | ✗ | ✓ | ✗ | ✓ |
| LoCoMo (Maharana et al., 2024) | 7512 | 10k | ✓ | ✗ | ✗ | ✗ | ✓ | ✓ | ✗ |
| PerLTQA (Du et al., 2024) | 8593 | 1M\* | ✓ | ✗ | ✗ | ✗ | ✓ | ✓ | ✗ |
| RealTalk (Lee et al., 2025) | 728 | 375k\* | ✓ | ✗ | ✓ | ✗ | ✓ | ✗ | ✗ |
| LongMemEval (Wu et al., 2025) | 500 | 115k, 1.5M | ✓ | ✗ | ✗ | ✗ | ✓ | ✓ | ✗ |
| StoryBench (Wan & Ma, 2025) | 86 | - | ✓ | ✗ | ✓ | ✗ | ✓ | ✗ | ✗ |
| MemoryAgentBench | 2071 | 103k-1.44M | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

### 2.2 具备记忆机制的智能体

记忆机制近来正吸引越来越多的关注 (Wang et al., 2025/02)。LLM 的近期进展已展示出处理扩展上下文长度的能力，范围从 100K 到超过 1 million tokens。例如，GPT-4o (OpenAI, 2025b) 和 Claude 3.7 (Anthropic, 2025) 等模型可以处理大约 100K 到 200K tokens 的输入，而 Gemini 2.0 Pro (DeepMind, 2025) 和 GPT-4.1 系列等模型将这一容量扩展到超过 1 million tokens。这些强大的长上下文能力使得一种简单而有效的记忆形式成为可能：将信息直接存储在上下文窗口中。然而，这种方法本质上受硬限制约束——一旦超出上下文窗口，更早的信息必须被丢弃。

与此同时，RAG 继续作为管理过量上下文的主导范式。通过从更早的上下文中检索相关信息并馈送给 LLM，RAG 使系统能够克服上下文长度限制。例如，OpenAI 近期的记忆功能¹结合了显式用户偏好跟踪与引用先前交互的基于检索的方法。RAG 方法可大致分为三类：**简单 RAG：** 这些方法依赖 TF-IDF、BM25 (Robertson & Walker, 1994) 和 BMX (Li et al., 2024) 等字符串匹配技术，完全非神经，在字符串级相似度上运作。**基于嵌入的 RAG：** 该类利用神经编码器，主要是 transformer，将文本映射为稠密向量表示 (Wu et al., 2022)。早期方法如 DPR (Karpukhin et al., 2020) 和 Contriever (Izacard et al., 2021) 基于 BERT (Devlin et al., 2019)，而 Qwen3-Embedding (Zhang et al., 2025) 等更新近的模型取得了显著改善的检索性能。**结构增强 RAG：** 这些方法用图或树等结构表示增强检索。代表性系统包括 GraphRAG (Edge et al., 2024)、RAPTOR (Sarthi et al., 2024)、HippoRAG-V2 (Gutiérrez et al., 2025)、Cognee、Zep (Rasmussen et al., 2025)、MemoRAG (Qian et al., 2025)、Mem0 (Chhikara et al., 2025)、MemoryOS (Kang et al., 2025)、Memary (kingjulio8238 & Memary contributors, 2024) 和 Memobase (memodb-io & Memobase contributors, 2025)。尽管有效，基于 RAG 的方法在模糊查询、多跳推理和长程理解方面面临挑战。当问题需要整合整个会话的知识或从长的、编码技能的输入中学习时，检索机制——受限于 top-k 最相关段落——可能无法浮现必要信息。为解决这些局限，智能体式记忆智能体引入了一种迭代的、决策驱动的框架。这些智能体并非依赖单次检索，而是动态处理查询、检索证据、反思，并经过多个检索与推理循环进行迭代。示例包括 MemGPT (Packer et al., 2023)、Self-RAG (Asai et al., 2023)、Auto-RAG (Yu et al., 2024)、A-MEM (Xu et al., 2025)、Mem1 (Zhou et al., 2025)、MemAgent (Yu et al., 2025) 和 MIRIX (Wang & Chen, 2025)。这种智能体式设计对解决模糊或多步查询特别有效。尽管如此，这些方法根本上仍受 RAG 局限的约束——即无法完全理解或学习仅通过检索无法访问的长程上下文。

¹ https://openai.com/index/memory-and-new-controls-for-chatgpt/

**表 2：评估数据集概览。我们选择覆盖各种重要长上下文能力的数据集。表中对我们自行构建的数据集加下划线。AvgL.：平均上下文长度（使用 GPT-4o-mini 模型的 tokenizer 测量）。**

| 类别 | 数据集 | 指标 | AvgL. | 描述 |
|---|---|---|---:|---|
| 准确检索 | SH-Doc QA | Accuracy | 197K | 单跳黄金段落检索 QA。 |
| 准确检索 | MH-Doc QA | Accuracy | 421K | 多跳黄金段落检索 QA。 |
| 准确检索 | LongMemEval (S\*) | Accuracy | 355K | 基于对话的 QA。 |
| 准确检索 | <u>EventQA</u> | Accuracy | 534K | 关于角色事件的小说多项选择 QA。 |
| 测试时学习 | BANKING77 | Accuracy | 103K | 银行业务意图分类，77 个标签。 |
| 测试时学习 | CLINC150 | Accuracy | 103K | 意图分类，151 个标签。 |
| 测试时学习 | NLU | Accuracy | 103K | 任务意图分类，68 个标签。 |
| 测试时学习 | TREC Coarse | Accuracy | 103K | 问题类型分类，6 个标签。 |
| 测试时学习 | TREC Fine | Accuracy | 103K | 问题类型分类，50 个标签。 |
| 测试时学习 | Movie Recommendation | Recall@5 | 1.44M | 基于所提供的对话示例推荐电影。 |
| 长程理解 | ∞Bench-Sum | F1-Score | 172K | 带实体替换的小说摘要。 |
| 长程理解 | Detective QA | Accuaracy | 124K | 侦探小说上的长程推理 QA。 |
| 选择性遗忘 | <u>FactConsolidation-SH</u> | Accuracy | 262K | 事实判断中的单跳推理。 |
| 选择性遗忘 | <u>FactConsolidation-MH</u> | Accuracy | 262K | 事实判断中的多跳推理。 |


# 3 MemoryAgentBench

## 3.1 数据集准备

本节说明如何重构现有数据集并构建新数据集，以评估各项能力。所有数据集及其类别见表 2；数据整理细节见附录 B。

**准确检索（AR）数据集。** 我们采用四个数据集评估记忆智能体的准确检索能力，其中三个由现有基准重构，一个为新建数据集：（1）**文档问答：** 这是一项 NIAH 风格的 QA 任务；长文本中包含一个（SH-QA）或多个（MH-QA）能够回答输入问题的文档。智能体必须从扩展上下文中识别并提取相关片段。（2）**LongMemEval：** 该基准在长对话历史上评估记忆智能体。虽然它包含信息抽取（IE）、多会话推理等任务类型，但大多数任务都可重新表述为单次检索问题，即要求智能体从长篇多轮对话中检索正确片段。我们将聊天历史重构为五段长对话（约 355K tokens），并配以 300 个问题（表 2 中的 LongMemEval (S*)）。我们专门创建 LongMemEval (S*)，以增加每个上下文对应的问题数量，减轻为每个问题重新构建记忆所带来的高昂开销。（3）**EventQA（本文提出）：** 我们引入 EventQA 这一推理式 NIAH 任务，评估智能体在长篇叙事中回忆事件时间序列并进行推理的能力。在该数据集中，智能体须阅读一部小说；获得至多五个先前事件后，从一组候选项中选择正确事件。不同于其他需要大量人工标注的长篇叙事文本数据集（Zhang et al., 2024; Xu et al., 2024），我们的数据集通过全自动流水线构建，因而效率更高，也更易扩展。此外，该流水线可直接应用于其他小说类文本。

**测试时学习（TTL）数据集。** 我们通过两类任务评估 TTL：（1）**多类分类（MCC）：** 我们重构了以往 TTL 工作使用的五个分类数据集（Bertsch et al., 2024; Yen et al., 2024）：BANKING77（Casanueva et al., 2020）、CLINC150（Larson et al., 2019）、TREC-Coarse、TREC-Fine（Li & Roth, 2002）以及 NLU（Liu et al., 2019）。每项任务都要求智能体利用上下文中先前出现的带标签样例，将句子映射到类别标签。（2）**推荐：** 根据 Li et al. (2018) 和 He et al. (2023) 的设定，我们构建一个通过对话历史评估电影推荐的数据集。智能体会接触数千轮与电影有关的对话，并须依据长篇交互历史推荐二十部相关电影。

**长程理解（LRU）数据集。** 我们通过两项任务评估 LRU：（1）**小说摘要（Summ.）：** 采用 ∞-Bench（Zhang et al., 2024）的摘要任务 En.Sum。智能体须分析并组织小说情节与人物，随后撰写一篇 1000 至 1200 词的摘要。（2）**Detective QA（Det QA）：** 我们还基于 Detective QA（Xu et al., 2024）构建了一个高难度问题集，其中包含十部小说和 71 个问题；这些问题要求智能体在较长的叙事范围内推理。

**选择性遗忘（SF）数据集。** 为评估智能体能否遗忘过时记忆并在此基础上推理，我们构建了一个名为 FactConsolidation 的新数据集。具体而言，我们使用 MQUAKE（Zhong et al., 2023b）的反事实编辑对构建该基准。每一对都包含一个真实事实和一个改写后的矛盾版本；排序时令改写后的（新）事实出现在原始事实之后，以模拟现实的更新情境。我们将多组编辑对拼接起来，生成长度为 6K、32K、64K、262K 的长上下文。随后沿用 MQUAKE 的原始问题，并将其分为：（1）**FactConsolidation-SH（本文提出）**（SH 表示 Single-Hop），要求直接回忆事实（如“工具 A 是在哪个国家发明的？”）；（2）**FactConsolidation-MH（本文提出）**（MH 表示 Multi-Hop），要求基于多个事实推断（如“人物 B 的配偶去世于何处？”）。提示词要求智能体在发生冲突时优先采用较晚的信息，并依据最终记忆状态推理。这一设定直接评估长序列上选择性遗忘的强度与一致性。

## 3.2 不同类别的记忆智能体

我们评估三类代表常见长期信息处理策略的记忆智能体：长上下文智能体、RAG 智能体和智能体式记忆智能体。三者存储、检索和推理历史输入的方式不同。

（1）**长上下文智能体。** 现代语言模型通常支持 128K 至超过 1M tokens 的扩展上下文窗口。一种直接的记忆策略是维护由最近 tokens 构成的上下文缓冲区。例如，对于上限为 128K tokens 的模型，智能体持续拼接所有传入块，直至总长度超过窗口大小；达到上限后，以 FIFO（先进先出）方式移除最早的块。该设计仅依赖位置上的新近性，并假设模型能对当前上下文窗口进行有效注意。

（2）**RAG 智能体。** 基于 RAG 的智能体把历史信息存入外部记忆池，并按需检索相关内容，从而应对上下文限制。我们考虑三种 RAG 变体：**简单 RAG 智能体：** 将所有输入块保存为原始文本；推理时通过关键词或基于规则的字符串匹配机制检索相关段落。**基于嵌入的 RAG 智能体：** 对每个输入块进行嵌入并保存；查询时对查询进行嵌入，再根据嵌入之间的余弦相似度检索。**结构增强 RAG 智能体：** 摄取全部输入块后，构建结构化表示（如知识图谱或事件时间线），随后基于该结构化记忆回答查询。

（3）**智能体式记忆智能体。** 这类智能体不局限于静态记忆存储，而是采用智能体循环，即迭代推理周期；智能体可以重新表述问题、查找记忆并更新工作记忆。其设计旨在模拟更类似人类的回忆、验证与知识整合过程。

## 3.3 数据集与智能体形式化

**数据集形式化。** 我们将所有数据集标准化为：$c_1,c_2,\cdots,c_n$（块）、$q_1,q_2,\cdots,q_m$（问题）以及 $a_1,a_2,\cdots,a_m$（答案）。其中，$c_i$ 表示第 $i$ 个块；它被包装为一条用户消息，其中含有在顺序输入过程中记住该内容的指令；$c_1,c_2,\cdots,c_n$ 共同构成一段对话。每个块均附带提示智能体记忆其内容的指令，示例见附录 D.1。

在整理 EventQA 和 FactConsolidation 等数据集时，我们有意设计“一份上下文对应多个问题”的场景，从而通过一次顺序注入，多次探测模型的记忆。例如，LME (S*) 用五份上下文对应 300 个问题（见附录 B 表 6）。这一设计反映出一个关键趋势：LLM 支持的上下文窗口越来越长，记忆智能体处理扩展输入的能力也越来越强，因此评估数据集必须相应扩展。仅为一个问题注入 1M tokens 会造成资源浪费，而让同一输入对应多个问题则能显著提高利用率。

**提示词形式与交互协议。** 不同于直接输入原始文本的标准长上下文评估，我们将所有输入块包装在模拟的 User-Assistant 对话中，以显式触发智能体的记忆机制。每个输入块 $c_i$ 前都置入记忆指令（如“Please memorize it and I will ask some questions...”），明确要求存储信息。同时，我们针对每个数据集精心设计指令，确保智能体准确理解任务意图并执行所需操作。尤其重要的是，对选择性遗忘能力，我们在提示词中加入显式护栏：明确告知智能体事实按序号索引，且“newer facts have larger serial numbers.”。智能体必须寻找最新事实来解决冲突（完整模板见附录 D）。

**智能体形式化。** 在我们的框架中，所有智能体都必须逐块接收输入，将其吸收到记忆中，并增量更新记忆。看完全部块后，再回答相关问题。为保证公平比较，每个评估类别内的所有智能体均使用标准化提示词模板，仅在必要时作最小调整。

# 4 实验

## 4.1 实验设置

数据集被分为四类，全部数据集的统计信息见表 6，评估指标及更多细节见表 2。如第 3.2 节所述，智能体分为长上下文智能体、RAG 智能体和智能体式记忆智能体；RAG 智能体又分为简单 RAG 智能体、基于嵌入的 RAG 智能体以及结构增强 RAG 智能体。各记忆智能体的详细介绍见附录 C。

块大小方面，AR 中的 SH-Doc QA、MH-Doc QA、LME(S*) 以及 SF 中的全部任务均使用 512，主要因为这些任务由多个短文本合成长文本。其他任务使用 4096。考虑计算开销与 API 成本，Mem0、Cognee、Zep 和 MIRIX 统一使用 4096。完整块大小设置见附录 E 表 15。

**表 3：总体性能比较。未指定模型时，所有 RAG 智能体和商业记忆智能体均使用 GPT-4o-mini 作为主干，因此高亮 GPT-4o-mini 的性能作为参照。FC-SH 和 FC-MH 分别表示 FactConsolidation Single Hop 与 FactConsolidation Multi Hop。建议以彩色查看。**

| 智能体类型 | SH-QA | MH-QA | LME(S*) | EventQA | AR Avg. | MCC | Recom. | TTL Avg. | Summ. | DetQA | LRU Avg. | FC-SH | FC-MH | SF Avg. | Overall Scores |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| **长上下文智能体** | | | | | | | | | | | | | | | |
| GPT-4o (128K) | 72.0 | 51.0 | 32.0 | 77.2 | 58.1 | 87.6 | 12.3 | 50.0 | 32.2 | 77.5 | 54.9 | 60.0 | 5.0 | 32.5 | 48.8 |
| GPT-4o-mini (128K) | 64.0 | 43.0 | 30.7 | 59.0 | 49.2 | 82.0 | 15.1 | 48.6 | 28.9 | 63.4 | 46.2 | 45.0 | 5.0 | 25.0 | 42.2 |
| Claude-3.7-Sonnet (200K) | 77.0 | 53.0 | 34.0 | 74.6 | 59.7 | 89.4 | 18.3 | 53.9 | 52.5 | 71.8 | 62.2 | 43.0 | 2.0 | 22.5 | 49.6 |
| GPT-5-mini (400K) | 85.0 | 71.0 | 63.3 | 78.2 | 74.4 | 84.0 | 13.2 | 48.6 | 56.3 | 76.1 | 66.2 | 78.0 | 28.0 | 53.0 | 60.6 |
| GPT-4.1-mini (1M) | 83.0 | 66.0 | 55.7 | 82.6 | 71.8 | 75.6 | 16.7 | 46.2 | 41.9 | 56.3 | 49.1 | 36.0 | 5.0 | 20.5 | 46.9 |
| Gemini-2.0-Flash (1M) | 87.0 | 59.0 | 47.0 | 67.2 | 65.1 | 84.0 | 8.7 | 46.4 | 23.9 | 59.2 | 41.6 | 30.0 | 3.0 | 16.5 | 42.4 |
| **GPT-4o-mini** | **64.0** | **43.0** | **30.7** | **59.0** | **49.2** | **82.0** | **15.1** | **48.6** | **28.9** | **63.4** | **46.2** | **45.0** | **5.0** | **25.0** | **42.3** |
| **简单 RAG 智能体** | | | | | | | | | | | | | | | |
| BM25 | 66.0 | 56.0 | 45.3 | 74.6 | 60.5 | 75.4 | 13.6 | 44.5 | 19.0 | 52.1 | 35.6 | 48.0 | 3.0 | 25.5 | 41.5 |
| **基于嵌入的 RAG 智能体** | | | | | | | | | | | | | | | |
| Contriever | 22.0 | 31.0 | 15.7 | 66.8 | 33.9 | 70.6 | 15.2 | 42.9 | 17.2 | 42.3 | 29.8 | 18.0 | 7.0 | 12.5 | 29.8 |
| Text-Embed-3-Small | 60.0 | 44.0 | 48.3 | 63.0 | 53.8 | 70.0 | 15.3 | 42.7 | 17.7 | 54.9 | 36.3 | 28.0 | 3.0 | 15.5 | 37.1 |
| Text-Embed-3-Large | 54.0 | 44.0 | 50.3 | 70.0 | 54.6 | 72.4 | 16.2 | 44.3 | 18.2 | 56.3 | 37.3 | 28.0 | 4.0 | 16.0 | 38.0 |
| Qwen3-Embedding-4B | 57.0 | 47.0 | 43.3 | 71.4 | 54.7 | 78.0 | 12.2 | 45.1 | 14.8 | 59.2 | 37.0 | 29.0 | 3.0 | 16.0 | 38.2 |
| **结构增强 RAG 智能体** | | | | | | | | | | | | | | | |
| RAPTOR | 29.0 | 38.0 | 34.3 | 45.8 | 36.8 | 59.4 | 12.3 | 35.9 | 13.4 | 42.3 | 27.9 | 14.0 | 1.0 | 7.5 | 27.0 |
| GraphRAG | 47.0 | 47.0 | 35.0 | 34.4 | 40.9 | 39.8 | 9.8 | 24.8 | 0.4 | 39.4 | 19.9 | 14.0 | 2.0 | 8.0 | 23.4 |
| MemoRAG | 29.0 | 33.0 | 20.0 | 56.0 | 34.5 | 77.0 | 13.1 | 45.1 | 9.2 | 50.7 | 30.0 | 21.0 | 7.0 | 14.0 | 30.9 |
| HippoRAG-v2 | 76.0 | 66.0 | 50.7 | 67.6 | 65.1 | 61.4 | 10.2 | 35.8 | 14.6 | 57.7 | 36.2 | 54.0 | 5.0 | 29.5 | 41.6 |
| Mem0 | 25.0 | 32.0 | 36.0 | 37.5 | 32.6 | 32.4 | 10.0 | 21.2 | 4.8 | 36.6 | 20.7 | 18.0 | 2.0 | 10.0 | 21.1 |
| Cognee | 31.0 | 26.0 | 29.3 | 26.8 | 28.3 | 35.4 | 10.1 | 22.8 | 2.3 | 29.6 | 16.0 | 28.0 | 3.0 | 15.5 | 20.6 |
| Zep | 44.0 | 25.0 | 38.3 | 42.5 | 37.5 | 62.8 | 12.1 | 37.5 | 4.2 | 28.2 | 16.2 | 7.0 | 3.0 | 5.0 | 24.0 |
| **智能体式记忆智能体** | | | | | | | | | | | | | | | |
| Self-RAG | 35.0 | 42.0 | 25.7 | 31.8 | 33.6 | 11.6 | 12.8 | 12.2 | 0.9 | 35.2 | 18.1 | 19.0 | 3.0 | 11.0 | 18.7 |
| MemGPT | 41.0 | 38.0 | 32.0 | 26.2 | 34.3 | 67.6 | 14.0 | 40.8 | 2.5 | 42.3 | 22.4 | 28.0 | 3.0 | 15.5 | 28.3 |
| MIRIX | 62.0 | 61.0 | 37.3 | 29.8 | 47.5 | 38.4 | 9.8 | 24.1 | 9.9 | 40.8 | 25.4 | 14.0 | 2.0 | 8.0 | 26.2 |
| MIRIX (4.1-mini) | 73.0 | 75.0 | 51.0 | 53.0 | 63.0 | 61.0 | 10.3 | 35.7 | 18.9 | 62.0 | 40.5 | 20.0 | 3.0 | 11.5 | 37.7 |

## 4.2 总体性能比较

表 3 给出各基准上的总体性能。关键发现如下：（1）**RAG 方法在准确检索任务上更优。** 准确检索类别中的多数 RAG 智能体优于主干模型 GPT-4o-mini。这符合直觉：RAG 智能体通常善于提取回答问题所必需的一小段文本。（2）**长上下文模型在测试时学习与长程理解上更优。** 长上下文模型在 TTL 和 LRU 上取得最佳性能。这揭示了 RAG 方法及商业记忆智能体的一项根本局限：它们仍遵循智能体式 RAG 范式，只检索历史上下文的一部分，无法形成对输入的整体理解，更谈不上在其上学习。（3）**全部现有方法在选择性遗忘上都存在局限。** 尽管这是模型编辑社区充分讨论的任务（Mitchell et al., 2022; Fang et al., 2024），遗忘过时记忆对记忆智能体仍是重大挑战。所有方法在多跳情境下均失败，最高准确率仅为 28%。只有长上下文智能体在单跳情境下取得了较为合理的结果。第 4.3.4 节表明，当前推理模型可取得更好性能，但这不改变选择性遗忘仍对全部记忆机制造成重大挑战的结论。

## 4.3 分析与消融研究

本节从五个维度给出实验与分析：输入块大小、检索 top-k、主干模型以及数据集验证。附录还给出计算延迟（附录 E.5）、上下文长度分析（附录 E.4）、延迟和 GPU 显存用量比较（附录 E.5、E.6）、块大小与 top-k 消融的更多细节（附录 E.2、E.3）以及成本—性能估计（附录 I）。

**图 2：不同块大小下 SH-Doc QA 与 ∞-Bench-Sum 的性能。** 图例为 BM25、HippoRAG v2、Qwen3-Embedding-4B、MemGPT。（a）SH-Doc QA 性能：纵轴为 Accurcy (%)，横轴为 Chunk Size（512、1024、2048、4096）。（b）∞Bench-Sum 性能：纵轴为 Model Based F1，横轴为 Chunk Size（512、1024、2048、4096）。

**图 3：检索 top-k 取 2、5、10 时，各基准上的准确率。** 图例为 BM25、Text-Embed-3-Large、HippoRAG-v2；子图为 MH-Doc QA、EventQA、Multi-Class Classification；纵轴为 Accuracy (%)，横轴为 Top-K（2、5、10）。

### 4.3.1 输入块大小消融研究

为理解块大小如何影响性能，尤其是如何影响 RAG 方法和智能体式记忆智能体，我们固定检索块数为 10，同时改变块大小，结果见图 2。在资源允许时，采用更小的块，并在记忆构建期间增加检索调用次数，可提升准确检索（AR）任务的性能。更细粒度的切分可提高所检索信息的相关性，对基于嵌入的方法尤其如此。然而，在需要长程理解（LRU）的任务上，改变块大小反而损害性能。这可能是因为 RAG 本质上不适合需要整合大段连贯上下文信息的任务。

### 4.3.2 检索 TopK 消融研究

表 3 中多数结果将检索块数设为 10；我们还对不同检索规模进行了消融。部分结果见图 3，完整结果见附录 E 表 9。增加检索块数通常能改善多数任务的性能。需要注意，块大小为 4096 tokens 时，检索 10 个块就已产生约 40k tokens 的输入，对模型容量提出很高要求。因此，我们没有评估检索 20 个块的设定。

### 4.3.3 主干模型消融研究

为考察不同主干模型如何影响各种记忆智能体，我们使用三种主干模型开展实验，并从 RAG 智能体与智能体式记忆两类中选取四种代表性方法。完整结果见表 4。

**表 4：三种不同主干 LLM 与四种代表性记忆智能体上的性能比较。每项能力选择一个数据集评估。**

| 智能体类型 | 主干模型 | EventQA | Recom | ∞Bench-Sum | FactCon-SH | Avg. |
|---|---|---:|---:|---:|---:|---:|
| BM25 | GPT-4o-mini | 74.6 | 13.6 | 19.0 | 48.0 | 38.8 |
| BM25 | GPT-4.1-mini | 76.4 | 14.0 | 19.4 | 51.0 | 40.2 |
| BM25 | Gemini-2.0-Flash | 70.8 | 10.0 | 18.9 | 47.0 | 36.7 |
| Text-Embed-3-Small | GPT-4o-mini | 63.0 | 15.3 | 17.7 | 28.0 | 31.0 |
| Text-Embed-3-Small | GPT-4.1-mini | 62.0 | 15.5 | 17.9 | 30.0 | 31.4 |
| Text-Embed-3-Small | Gemini-2.0-Flash | 64.0 | 10.3 | 17.2 | 27.0 | 29.6 |
| GraphRAG | GPT-4o-mini | 34.4 | 9.8 | 0.4 | 14.0 | 14.7 |
| GraphRAG | GPT-4.1-mini | 39.0 | 10.3 | 1.2 | 16.0 | 16.6 |
| GraphRAG | Gemini-2.0-Flash | 36.2 | 7.2 | 0.8 | 13.0 | 14.3 |
| MIRIX | GPT-4o-mini | 29.8 | 9.8 | 9.9 | 14.0 | 15.9 |
| MIRIX | GPT-4.1-mini | 53.0 (23.2↑) | 10.3 (0.5↑) | 18.9 (9.0↑) | 20.0 (6.0↑) | 25.6 (9.7↑) |

结果表明，对 RAG 智能体而言，只要主干足够强，它就不再是主要性能瓶颈。与默认设定相比，升级到 GPT-4.1-mini 等更强模型只带来很小改进。相比之下，表 3 中智能体式记忆类别下的 MIRIX 在采用更强主干后性能显著提升。这意味着未来主干模型的进步可进一步提高智能体式记忆方法的有效性。

### 4.3.4 FactConsolidation 数据集验证

不同模型在该数据集上的性能都极低，因此我们改用更强的推理模型 o4-mini，并检查它在较小版本数据集上的性能，以验证该数据集。结果见表 5。

**表 5：推理模型在 FactConsolidation 数据集上的性能。**

| 模型 | FactCon-SH 6K | 32K | FactCon-MH 6K | 32K |
|---|---:|---:|---:|---:|
| GPT-4o | 92.0 | 88.0 | 28.0 | 10.0 |
| O4-mini | 100.0 | 61.0 | 80.0 | 14.0 |

在 FactCon-SH 的 6K 版本上，两个模型均表现良好，通常能够有效完成任务；上下文增至 32K 后，性能下降。类似地，在 FactCon-MH 的 6K 版本上，更强的 O4-mini 达到 80.0，但上下文窗口达到 32K 后显著降至 14.0。这表明数据集在短上下文设定下可解；当前记忆智能体仍缺乏强大的长程推理能力，因此面对更长历史输入时无法完成该任务。

# 5 结论

本文提出 MemoryAgentBench，一个从四项关键能力——准确检索、测试时学习、长程理解与选择性遗忘——统一评估记忆智能体的基准。以往基准主要关注技能执行或长上下文问答，而 MemoryAgentBench 通过评估智能体如何在多轮交互中存储、更新和使用长期信息，填补了关键空白。为构建该基准，我们重组已有数据集，并提出 EventQA 与 FactConsolidation 两个新数据集，专门对以往工作常忽视的特定记忆行为施加压力。我们在一致的评估协议下评估广泛的智能体，包括长上下文模型、基于 RAG 的系统及商业记忆智能体。结果表明，尽管近期已有进展，当前记忆智能体面对需要动态记忆更新和长程一致性的任务时，仍存在显著局限。本文的一项局限是受预算约束，我们只能在若干较有代表性的记忆智能体上开展实验。未来将为更多记忆智能体补充评估结果。

# 伦理声明

本工作遵守 ICLR Code of Ethics 及相关作者指南；我们评估潜在影响并记录相应缓解措施。我们使用对话和符合许可要求的语料评估 LLM 智能体的记忆；未收集任何个人身份信息或未成年人数据。为降低双重用途风险，我们仅发布经过安全筛查的提示词，并提供使用说明，劝阻将其用于监控类应用。代码将以 MIT License 发布，数据集与基准产物以 CC BY 4.0 发布；第三方材料保留原许可证。

# 可复现性声明

论文录用后，我们将开源本文使用的全部代码与数据。仓库将包括：（i）训练/评估脚本、配置文件及精确提示词；（ii）数据集、带随机种子的生成脚本，可完整再现交互；（iii）端到端运行方案。我们将固定软件依赖，提供容器化环境（Dockerfile 以及 conda/requirements.txt），并报告硬件、CUDA/cuDNN 和 OS 细节，以支持确定性复现，遵循社区关于可复现性声明与产物准备的指导。


# 参考文献

> 为避免影响检索，参考文献题名及书目信息按原文保留。

Michael C. Anderson and James H. Neely. Interference and inhibition in memory retrieval.
In Elizabeth Ligon Bjork and Robert A. Bjork (eds.), Memory, Handbook of Perception
and Cognition, pp. 237–313. Academic Press, San Diego, CA, 2 edition, 1996. URL
https://memorycontrol.net/an1996.pdf.

Anthropic. Claude 3.7 sonnet, 2025. URL https://www.anthropic.com/news/
claude-3-7-sonnet. This announcement introduces Claude 3.7 Sonnet, described as
Anthropic’s most intelligent model to date and the first hybrid reasoning model generally
available on the market.

Akari Asai, Zeqiu Wu, Yizhong Wang, Avirup Sil, and Hannaneh Hajishirzi. Self-rag: Learn-
ing to retrieve, generate, and critique through self-reflection. In The Twelfth International
Conference on Learning Representations, 2023.

Yushi Bai, Xin Lv, Jiajie Zhang, Hongchang Lyu, Jiankai Tang, Zhidian Huang, Zhengxiao
Du, Xiao Liu, Aohan Zeng, Lei Hou, et al. Longbench: A bilingual, multitask benchmark
for long context understanding. arXiv preprint arXiv:2308.14508, 2023.

Yushi Bai, Shangqing Tu, Jiajie Zhang, Hao Peng, Xiaozhi Wang, Xin Lv, Shulin Cao,
Jiazheng Xu, Lei Hou, Yuxiao Dong, et al. Longbench v2: Towards deeper understanding
and reasoning on realistic long-context multitasks. arXiv preprint arXiv:2412.15204, 2024.

Amanda Bertsch, Maor Ivgi, Emily Xiao, Uri Alon, Jonathan Berant, Matthew R Gorm-
ley, and Graham Neubig. In-context learning with long-context models: An in-depth
exploration. arXiv preprint arXiv:2405.00200, 2024.

Iñigo Casanueva, Tadas Temčinas, Daniela Gerz, Matthew Henderson, and Ivan Vulić. Ef-
ficient intent detection with dual sentence encoders. In Tsung-Hsien Wen, Asli Celikyil-
maz, Zhou Yu, Alexandros Papangelis, Mihail Eric, Anuj Kumar, Iñigo Casanueva, and
Rushin Shah (eds.), Proceedings of the 2nd Workshop on Natural Language Processing for
Conversational AI, pp. 38–45, Online, July 2020. Association for Computational Linguis-
tics. doi: 10.18653/v1/2020.nlp4convai-1.5. URL https://aclanthology.org/2020.

nlp4convai-1.5/.

Prateek Chhikara, Dev Khant, Saket Aryan, Taranjeet Singh, and Deshraj Yadav. Mem0:
Building production-ready ai agents with scalable long-term memory. arXiv preprint
arXiv:2504.19413, 2025.
11Published as a conference paper at ICLR 2026
DeepMind. Gemini pro, 2025. URL https://deepmind.google/technologies/gemini/
pro/.

Jacob Devlin, Ming-Wei Chang, Kenton Lee, and Kristina Toutanova. Bert: Pre-training
of deep bidirectional transformers for language understanding. In Proceedings of the 2019
conference of the North American chapter of the association for computational linguistics:
human language technologies, volume 1 (long and short papers), pp. 4171–4186, 2019.

Yiming Du, Hongru Wang, Zhengyi Zhao, Bin Liang, Baojun Wang, Wanjun Zhong,
Zezhong Wang, and Kam-Fai Wong. Perltqa: A personal long-term memory dataset
for memory classification, retrieval, and fusion in question answering. In Proceedings of
the 10th SIGHAN Workshop on Chinese Language Processing (SIGHAN-10), pp. 152–
164, Bangkok, Thailand, August 2024. Association for Computational Linguistics. URL
https://aclanthology.org/2024.sighan-1.18/.

Hermann Ebbinghaus. Memory: A contribution to experimental psychology. Annals of
Neurosciences, 20(4):155–156, 2013. doi: 10.5214/ans.0972.7531.200408. URL https:
//pubmed.ncbi.nlm.nih.gov/25206041/.

Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven
Truitt, Dasha Metropolitansky, Robert Osazuwa Ness, and Jonathan Larson. From lo-
cal to global: A graph rag approach to query-focused summarization. arXiv preprint
arXiv:2404.16130, 2024.

Junfeng Fang, Houcheng Jiang, Kun Wang, Yunshan Ma, Shi Jie, Xiang Wang, Xiangnan
He, and Tat-Seng Chua. Alphaedit: Null-space constrained knowledge editing for language
models. arXiv preprint arXiv:2410.02355, 2024.

Wei Feng and Zongyuan Ge. Generalized category discovery under domain shift: A frequency
domain perspective. Advances in Neural Information Processing Systems, 38:111721–
111749, 2026.

Wei Feng, Sijin Zhou, Yiwen Jiang, and Zongyuan Ge. Prism: Progressive robust learning
for open-world continual category discovery. In The Fourteenth International Conference
on Learning Representations, 2024.

Robert Friel, Masha Belyi, and Atindriyo Sanyal. Ragbench: Explainable benchmark for
retrieval-augmented generation systems. arXiv preprint arXiv:2407.11005, 2024.

Bernal Jiménez Gutiérrez, Yiheng Shu, Weijian Qi, Sizhe Zhou, and Yu Su. From rag to
memory: Non-parametric continual learning for large language models. arXiv preprint
arXiv:2502.14802, 2025.

Chunming He, Kai Li, Yachao Zhang, Guoxia Xu, Longxiang Tang, Yulun Zhang, Zhen-
hua Guo, and Xiu Li. Weakly-supervised concealed object segmentation with sam-based
pseudo labeling and multi-scale feature grouping. NeurIPS, 36, 2024.

Chunming He, Yuqi Shen, Chengyu Fang, Fengyang Xiao, Longxiang Tang, Yulun Zhang,
Wangmeng Zuo, Zhenhua Guo, and Xiu Li. Diffusion models in low-level vision: A survey.

TPAMI, 2025.

Zhankui He, Zhouhang Xie, Rahul Jha, Harald Steck, Dawen Liang, Yesu Feng, Bod-
hisattwa Prasad Majumder, Nathan Kallus, and Julian McAuley. Large language models
as zero-shot conversational recommenders. In Proceedings of the 32nd ACM international
conference on information and knowledge management, pp. 720–730, 2023.

Cheng-Ping Hsieh, Simeng Sun, Samuel Kriman, Shantanu Acharya, Dima Rekesh, Fei Jia,
Yang Zhang, and Boris Ginsburg. RULER: What’s the Real Context Size of Your Long-
Context Language Models?, August 2024. URL http://arxiv.org/abs/2404.06654.

arXiv:2404.06654 [cs].
12Published as a conference paper at ICLR 2026
Mengkang Hu, Yuhang Zhou, Wendong Fan, Yuzhou Nie, Bowei Xia, Tao Sun, Ziyu Ye,
Zhaoxuan Jin, Yingru Li, Zeyu Zhang, Yifeng Wang, Qianshuo Ye, Ping Luo, and Guohao
Li. Owl: Optimized workforce learning for general multi-agent assistance in real-world
task automation, 2025. URL https://github.com/camel-ai/owl.

Gautier Izacard, Mathilde Caron, Lucas Hosseini, Sebastian Riedel, Piotr Bojanowski, Ar-
mand Joulin, and Edouard Grave. Unsupervised dense information retrieval with con-
trastive learning. arXiv preprint arXiv:2112.09118, 2021.

William James. The Principles of Psychology, volume 1. Macmillan, London, 1890. URL
https://books.google.com/books?id=JO1RL9BcI44C.

Carlos E Jimenez, John Yang, Alexander Wettig, Shunyu Yao, Kexin Pei, Ofir Press, and
Karthik Narasimhan. Swe-bench: Can language models resolve real-world github issues?
arXiv preprint arXiv:2310.06770, 2023.

Jiazheng Kang, Mingming Ji, Zhe Zhao, and Ting Bai. Memory os of ai agent. arXiv
preprint arXiv:2506.06326, 2025.

Marzena Karpinska, Katherine Thai, Kyle Lo, Tanya Goyal, and Mohit Iyyer. One thousand
and one pairs: A" novel" challenge for long-context language models. arXiv preprint
arXiv:2406.16264, 2024.

Vladimir Karpukhin, Barlas Oguz, Sewon Min, Patrick SH Lewis, Ledell Wu, Sergey
Edunov, Danqi Chen, and Wen-tau Yih. Dense passage retrieval for open-domain question
answering. In EMNLP (1), pp. 6769–6781, 2020.

kingjulio8238 and Memary contributors. Memary: The open source memory layer for ai
agents, 2024. URL https://github.com/kingjulio8238/Memary. GitHub repository.

Satyapriya Krishna, Kalpesh Krishna, Anhad Mohananey, Steven Schwarcz, Adam Stam-
bler, Shyam Upadhyay, and Manaal Faruqui. Fact, fetch, and reason: A unified evaluation
of retrieval-augmented generation. In Proceedings of the 2025 Conference of the Nations of
the Americas Chapter of the Association for Computational Linguistics: Human Language
Technologies (Volume 1: Long Papers), pp. 4745–4759, 2025.

Stefan Larson, Anish Mahendran, Joseph J. Peper, Christopher Clarke, Andrew Lee, Parker
Hill, Jonathan K. Kummerfeld, Kevin Leach, Michael A. Laurenzano, Lingjia Tang, and
Jason Mars. An evaluation dataset for intent classification and out-of-scope prediction.

In Kentaro Inui, Jing Jiang, Vincent Ng, and Xiaojun Wan (eds.), Proceedings of the
2019 Conference on Empirical Methods in Natural Language Processing and the 9th In-
ternational Joint Conference on Natural Language Processing (EMNLP-IJCNLP), pp.
1311–1316, Hong Kong, China, November 2019. Association for Computational Linguis-
tics. doi: 10.18653/v1/D19-1131. URL https://aclanthology.org/D19-1131/.

Dong-Ho Lee, Adyasha Maharana, Jay Pujara, Xiang Ren, and Francesco Barbieri. Realtalk:
A 21-day real-world dataset for long-term conversation. arXiv preprint arXiv:2502.13270,
2025.

Jiaqi Li, Mengmeng Wang, Zilong Zheng, and Muhan Zhang. Loogle: Can long-context
language models understand long contexts? arXiv preprint arXiv:2311.04939, 2023.

Kuan Li, Liwen Zhang, Yong Jiang, Pengjun Xie, Fei Huang, Shuai Wang, and Minhao
Cheng. Lara: Benchmarking retrieval-augmented generation and long-context llms–no
silver bullet for lc or rag routing. In Forty-second International Conference on Machine
Learning.

Raymond Li, Samira Ebrahimi Kahou, Hannes Schulz, Vincent Michalski, Laurent Char-
lin, and Chris Pal. Towards deep conversational recommendations. Advances in neural
information processing systems, 31, 2018.

Xianming Li, Julius Lipp, Aamir Shakir, Rui Huang, and Jing Li. Bmx: Entropy-weighted
similarity and semantic-enhanced lexical search. arXiv preprint arXiv:2408.06643, 2024.
13Published as a conference paper at ICLR 2026
Xin Li and Dan Roth. Learning question classifiers. In COLING 2002: The 19th Inter-
national Conference on Computational Linguistics, 2002. URL https://aclanthology.

org/C02-1150/.

Yuanhao Li, Mingshan Liu, Hongbo Wang, Yiding Zhang, Yifei Ma, and Wei Tan. DRAFT-
RL: Multi-agent chain-of-draft reasoning for reinforcement learning-enhanced llms. In
Proceedings of the AAAI Conference on Artificial Intelligence, volume 40, pp. 29530–
29537, 2026a. doi: 10.1609/aaai.v40i35.40195. URL https://doi.org/10.1609/aaai.

v40i35.40195.

Yuanhao Li, Hongbo Wang, Xiaotang Shang, Xunzhu Tang, Yiming Cao, and Xuhong Chen.

BoostAPR: Boosting automated program repair via execution-grounded reinforcement
learning with dual reward models, 2026b. URL https://arxiv.org/abs/2605.09134.

Kevin Lin, Charlie Snell, Yu Wang, Charles Packer, Sarah Wooders, Ion Stoica, and
Joseph E Gonzalez. Sleep-time compute: Beyond inference scaling at test-time. arXiv
preprint arXiv:2504.13171, 2025.

Xingkun Liu, Arash Eshghi, Pawel Swietojanski, and Verena Rieser. Benchmarking natural
language understanding services for building conversational agents, 2019. URL https:
//arxiv.org/abs/1903.05566.

Yuanjie Lyu, Zhiyu Li, Simin Niu, Feiyu Xiong, Bo Tang, Wenjin Wang, Hao Wu, Huany-
ong Liu, Tong Xu, and Enhong Chen. Crud-rag: A comprehensive chinese benchmark
for retrieval-augmented generation of large language models. ACM Transactions on In-
formation Systems, 43(2):1–32, 2025.

Adyasha Maharana, Dong-Ho Lee, Sergey Tulyakov, Mohit Bansal, Francesco Barbieri, and
Yuwei Fang. Evaluating very long-term conversational memory of llm agents. arXiv
preprint arXiv:2402.17753, 2024.

Vasilije Markovic, Lazar Obradovic, Laszlo Hajdu, and Jovan Pavlovic. Optimizing the
interface between knowledge graphs and llms for complex reasoning. arXiv preprint
arXiv:2505.24478, 2025.

James L. McClelland, Bruce L. McNaughton, and Randall C. O’Reilly. Why there are
complementary learning systems in the hippocampus and neocortex: Insights from the
successes and failures of connectionist models of learning and memory. Psychological
Review, 102(3):419–457, 1995. doi: 10.1037/0033-295X.102.3.419. URL https://pubmed.

ncbi.nlm.nih.gov/7624455/.

memodb-io and Memobase contributors. Memobase: Profile-based long-term memory for
ai applications, 2025. URL https://github.com/memodb-io/memobase. GitHub reposi-
tory.

Kevin Meng, Arnab Sen Sharma, Alex J. Andonian, Yonatan Belinkov, and David Bau.

Mass-editing memory in a transformer. In ICLR. OpenReview.net, 2023.

Grégoire Mialon, Clémentine Fourrier, Thomas Wolf, Yann LeCun, and Thomas Scialom.

Gaia: a benchmark for general ai assistants. In The Twelfth International Conference on
Learning Representations, 2023.

Eric Mitchell, Charles Lin, Antoine Bosselut, Christopher D. Manning, and Chelsea Finn.

Memory-based model editing at scale. In ICML, volume 162 of Proceedings of Machine
Learning Research, pp. 15817–15831. PMLR, 2022.

Ali Modarressi, Hanieh Deilamsalehy, Franck Dernoncourt, Trung Bui, Ryan A Rossi, Se-
unghyun Yoon, and Hinrich Schütze. Nolima: Long-context evaluation beyond literal
matching. arXiv preprint arXiv:2502.05167, 2025.

Magnus Müller and Gregor Žunič. Browser use: Enable ai to control your browser, 2024.

URL https://github.com/browser-use/browser-use.
14Published as a conference paper at ICLR 2026
Cheng Niu, Yuanhao Wu, Juno Zhu, Siliang Xu, Kashun Shum, Randy Zhong, Juntong
Song, and Tong Zhang. Ragtruth: A hallucination corpus for developing trustworthy
retrieval-augmented language models. In Proceedings of the 62nd Annual Meeting of the
Association for Computational Linguistics (Volume 1: Long Papers), pp. 10862–10878,
2024.


OpenAI. New embedding models and api updates, 2024. URL https://openai.com/index/
new-embedding-models-and-api-updates/.


OpenAI. Introducing gpt-4.1 in the api, 2025a. URL https://openai.com/index/
gpt-4-1/.


OpenAI. Gpt-4o system card, 2025b. URL https://openai.com/index/
gpt-4o-system-card/. This report outlines the safety work carried out prior to
releasing GPT-4o including external red teaming, frontier risk evaluations according to
our Preparedness Framework, and an overview of the mitigations we built in to address
key risk areas.


OpenAI. Introducing gpt-5, 2025c. URL https://openai.com/index/
introducing-gpt-5/.

Charles Packer, Vivian Fang, Shishir_G Patil, Kevin Lin, Sarah Wooders, and Joseph_E
Gonzalez. Memgpt: Towards llms as operating systems. 2023.

Fabio Petroni, Aleksandra Piktus, Angela Fan, Patrick Lewis, Majid Yazdani, Nicola
De Cao, James Thorne, Yacine Jernite, Vladimir Karpukhin, Jean Maillard, et al. Kilt: a
benchmark for knowledge intensive language tasks. In Proceedings of the 2021 Conference
of the North American Chapter of the Association for Computational Linguistics: Human
Language Technologies, pp. 2523–2544, 2021.

Zehan Qi, Rongwu Xu, Zhijiang Guo, Cunxiang Wang, Hao Zhang, and Wei Xu. Long2rag:
Evaluating long-context & long-form retrieval-augmented generation with key point recall.

In Findings of the Association for Computational Linguistics: EMNLP 2024, pp. 4852–
4872, 2024.

Hongjin Qian, Zheng Liu, Peitian Zhang, Kelong Mao, Defu Lian, Zhicheng Dou, and
Tiejun Huang. Memorag: Boosting long context processing with global memory-enhanced
retrieval augmentation. In Proceedings of the ACM on Web Conference 2025, pp. 2366–
2377, 2025.

Preston Rasmussen, Pavlo Paliychuk, Travis Beauvais, Jack Ryan, and Daniel Chalef.

Zep: A temporal knowledge graph architecture for agent memory. arXiv preprint
arXiv:2501.13956, 2025.

Stephen E Robertson and Steve Walker. Some simple effective approximations to the 2-
poisson model for probabilistic weighted retrieval. In SIGIR’94: Proceedings of the Sev-
enteenth Annual International ACM-SIGIR Conference on Research and Development in
Information Retrieval, organised by Dublin City University, pp. 232–241. Springer, 1994.

Parth Sarthi, Salman Abdullah, Aditi Tuli, Shubh Khanna, Anna Goldie, and Christopher D
Manning. Raptor: Recursive abstractive processing for tree-organized retrieval. In The
Twelfth International Conference on Learning Representations, 2024.

Boyu Shi, Shiyu Xia, Xu Yang, Haokun Chen, Zhiqiang Kou, and Xin Geng. Building
variable-sized models via learngene pool. In Proceedings of the AAAI Conference on
Artificial Intelligence, volume 38, pp. 14946–14954, 2024.

Boyu Shi, YiCheng Jiang, Chang Liu, Qiufeng Wang, Xu Yang, and Xin Geng. Chain-based
distillation for effective initialization of variable-sized small language models, 2026. URL
https://arxiv.org/abs/2605.07783.

Jan Strich, Enes Kutay Isgorur, Maximilian Trescher, Chris Biemann, and Martin Semmann.

T 2
-ragbench: Text-and-table benchmark for evaluating retrieval-augmented generation.

arXiv preprint arXiv:2506.12071, 2025.
15Published as a conference paper at ICLR 2026
Peilin Tan, Chuanqi Shi, Dian Tu, and Liang Xie. Magnet: A mamba dual-hypergraph
network for stock prediction via temporal-causal and global relational learning, 2025a.

Peilin Tan, Liang Xie, Churan Zhi, Dian Tu, and Chuanqi Shi. H3m-ssmoes: Hypergraph-
based multimodal learning with llm reasoning and style-structured mixture of experts,
2025b.

Nandan Thakur, Nils Reimers, Andreas Rücklé, Abhishek Srivastava, and Iryna Gurevych.

Beir: A heterogenous benchmark for zero-shot evaluation of information retrieval models.

arXiv preprint arXiv:2104.08663, 2021.

Tu Vu, Mohit Iyyer, Xuezhi Wang, Noah Constant, Jerry Wei, Jason Wei, Chris Tar, Yun-
Hsuan Sung, Denny Zhou, Quoc Le, et al. Freshllms: Refreshing large language models
with search engine augmentation. In Findings of the Association for Computational Lin-
guistics: ACL 2024, pp. 13697–13720, 2024.

Luanbo Wan and Weizhi Ma. Storybench: A dynamic benchmark for evaluating long-term
memory with multi turns. arXiv preprint arXiv:2506.13356, 2025.

Cunxiang Wang, Ruoxi Ning, Boqi Pan, Tonghui Wu, Qipeng Guo, Cheng Deng, Guang-
sheng Bao, Qian Wang, and Yue Zhang. Novelqa: A benchmark for long-range novel
question answering. arXiv preprint arXiv:2403.12766, 2024a.

Minzheng Wang, Longze Chen, Fu Cheng, Shengyi Liao, Xinghua Zhang, Bingli Wu,
Haiyang Yu, Nan Xu, Lei Zhang, Run Luo, et al. Leave no document behind: Benchmark-
ing long-context llms with extended multi-doc qa. In Proceedings of the 2024 Conference
on Empirical Methods in Natural Language Processing, pp. 5627–5646, 2024b.

Xingyao Wang, Boxuan Li, Yufan Song, Frank F Xu, Xiangru Tang, Mingchen Zhuge, Jiayi
Pan, Yueqi Song, Bowen Li, Jaskirat Singh, et al. Openhands: An open platform for ai
software developers as generalist agents. In The Thirteenth International Conference on
Learning Representations, 2024c.

Yu Wang and Xi Chen. Mirix: Multi-agent memory system for llm-based agents. arXiv
preprint arXiv:2507.07957, 2025.

Yu Wang, Xinshuang Liu, Xiusi Chen, Sean O’Brien, Junda Wu, and Julian McAuley. Self-
updatable large language models by integrating context into model parameters. In The
Thirteenth International Conference on Learning Representations.

Yu Wang, Yifan Gao, Xiusi Chen, Haoming Jiang, Shiyang Li, Jingfeng Yang, Qingyu Yin,
Zheng Li, Xian Li, Bing Yin, et al. Memoryllm: Towards self-updatable large language
models. arXiv preprint arXiv:2402.04624, 2024d.

Yu Wang, Ruihan Wu, Zexue He, Xiusi Chen, and Julian McAuley. Large scale knowledge
washing. arXiv preprint arXiv:2405.16720, 2024e.

Yu Wang, Dmitry Krotov, Yuanzhe Hu, Yifan Gao, Wangchunshu Zhou, Julian McAuley,
Dan Gutfreund, Rogerio Feris, and Zexue He. M+: Extending memoryLLM with scalable
long-term memory. In Forty-second International Conference on Machine Learning, 2025.

URL https://openreview.net/forum?id=OcqbkROe8J.

Yu Wang, Chi Han, Tongtong Wu, Xiaoxin He, Wangchunshu Zhou, Nafis Sadeq, Xiusi
Chen, Zexue He, Wei Wang, Gholamreza Haffari, Heng Ji, and Julian J. McAuley. Towards
lifespan cognitive systems. TMLR, 2025/02.

Maria Wimber, Arjen Alink, Ian Charest, Nikolaus Kriegeskorte, and Michael C. Ander-
son. Retrieval induces adaptive forgetting of competing memories via cortical pattern
suppression. Nature Neuroscience, 18(4):582–589, 2015. doi: 10.1038/nn.3973. URL
https://www.nature.com/articles/nn.3973.

Di Wu, Hongwei Wang, Wenhao Yu, Yuwei Zhang, Kai-Wei Chang, and Dong Yu. Long-
memeval: Benchmarking chat assistants on long-term interactive memory. In The Thir-
teenth International Conference on Learning Representations, 2025.
16Published as a conference paper at ICLR 2026
Qiyu Wu, Chongyang Tao, Tao Shen, Can Xu, Xiubo Geng, and Daxin Jiang. Pcl: Peer-
contrastive learning with diverse augmentations for unsupervised sentence embeddings.

arXiv preprint arXiv:2201.12093, 2022.

Wujiang Xu, Kai Mei, Hang Gao, Juntao Tan, Zujie Liang, and Yongfeng Zhang. A-mem:
Agentic memory for llm agents. arXiv preprint arXiv:2502.12110, 2025.

Zhe Xu, Jiasheng Ye, Xiaoran Liu, Xiangyang Liu, Tianxiang Sun, Zhigeng Liu, Qipeng
Guo, Linlin Li, Qun Liu, Xuanjing Huang, et al. Detectiveqa: Evaluating long-context
reasoning on detective novels. arXiv preprint arXiv:2409.02465, 2024.

Howard Yen, Tianyu Gao, Minmin Hou, Ke Ding, Daniel Fleischer, Peter Izsak, Moshe
Wasserblat, and Danqi Chen. Helmet: How to evaluate long-context language models
effectively and thoroughly. arXiv preprint arXiv:2410.02694, 2024.

Zhangyue Yin, Qiushi Sun, Qipeng Guo, Zhiyuan Zeng, Qinyuan Cheng, Xipeng Qiu, and
Xuan-Jing Huang. Explicit memory learning with expectation maximization. In Proceed-
ings of the 2024 Conference on Empirical Methods in Natural Language Processing, pp.
16618–16635, 2024.

Hongli Yu, Tinghong Chen, Jiangtao Feng, Jiangjie Chen, Weinan Dai, Qiying Yu, Ya-Qin
Zhang, Wei-Ying Ma, Jingjing Liu, Mingxuan Wang, et al. Memagent: Reshaping long-
context llm with multi-conv rl-based memory agent. arXiv preprint arXiv:2507.02259,
2025.

Tian Yu, Shaolei Zhang, and Yang Feng. Auto-rag: Autonomous retrieval-augmented gen-
eration for large language models. arXiv preprint arXiv:2411.19443, 2024.

Xinrong Zhang, Yingfa Chen, Shengding Hu, Zihang Xu, Junhao Chen, Moo Hao, Xu Han,
Zhen Thai, Shuo Wang, Zhiyuan Liu, et al. ∞bench: Extending long context evaluation
beyond 100k tokens. In Proceedings of the 62nd Annual Meeting of the Association for
Computational Linguistics (Volume 1: Long Papers), pp. 15262–15277, 2024.

Yanzhao Zhang, Mingxin Li, Dingkun Long, Xin Zhang, Huan Lin, Baosong Yang, Pengjun
Xie, An Yang, Dayiheng Liu, Junyang Lin, et al. Qwen3 embedding: Advancing text
embedding and reranking through foundation models. arXiv preprint arXiv:2506.05176,
2025.

Wanjun Zhong, Lianghong Guo, Qiqi Gao, and Yanlin Wang. Memorybank: Enhancing
large language models with long-term memory. arXiv preprint arXiv:2305.10250, 2023a.

Zexuan Zhong, Zhengxuan Wu, Christopher D Manning, Christopher Potts, and Danqi
Chen. Mquake: Assessing knowledge editing in language models via multi-hop questions.

arXiv preprint arXiv:2305.14795, 2023b.

Zijian Zhou, Ao Qu, Zhaoxuan Wu, Sunghwan Kim, Alok Prakash, Daniela Rus, Jinhua
Zhao, Bryan Kian Hsiang Low, and Paul Pu Liang. Mem1: Learning to synergize memory
and reasoning for efficient long-horizon agents. arXiv preprint arXiv:2506.15841, 2025.


# A 大语言模型（LLM）的使用

在本文写作过程中，我们使用 LLM 辅助润色内容，例如识别语法错误，并对不清晰或可能存在歧义的句子提出修改建议。此外，我们还使用 LLM 生成角色图标，随后将其用于制作主图可视化。

# B 数据集详情

本节详细介绍用于评估四项核心能力的数据集，包括数据整理过程、对应指标、平均上下文长度与简要说明，详情见表 2。

## B.1 准确检索（AR）

### B.1.1 AR 的定义

准确检索信息这一任务已在以往工作中得到广泛研究。在长上下文建模领域，Needle-in-a-Haystack（NIAH）任务被广泛用于评估模型根据给定键，在冗长输入中定位特定值的能力。在 RAG 设定中，这对应于基于文档的 QA：模型必须从一个或多个文档中识别并提取相关片段来回答查询。这些片段可能位于单一位置，也可能分布在多个文档中。本文聚焦智能体式设定，其中“长上下文”或“多个文档”转化为长篇对话。我们将准确检索（AR）定义为：智能体识别并检索可能分散在长对话历史中的重要信息的能力。

### B.1.2 AR 数据集详情

我们使用四个数据集评估记忆智能体的准确检索能力。

**（1）文档问答。** 我们改进了 Hsieh et al. (2024) 的两个 QA 数据集。这些数据集提供多份长度各异的合成上下文，从 3K 到超过 200K tokens。我们从上下文较短的数据中选择 100 个问题。针对每个问题，我们收集其上下文，去除重复的短文档，再打乱并拼接，生成 197K 或 421K tokens 的新长文档，并确保新上下文含有黄金答案段落。由于大多数答案是年份、姓名或是/否等简短信息实体，我们使用子串精确匹配（Substring Exact Match，SubEM）计算 QA 准确率。SubEM 检查预测答案是否与黄金答案形成精确的子串匹配，是问答系统的常用标准。

**（2）LongMemEval。** 这是一个基于对话的 QA 数据集。对于 LME(S*)，我们使用多段历史对话数据，按时间顺序排列后拼接成五份长对话历史，每份长度约为 355K tokens。部分问题的答案为开放形式，因此我们沿用以往方法，由 GPT-4o 判断智能体回答是否满足要求；若满足，则标为 True，最后以满意回答的比例作为评估指标。Wu et al. (2025) 在表 6 中报告，经提示工程设计的 GPT-4o 评判器准确率达到 98.0%，稳定性极高。

**（3）EventQA。** 我们使用 ∞-Bench 中的五本书，每本均超过 390K tokens（使用 gpt-4o-mini tokenizer 计数），通过 SpaCy NER 识别被提及最频繁的十个角色，并用 gpt-4o 抽取关键角色经历的 101 个事件。对每个事件，我们把真实事件与 gpt-4o 生成的五个干扰项配对，构成一道六选一题。智能体获得至多五个先前事件，必须识别正确的后续事件。我们报告每本书 100 道此类题目的平均准确率，最终再报告五本书的平均准确率。

## B.2 测试时学习（TTL）

### B.2.1 TTL 的定义

现实世界智能体的一项关键能力，是通过与用户交互动态获得新技能。这与 LLM 的上下文学习（In-Context Learning，ICL）概念相似：模型从含有少量示例的提示词中学习，通常表现为少样本分类任务。理想情况下，提示词中的示例越多，性能越高。在对话智能体设定中，提示词由对话历史取代。我们将测试时学习（TTL）定义为：智能体直接从对话中学会执行新任务的能力。这一属性对于实现能够在现实部署中持续适应和改进的自演化智能体至关重要。

### B.2.2 TTL 数据集详情

我们通过两类任务评估 TTL：

**（1）多类分类（MCC）。** 我们采用以往 TTL 工作使用的五个分类数据集。数据整理时，我们使用来自不同类别的数千条句子样本，并为每类样本分配一个数字标签。按照 `{sentence} \n Label: {label} \n` 的格式，将全部句子拼接成长上下文，再打乱顺序，防止同类样本过度集中。智能体须参考长上下文，对输入内容进行正确分类，因此使用平均准确率作为评估指标。

**（2）推荐（Recom.）。** 我们拼接原始数据集中多段电影推荐短对话，去除重复对话，生成包含一千多个推荐实例的长上下文。智能体须根据对话内容推荐 20 部电影。我们使用 Recall@5 评估推荐结果，即衡量排名前 5 的推荐电影与标准答案之间的重合程度。

## B.3 长程理解（LRU）

### B.3.1 LRU 的定义

长程理解是指智能体针对长篇对话形成抽象、高层次理解的能力。例如，用户讲述一个很长的故事时，智能体应保留其内容并形成整体理解，而非仅回忆孤立事实。我们将长程理解（LRU）定义为：对长篇输入进行推理，并回答需要理解整体内容而非回忆细节的高层次问题的能力。示例问题是：“概括 Harry Potter 的主要经历。”

### B.3.2 LRU 数据集详情

我们使用 ∞-Bench（Zhang et al., 2024）的摘要任务 En.Sum 评估 LRU。沿用 Yen et al. (2024) 的设定，以 GPT-4o 评估摘要文本；评估过程中，对输入文本的流畅性打分（0 或 1），并将该分数与 F1 分数的点积作为最终评估指标。

## B.4 选择性遗忘（SF）

### B.4.1 SF 的定义

长期交互中，智能体经常面对不断变化或彼此冲突的信息，无论是有关外部世界的信息（如政治领导人变更），还是用户特定事实（如职业变化）。这一挑战与模型编辑（Meng et al., 2023; Fang et al., 2024）和知识遗忘（Wang et al., 2024e）密切相关；后两者关注修改或移除语言模型中的事实知识。我们将选择性遗忘（SF）定义为：智能体检测并解决过时知识与新获信息之间矛盾的能力，从而确保智能体与当前现实和用户状态一致。

SF 与“抽象检索”（Abstractive Retrieval，AR，原文如此）存在两个关键区别。（1）某些需要 SF 的问题无法仅靠 AR 回答。如图 1 所示，检索出全部梨相关事实的智能体，可能无法识别第二条消息中的更新信息。（2）在 AR 中，即便需要多条证据，早期消息仍然相关，应该保留；相比之下，SF 要求识别并丢弃过时或错误的信息。也就是说，AR 要求保留全部相关内容，而 SF 要求覆盖旧事实，以反映最新真相。

### B.4.2 SF 数据集详情

我们使用 MQUAKE（Zhong et al., 2023b）的反事实编辑对。每个包含信息的句子都被赋予一个数字。在每组编辑对中，代表过时信息的句子（干扰项）编号较小，代表较新信息的句子（含有答案者）编号较大。随后按编号顺序将这些句子拼接成长上下文。我们通过两个数据集评估 SF：Single-Hop FactConsolidation 与 Multi-Hop FactConsolidation。智能体在这些任务中的回答大多是信息实体，因此同样使用 SubEM（Substring Exact Match）计算 QA 准确率。

## B.5 基于认知科学的能力论证

准确检索是人类记忆研究的核心；强调回忆保真度的经典遗忘曲线与回忆测试即是例证（Ebbinghaus, 2013）。然而，只关注准确性会掩盖另一条基本轴线：学习与巩固的时间尺度。Ebbinghaus 观察到，若无强化，最初短暂的掌握很少能够持久（Ebbinghaus, 2013）；James (1890) 则区分了初级（即时）记忆和次级（持久）记忆。这些经典区分为我们的测试时学习（通过记忆纳入新信息）与长程理解（持久、整合的知识）概念奠定了基础。与此一致，互补学习系统（Complementary Learning Systems，CLS）框架区分了负责快速情景学习的海马系统，以及负责渐进式结构化知识积累的新皮层系统，强调必须同时评估快速记忆与长时保持（McClelland et al., 1995）。

在获取—巩固轴线之外，另一个同等根本的挑战是选择性遗忘。重叠或矛盾的记忆痕迹会妨碍检索；认知心理学长期把干扰视为遗忘的主要原因（Anderson & Neely, 1996）。神经认知证据进一步表明，大脑会在检索时启用有针对性的控制机制来解决此类干扰（Wimber et al., 2015）。因此，我们将选择性遗忘——处理干扰与矛盾的能力——纳入核心维度。

总之，我们的四类能力——准确检索、测试时学习、长程理解与选择性遗忘——与认知科学及 AI 记忆系统所识别的关键记忆维度一致，覆盖了任何稳健记忆机制在实践中都必须支持的基本能力。值得注意的是，在增量接纳新类别的同时保留既有知识这一挑战，也在持续学习和开放世界发现中受到广泛研究（Feng et al. (2024); Feng & Ge (2026)）；模型必须在分布偏移下平衡稳定性与可塑性，这一张力与我们框架中准确检索和选择性遗忘之间的相互作用直接对应。更广泛地说，从有限或弱监督中学习（He et al. (2024)）以及对新兴方法的系统综述（He et al. (2025)），进一步证明了统一评估框架对于推动 AI 各子领域进步的价值，也为我们的基准设计提供了更多动机。

**表 6：按具体评估方面分类的数据集。此处 1K 为 1024。**

| 能力 | 任务 | 序列数 : QA 数 | 平均长度 |
|---|---|---:|---:|
| 准确检索 | SH-Doc QA | 1 : 100 | 197K |
| 准确检索 | MH-Doc QA | 1 : 100 | 421K |
| 准确检索 | LongMemEval (S*) | 5 : 300 | 355K |
| 准确检索 | EventQA | 5 : 500 | 534K |
| 测试时学习 | BANKING-77 | 1 : 100 | 103K |
| 测试时学习 | CLINC-150 | 1 : 100 | 103K |
| 测试时学习 | NLU | 1 : 100 | 103K |
| 测试时学习 | TREC (Coarse) | 1 : 100 | 103K |
| 测试时学习 | TREC (Fine) | 1 : 100 | 103K |
| 测试时学习 | Movie-Rec Redial | 1 : 200 | 1.44M |
| 长程理解 | ∞Bench-Sum | 100 : 100 | 172K |
| 长程理解 | Detective QA | 10 : 71 | 124K |
| 选择性遗忘 | FactConsolidation-SH | 1 : 100 | 262K |
| 选择性遗忘 | FactConsolidation-MH | 1 : 100 | 262K |

# C 记忆智能体详细说明

本节详细说明实验中使用的记忆智能体。

## C.1 长上下文智能体

我们评估六个现代长上下文 LLM。GPT-4o（OpenAI, 2025b）是高性能、低延迟模型，成本效率优于前代。GPT-4o-mini 则是轻量且经济的替代方案，响应更快、每 token 成本更低，适合大规模评估。GPT-4.1 系列（OpenAI, 2025a）强化了指令遵循能力，并在极大的上下文窗口（据报告可达 1M tokens）上维持强劲性能；考虑较高的 token 成本，我们选择 GPT-4.1-mini 参与评估。GPT-5-mini 是 GPT-5（OpenAI, 2025c）的紧凑推理变体，提供 400K-token 上下文窗口和内置思维链能力，同时降低延迟与成本。Gemini-2.0-Flash（DeepMind, 2025）面向高吞吐量和内置工具使用，提供 1M token 上下文窗口，可高效处理长上下文。Claude-3.7-Sonnet（Anthropic, 2025）是混合推理模型，具备可选且可见的“extended thinking”、强大的数学/编程能力，以及由开发者控制的思考预算。

## C.2 RAG 智能体

我们考虑三种 RAG 变体：简单 RAG 智能体、基于嵌入的 RAG 智能体和结构增强 RAG 智能体。

**（1）简单 RAG 智能体。** 我们实现 BM25（Robertson & Walker, 1994）检索器作为强词法基线：它依据带饱和效应的词频和逆文档频率对文档评分，并由参数 $k_1$ 和 $b$ 控制长度归一化。BM25 在精确匹配查询上仍具竞争力，并以关键词问题上的稳健精确率补充稠密检索器。

**（2）基于嵌入的 RAG 智能体。** Contriever（Izacard et al., 2021）是在大规模文本语料上通过对比学习训练的无监督稠密检索器，无需标注文本对即可实现语义匹配。Text-Embedding-3-Small/Large（OpenAI, 2024）是 OpenAI 的通用嵌入模型，为搜索与检索提供成本—质量权衡（例如 1,536 维与 3,072 维）。Qwen3-Embedding-4B（Zhang et al., 2025）是一个面向多语言检索和长文本理解的 4B 参数嵌入/重排序模型系列。

**（3）结构增强 RAG 智能体。** RAPTOR（Sarthi et al., 2024）通过自底向上的聚类与抽象构建递归摘要层次树，并跨层检索以完成长文档 QA。GraphRAG（Edge et al., 2024）抽取知识图谱与社区层次结构，再执行图感知检索和摘要。MemoRAG（Qian et al., 2025）引入双系统流水线，以轻量“global-memory”模型指导检索，再由更强模型给出最终答案。HippoRAG-v2（Gutiérrez et al., 2025）扩展受海马启发的检索，相比标准 RAG，在事实记忆、意义建构和联想记忆任务上取得改进。

我们还评估三个开源记忆智能体：Mem0、Cognee 和 Zep。Mem0（Chhikara et al., 2025）提供持久的智能体记忆层，用于存储/检索用户特定知识，以增强个性化。Cognee（Markovic et al., 2025）是开源记忆引擎，通过 ECL 流水线构建结构化（图原生）记忆，为图感知 RAG 提供支持。Zep（Rasmussen et al., 2025）是面向智能体的时序知识图谱记忆平台，用于汇集和检索长期对话与业务上下文。除成对图结构外，能够捕捉时间序列中高阶组级关系的超图架构（Tan et al. (2025b); Tan et al. (2025a)），是增强结构增强记忆智能体的一个有前景方向。

## C.3 智能体式记忆智能体

对于智能体式记忆智能体，我们在本基准上评估 Self-RAG（Asai et al., 2023）、MemGPT（Packer et al., 2023）和 MIRIX（Wang & Chen, 2025）。Self-RAG 使用 LLM 作为智能体，由其决定何时检索、检索什么，并批评自身输出。MemGPT 采用层次化记忆管理，在短期存储与长期存储之间调入相关片段，并使用事件驱动中断，在长篇交互中保持一致性与可演化性。MIRIX 采用多智能体记忆架构，包含六种专用记忆类型（Core、Episodic、Semantic、Procedural、Resource、Knowledge Vault），并由协调器编排不同智能体之间的更新与检索。

为保证可比性，我们对上述全部系统统一提示词、工具访问权限以及检索 TopK、输入块大小等设置。


# D 提示词

本节介绍用于记忆构建和任务执行的提示词示例。

## D.1 记忆构建指令

处理长上下文输入时，我们将内容切分为指定大小的块，并把这些块作为记忆依次送入智能体。随后，智能体可以根据查询从记忆中提取相关信息，以辅助执行查询。这种分块方法有助于组织和管理大量上下文信息，使检索与推理更加高效。图 4 给出了若干要求智能体记住相应上下文的示例指令。

**图 4：智能体创建记忆时使用的提示词。**

**文档问答（SH-Doc QA 或 MH-Doc QA）：**

```text
以下是 User 与 Assistant 在 ⟨time⟩ 的一段对话：
⟨User⟩：以下上下文由我读过的文档组成：⟨chunk⟩。请记住它，之后我会基于它提出一些问题。
⟨Assistant⟩：好的！我已经学习了这些文档，并会回答你提出的问题。
```

**LME(S*)：**

```text
以下是 User 与 Assistant 在 ⟨time⟩ 的一段对话：
⟨User⟩：以下上下文是我曾与一个 ChatBoT 交谈的对话历史：⟨chunk⟩。请记住它，之后我会基于它提出一些问题。
⟨Assistant⟩：好的！我已经记住了这段对话历史，并会回答你提出的问题。
```

**EventQA：**

```text
以下是 User 与 Assistant 在 ⟨time⟩ 的一段对话：
⟨User⟩：以下上下文是我读过的小说章节：⟨chunk⟩。请记住它，之后我会基于它提出一些问题。
⟨Assistant⟩：好的！我已经记住了这些小说章节，并会回答你提出的问题。
```

**多类分类（MCC）：**

```text
以下是 User 与 Assistant 在 ⟨time⟩ 的一段对话：
⟨User⟩：以下上下文是我学过的示例：⟨chunk⟩。请记住它，之后我会基于它提出一些问题。
⟨Assistant⟩：好的！我已经记住了这些示例，并会回答你提出的问题。
```

**推荐（Recom）：**

```text
以下是 User 与 Assistant 在 ⟨time⟩ 的一段对话：
⟨User⟩：以下上下文是我曾与一个推荐系统交谈的对话历史：⟨chunk⟩。请记住它，之后我会基于它提出一些问题。
⟨Assistant⟩：好的！我已经记住了这些对话，并会回答你提出的问题。
```

**小说摘要：**

```text
以下是 User 与 Assistant 在 ⟨time⟩ 的一段对话：
⟨User⟩：以下上下文是我读过的小说章节：⟨chunk⟩。请记住它，之后我需要你基于它作摘要。
⟨Assistant⟩：好的！我已经记住了这些小说章节，并会在你提出要求时作摘要。
```

**Detective QA（Det QA）：**

```text
以下是 User 与 Assistant 在 ⟨time⟩ 的一段对话：
⟨User⟩：以下上下文是我读过的小说章节：⟨chunk⟩。请记住它，之后我需要你基于整部小说回答问题。
⟨Assistant⟩：好的！我已经记住了这些小说章节，并会回答你提出的问题。
```

**选择性遗忘（SF）：**

```text
以下是 User 与 Assistant 在 ⟨time⟩ 的一段对话：
⟨User⟩：以下上下文是我学到的事实：⟨chunk⟩。请记住它，之后我需要你依据事实的顺序回答问题。
⟨Assistant⟩：好的！我已经记住了这些事实，并会回答你提出的问题。
```

## D.2 任务执行指令

图 5 给出了在不同数据集上处理输入查询时使用的指令示例。对于部分已有数据集，我们参考了 Hsieh et al. (2024) 和 Wu et al. (2025) 等以往工作的提示词设定。对于 ∞Bench-Sum 数据集，我们还在提示词中插入两个答案示例作为 ⟨demo⟩，帮助智能体更好地理解问题并规范其输出。

**图 5：表 3 中记忆智能体使用的提示词示例。此处 ⟨memory⟩ 指由顺序输入累积得到的文本。**

**文档问答（SH-Doc QA 或 MH-Doc QA）**

```text
上下文如下：⟨memory⟩。 \n 请基于上下文回答问题。只给出答案，不要输出任何其他文字。 \n 现在回答问题：⟨question⟩ \n 答案：
```

**LME(S*)**

```text
以下是对话历史的上下文：⟨memory⟩ \n。请根据相关聊天历史简洁回答问题，如有可能只使用一个短语。\n 当前日期：⟨question_date⟩，\n 现在回答问题：⟨question⟩ \n 答案：
```

**EventQA**

```text
上下文如下：⟨memory⟩。 \n 请基于上述上下文完成以下任务：\n ⟨question⟩ \n 你的任务是根据书中节选，从上述事件中选择接下来发生的事件。回复时只写答案，不要包含任何其他内容。 \n 接下来发生的事件是：
```

**多类分类（MCC）**

```text
上下文如下：⟨memory⟩。 \n 使用上下文中提供的数值标签映射示例，为该上下文分配一个数值标签。只输出 "label: {{label}}"，不要输出其他内容。 \n 问题：⟨question⟩ \n label:
```

**推荐（Recom）**

```text
以下是对话历史的上下文：⟨memory⟩。 \n 假设你是一个电影推荐系统。你需要根据上述对话历史推荐电影。现在我会给你一段用户与你（推荐系统）之间的新对话。请根据这段对话给我 20 个推荐，不要添加额外句子。 \n 例如：\n [Conversation] \n 推荐结果是：\n 1.movie1 \n 2.movie2 \n ...\n 以下是对话：⟨question⟩ \n 推荐结果是：
```

**小说摘要**

```text
书的内容如下：⟨memory⟩ \n 上面给出了一本书，你的任务是为其撰写摘要。请写一篇约 1000 到 1200 词的摘要。只写故事的情节与人物。不要讨论书的主题或背景。不要提供任何分析或评论。 \n ⟨demo⟩ \n 现在请为这本书作摘要。
```

**Detective QA（Det QA）**

```text
上下文如下：⟨memory⟩。 \n 请基于上述上下文完成以下任务：你必须遵循严格的输出格式回答问题。\n ⟨question⟩ \n
```

**Fact Consolidation**

```text
以下是一个包含许多新事实的知识池：⟨memory⟩。 \n 假设你是一个知识管理系统。知识池中的每条事实开头都有一个序号，越新的事实序号越大。 \n 你需要找到最新事实，以解决知识池中的事实冲突。你需要依据这条规则回答问题。你应当给出非常简洁的答案，不要说其他话，并且**只能**依据你记住的知识池，而不是现实世界中的真实事实来回答问题。 \n 例如：\n [Knowledge Pool] \n 问题：根据所提供的 Knowledge Pool，Country R 的现任总统叫什么名字？ \n 答案：Person D。 \n 现在回答问题：根据所提供的 Knowledge Pool，⟨question⟩ \n 答案：
```

# E 详细实验结果

本节给出主文所示结果的详细版本。

## E.1 TTL 的详细结果

表 7 给出了多类分类（MCC）任务的详细结果。对于全部三类任务，基于 RAG 的智能体通常都不如各自的 GPT-4o-mini 主干。这一观察揭示了 RAG 方法固有的若干局限。例如，在 TTL 任务中，基于 RAG 的方法往往难以从记忆中更准确地检索与输入紧密相关的上下文。

**表 7：TTL 数据集上的总体性能比较。所有 RAG 智能体和商业记忆智能体均使用 GPT-4o-mini 作为主干。**

| 智能体类型 | BANKING | CLINIC | NLU | TREC C | TREC F |
|---|---:|---:|---:|---:|---:|
| **长上下文智能体** | | | | | |
| GPT-4o | 96.0 | 96.0 | 90.0 | 87.0 | 69.0 |
| GPT-4o-mini | 93.0 | 93.0 | 87.0 | 73.0 | 66.0 |
| GPT-4.1-mini | 93.0 | 82.0 | 85.0 | 68.0 | 50.0 |
| GPT-5-mini | 88.0 | 92.0 | 88.0 | 88.0 | 64.0 |
| Gemini-2.0-Flash | 91.0 | 90.0 | 84.0 | 88.0 | 67.0 |
| Claude-3.7-Sonnet | 97.0 | 98.0 | 86.0 | 87.0 | 79.0 |
| **GPT-4o-mini** | **93.0** | **93.0** | **87.0** | **73.0** | **66.0** |
| **简单 RAG 智能体** | | | | | |
| BM25 | 89.0 | 89.0 | 84.0 | 62.0 | 53.0 |
| **基于嵌入的 RAG 智能体** | | | | | |
| Contriever | 89.0 | 88.0 | 80.0 | 55.0 | 41.0 |
| Text-Embed-3-Small | 88.0 | 89.0 | 83.0 | 54.0 | 36.0 |
| Text-Embed-3-Large | 90.0 | 91.0 | 80.0 | 55.0 | 46.0 |
| Qwen3-Embedding-4B | 90.0 | 88.0 | 86.0 | 67.0 | 59.0 |
| **结构增强 RAG 智能体** | | | | | |
| RAPTOR | 78.0 | 75.0 | 73.0 | 48.0 | 23.0 |
| GraphRAG | 64.0 | 54.0 | 49.0 | 24.0 | 6.0 |
| MemoRAG | 90.0 | 87.0 | 86.0 | 66.0 | 56.0 |
| HippoRAG-v2 | 81.0 | 86.0 | 73.0 | 38.0 | 29.0 |
| Mem0 | 35.0 | 37.0 | 35.0 | 29.0 | 26.0 |
| Cognee | 34.0 | 42.0 | 42.0 | 41.0 | 18.0 |
| Zep | 83.0 | 74.0 | 70.0 | 50.0 | 37.0 |
| **智能体式记忆智能体** | | | | | |
| Self-RAG | 19.0 | 13.0 | 6.0 | 15.0 | 5.0 |
| MemGPT | 89.0 | 83.0 | 79.0 | 56.0 | 31.0 |
| MIRIX | 42.0 | 53.0 | 49.0 | 36.0 | 12.0 |
| MIRIX(4.1-mini) | 65.0 | 83.0 | 69.0 | 73.0 | 25.0 |

## E.2 输入块大小消融研究结果

表 8 报告在不同块大小和数据集上评估基于 RAG 的智能体所得的详细结果。我们从两组 {512, 4096} 和 {512, 1024, 2048, 4096} 中选择块大小。

**表 8：不同数据集与块大小上的性能比较。此处块大小取自 {512, 1024, 2048, 4096}，基于 RAG 的方法使用 k=10。**

| 方法 | SH-Doc QA 512 | 1024 | 2048 | 4096 | MH-Doc QA 512 | 1024 | 2048 | 4096 | ∞Bench-Sum 512 | 1024 | 2048 | 4096 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| BM25 | 66.0 | 67.0 | 68.0 | 66.0 | 56.0 | 54.0 | 52.0 | 56.0 | 11.5 | 13.2 | 15.2 | 19.0 |
| Qwen3-Embedding-4B | 57.0 | 53.0 | 52.0 | 50.0 | 47.0 | 44.0 | 40.0 | 38.0 | 7.9 | 9.4 | 13.2 | 14.8 |
| HippoRAG-v2 | 76.0 | 70.0 | 57.0 | 49.0 | 66.0 | 63.0 | 51.0 | 38.0 | 4.6 | 6.0 | 10.5 | 14.6 |
| MemGPT | 41.0 | 32.0 | 24.0 | 27.0 | 38.0 | 33.0 | 37.0 | 35.0 | 1.2 | 1.8 | 4.2 | 2.5 |

## E.3 检索 TopK 消融研究结果

表 9 报告所选基于 RAG 的智能体在五个数据集上的详细评估结果。TopK 取值为 {2, 5, 10}。

**表 9：不同检索数量下的性能比较。**

| 方法 | SH-Doc QA R=2 | R=5 | R=10 | MH-Doc QA R=2 | R=5 | R=10 | EventQA R=2 | R=5 | R=10 | TTL (MCC) R=2 | R=5 | R=10 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| BM25 | 50.0 | 60.0 | 66.0 | 49.0 | 54.0 | 56.0 | 66.6 | 71.2 | 74.6 | 67.8 | 74.6 | 75.4 |
| Contriever | 17.0 | 20.0 | 22.0 | 22.0 | 27.0 | 31.0 | 54.4 | 66.8 | 56.0 | 63.0 | 70.0 | 70.6 |
| Text-Embed-3-Large | 36.0 | 47.0 | 54.0 | 37.0 | 41.0 | 44.0 | 51.8 | 62.4 | 70.0 | 59.4 | 69.4 | 72.4 |
| RAPTOR | 22.0 | 27.0 | 29.0 | 30.0 | 36.0 | 38.0 | 45.8 | 41.8 | 40.4 | 56.3 | 57.4 | 59.4 |
| HippoRAG-v2 | 60.0 | 69.0 | 76.0 | 53.0 | 60.0 | 66.0 | 58.8 | 67.6 | 67.4 | 58.8 | 61.4 | 61.4 |
| Self-RAG | 27.0 | 33.0 | 35.0 | 34.0 | 39.0 | 42.0 | 28.2 | 30.6 | 31.8 | 9.0 | 11.6 | 11.6 |

## E.4 不同上下文长度消融研究结果

表 10 报告随输入长度扩展时不同智能体的性能。我们使用 GPT-4o-mini 的分词器测量平均上下文长度，此处 1K 为 1024。对于长上下文智能体，AR 系列任务通常在较小上下文长度（如约 50K tokens）下即可取得令人满意的性能；但随着上下文增长，这些智能体的性能相应下降。相比之下，对于基于 RAG 的智能体 Mem0 与 Cognee，即便上下文较短，其性能也显著低于主干 GPT-4o-mini。

**表 10：不同上下文长度下的性能比较。**

| 方法 | SH-Doc QA 51K | 104K | 197K | MH-Doc QA 51K | 104K | 421K | EventQA 51K | 108K | 534K | FactCon-SH 32K | 64K | 262K | FactCon-MH 32K | 64K | 262K |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| GPT-4o | 91.0 | 84.0 | 72.0 | 72.0 | 68.0 | 51.0 | 96.8 | 94.0 | 77.2 | 88.0 | 85.0 | 60.0 | 10.0 | 13.0 | 5.0 |
| GPT-4o-mini | 84.0 | 83.0 | 64.0 | 58.0 | 54.0 | 43.0 | 90.2 | 85.8 | 59.0 | 63.0 | 58.0 | 45.0 | 10.0 | 5.0 | 5.0 |
| GPT-4.1-mini | 93.0 | 86.0 | 83.0 | 72.0 | 75.0 | 66.0 | 97.0 | 93.8 | 82.6 | 82.0 | 72.0 | 36.0 | 7.0 | 9.0 | 5.0 |
| Gemini-2.0-Flash | 92.0 | 87.0 | 87.0 | 69.0 | 61.0 | 59.0 | 93.4 | 88.6 | 67.2 | 49.0 | 62.0 | 30.0 | 7.0 | 9.0 | 3.0 |
| Claude-3.7-Sonnet | 90.0 | 82.0 | 77.0 | 67.0 | 59.0 | 53.0 | 96.6 | 95.2 | 74.6 | 46.0 | 45.0 | 43.0 | 2.0 | 2.0 | 2.0 |
| Mem0 | 31.0 | 25.0 | 25.0 | 36.0 | 29.0 | 32.0 | 60.8 | 47.0 | 37.5 | 22.0 | 8.0 | 18.0 | 3.0 | 2.0 | 2.0 |
| Cognee | 38.0 | 42.0 | 31.0 | 36.0 | 38.0 | 26.0 | 53.4 | 39.0 | 26.8 | 39.0 | 31.0 | 28.0 | 4.0 | 5.0 | 3.0 |

## E.5 计算延迟分析结果

为说明各种记忆智能体在（1）记忆构建（M.C.）和（2）查询执行（Q.E.）方面的延迟，我们报告各智能体在 MH-QA 与 LME (S*) 上的延迟。实验在一台配有四块 NVDIA L40 GPU 和 AMD EPYC 7713 64-Core CPU 的服务器上完成。HippoRAG-v2 使用 NV-Embed-v2 (7B) 作为嵌入模型。结果见表 11 和表 12。表中显示，块越小，记忆构建所需时间显著越长，HippoRAG-v2、Mem0、Cognee 与 MemGPT 等方法尤其如此。同时，Mem0、Cognee 和 MIRIX 构建记忆时需要极高的资源。

**表 11：长上下文智能体的计算延迟（秒）比较。**

| 模型 | MH-QA | LME (S*) |
|---|---:|---:|
| GPT-4o | 17.0 | 20.1 |
| GPT-4o-mini | 4.9 | 5.4 |
| GPT-4.1-mini | 9.0 | 7.4 |
| Gemini-2.0-Flash | 12.4 | 10.1 |
| Claude-3.7-Sonnet | 23.3 | 22.7 |

**表 12：基于 RAG 的智能体计算延迟（秒）比较。M.C. 表示记忆构建，Q.E. 表示查询执行。* 表示该时间由估算获得。**

| 方法 | MH-QA 512 M.C. | Q.E. | 4096 M.C. | Q.E. | LME (S*) 512 M.C. | Q.E. | 4096 M.C. | Q.E. |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| BM25 | 0.12 | 0.47 | 0.11 | 1.7 | 0.09 | 1.1 | 0.08 | 1.9 |
| Contriever | 7.4 | 0.59 | 1.7 | 2.0 | 6.9 | 0.92 | 1.6 | 1.9 |
| Text-Embed-3-Large | 6.1 | 0.46 | 5.0 | 1.7 | 6.5 | 0.62 | 5.8 | 1.8 |
| Qwen3-Embedding-4B | 367 | 0.49 | 470 | 1.9 | 293 | 0.71 | 372 | 1.8 |
| RAPTOR | 193 | 0.41 | 161 | 0.67 | 108 | 0.60 | 104 | 0.53 |
| GraphRAG | 97.8 | 12.8 | 91.9 | 10.9 | 149 | 7.0 | 88.8 | 7.8 |
| HippoRAG-v2 | 1089 | 0.71 | 380 | 1.71 | 544 | 1.5 | 188 | 3.5 |
| Mem0 | 10804 | 0.79 | 1334 | 0.65 | 18483 | 1.6 | 2946 | 1.7 |
| Cognee | 11890 | 58.7 | 1185 | 4.8 | 4728 | 7.7 | 738 | 4.1 |
| Self-RAG | 11.4 | 3.1 | 8.1 | 2.4 | 5.3 | 0.82 | 5.2 | 1.0 |
| MemGPT | 433 | 9.4 | 101 | 10.5 | 392 | 11.7 | 85.5 | 12.3 |
| MIRIX | 29000* | - | 20171 | 14.1 | 12600* | - | 3258 | 8.7 |
| MIRIX (GPT-4.1-mini) | 28800* | - | 21361 | 16.9 | 9000* | - | 2512 | 9.2 |

## E.6 GPU 显存用量比较

主实验大多使用 LLM API 作为主干，因此无需本地 GPU。实验中的 HippoRAG-v2 (NV-Embed-v2) 与 Qwen3-Embedding-4B 需要在 GPU 上运行嵌入模型。表 13 报告其峰值 GPU 显存用量；全部实验均在单块 A100 80GB GPU 上进行。

**表 13：嵌入模型的峰值 GPU 显存用量（MB）。我们在 MH-QA 数据集上以不同块大小测量显存用量。**

| 智能体 / 块大小 | 512 | 4096 |
|---|---:|---:|
| HippoRAG-v2 (NV-Embed-v2) | 27674 | 60205 |
| Qwen3-Embedding-4B | 16671 | 41262 |

# F 实验设置

本节给出评估所用的实验设置。

## F.1 最大输出 Tokens

表 14 给出了各任务的 token 数量限制。

**表 14：各任务的最大输出 token 限制。**

| 任务 | 最大输出 Tokens |
|---|---:|
| SH-QA / MH-QA | 50 |
| LME(S*) | 100 |
| EventQA | 40 |
| MCC | 20 |
| Movie Recommendation | 300 |
| ∞ Bench-Sum | 1,200 |
| Detective QA | 500 |
| FactConsolidation | 10 |

## F.2 RAG 智能体的设置

在结构增强 RAG 智能体和智能体式记忆智能体中选择嵌入模型时，大多数方法使用 OpenAI 的嵌入模型，如 Text-Embed-3-Small。HippoRAG-v2 则遵循 Gutiérrez et al. (2025) 的相同实验设定，采用 NV-Embed-v2 模型。

主实验实现了三个开源记忆智能体：（1）对于 Mem0，记忆整合期间使用 `memory.add()` 函数，将包含各上下文块内容的消息加入智能体记忆库；查询执行期间通过 `memory.search()` 检索相关记忆元素。随后把检索到的记忆整合进查询，再由 GPT-4o-mini 主干模型处理，以完成所请求的任务。（2）对于 MemGPT，记忆整合阶段使用 `insert_passage()` 函数，将长上下文块注入 Archival Memory 结构。查询执行期间，智能体通过 `send_message()` 函数处理请求；该函数根据归档信息生成恰当回答。（3）对于 Cognee，记忆整合阶段使用 `cognee.add()` 和 `cognee.cognify()`，由输入块构建记忆图；查询执行期间使用 `cognee.search()`，根据输入查询从记忆图中检索与上下文相关的信息。

## F.3 块大小设置

AR 和 SF 使用的合成上下文采用较小块大小（512）。对于 ∞Bench 和 EventQA 等基于连续文本的任务，我们使用较大块大小（4096）。对于 MCC 与 Recom 等任务，考虑其任务特性及计算成本，也使用较大块大小（4096）。对于更耗时且 API 成本更高的记忆构建方法 Mem0、Zep、Cognee 与 MIRIX，所有数据集均统一使用 4096。详细设置见表 15。

**表 15：不同数据集的块大小选择。**

| 块大小 | 数据集 |
|---:|---|
| 512 | SH-QA、MH-QA；FactCon-SH、FactCon-MH；LME(S*) |
| 4096 | ∞Bench-Sum；MCC、Recom；EventQA、Detective QA |


# G 选择性遗忘任务的原理与论证

虽然选择性遗忘任务乍看之下可能显得专门化，甚至具有合成色彩，但它旨在解决长期记忆系统中一个根本而普遍的挑战：维持上下文效率，并缓解过时信息与更新信息之间的干扰。我们从四个紧密关联的维度论证这一任务的设计、新颖性与有效性：

1. **理论必要性：** 对任何记忆系统——无论是生物系统还是人工系统——而言，存储容量本质上都是有限的。自主丢弃（或选择性遗忘）过时、冗余或已被取代的信息，并非小众需求，而是使记忆表征保持精炼、稳健且不受冲突信号干扰的核心前提。我们的任务被设计为一个受控代理，用以评估这项尚未得到充分探索却至关重要的能力。

2. **与以往设定（如知识更新）的区别：** 以往工作虽然探索了知识更新（其重点是用新的冲突事实覆盖旧事实），但我们的工作独特地强调显式、主动地移除非必要信息，以释放认知与上下文空间。这使我们的任务有别于现有事实更新基准；我们把这一框架视为评估真实场景中更复杂记忆管理行为的基础一步。

3. **受控合成设定的合理性：** 我们承认当前任务设定包含合成成分；这一设计选择是有意作出的，并且在方法论上有充分依据。要构建一个具有长期交互历史（超过 100K tokens），同时又能对“哪些信息应被遗忘”给出无歧义、精确标准答案标注的自然数据集，本身就极具挑战，因为现实世界中的遗忘决策往往含糊且依赖上下文。我们的受控合成设定将选择性遗忘能力与混杂因素分离，从而支持在不同记忆智能体架构之间开展可复现、同等条件的比较。

4. **任务的有效性与可解性：** 关键在于，我们确认所提出的任务尽管采用合成设计，却完全有效且可解。正如消融研究所示，推理能力强的长上下文智能体在该数据集的短上下文（6K tokens）版本上取得了近乎完美的性能。这一结果直接证明：长上下文、全长度任务输入上的性能下降，不是源于任务定义本身的缺陷，而是源于当前记忆智能体在长程推理方面的根本局限——具体而言，是它们无法在扩展的交互历史中准确识别并丢弃过时信息。面向 LLM 的执行接地强化学习（Li et al. (2026b)）等新兴方法，可能为未来记忆智能体设计中强化此类长程推理能力提供一条路径。

# H 测试时学习的补充验证

本节对测试时学习（Test-Time Learning，TTL）评估范式的术语与设计作补充论证，并给出额外的零样本基线实验，以验证 TTL 任务的核心前提。

## H.1 术语与设计依据

我们使用“测试时学习”（TTL）一词来描述这样一类任务：它们评估智能体从交互历史中增量获得任务特定技能与规则，并在推理时把这些已学模式应用于未见输入的能力。这一术语及任务设计有三项核心原则作为依据：

1. **区分技能习得与静态信息检索：** 我们明确区分 TTL 与以检索为重点的标准任务（如准确检索、长程理解），以突出 TTL 的独特重点是学习，而非事实回忆。在检索任务中，智能体的核心目标是从交互历史中取回预先定义的静态事实。相比之下，TTL 任务（如 Banking77 上的多类分类、个性化电影推荐）要求智能体从对话历史中依序出现的带标签样例里归纳潜在的分类规则、偏好模式与任务图式，随后将这些模式泛化到全新、样本外的输入。这种操作化方式与学习的经典定义直接一致：根据累积经验更新行为策略；这一过程完全发生在测试时交互期间，不在目标任务上进行预训练或微调。这一通过交互学习的概念也与借助多智能体协作推理增强 LLM 能力的更广泛努力相关（Li et al. (2026a)）；在此类工作中，模型同样必须从有限或间接信号中抽取可泛化的模式，而不是依赖预定义知识。

2. **在线学习的受控操作化：** 完全动态的在线学习设定会包含记忆更新、检索、任务执行以及反馈驱动的记忆改进所构成的交错循环，但这种设定会引入大量混杂变量，使智能体的记忆能力无法与无关的推理、规划或执行错误分离。为稳健、可复现且无偏地评估纯粹由记忆赋能的学习能力，我们采用两阶段协议，在保留在线学习核心的同时排除混杂因素：

   - **习得阶段：** 智能体增量处理一系列带标签的任务样例，模拟现实智能体交互中经验随时间积累的过程。
   - **评估阶段：** 测试智能体能否把习得阶段获得的规则与模式应用于留出且未见的输入。

   这一设计确保模型之间的性能差异可直接归因于它们利用长期交互历史进行学习的能力，而不是动态反馈循环造成的伪因素。

3. **智能体记忆研究的基础框架：** 具备记忆能力的 LLM 智能体研究仍处于早期阶段，因此，我们的 TTL 评估框架为记忆智能体的原位学习提供了一种标准化、可复现的操作化方案。当前协议虽然为保证评估稳定性而简化了完整在线学习循环，却成功捕捉到现实世界自我改进智能体所需的核心能力：仅凭记忆过往交互并从中泛化来提升任务性能。未来工作将把这一框架扩展到更复杂、完全交错的在线学习场景。

## H.2 零样本基线验证实验

TTL 任务设计的一项核心前提是：完整记忆设定下的性能提升来自智能体从历史样例中学习的能力，而不是基础 LLM 预训练所编码的先验知识。为验证这一前提，我们进行了零样本基线评估：模型在无法访问历史样例序列（即没有测试时学习机会）的条件下接受 TTL 任务测试。

表 16 给出了三个主流 LLM 在两项核心 TTL 任务上的零样本性能：多类分类（MCC，Banking77）与个性化电影推荐（Recom.）。我们还列出 GPT-4o-mini 的完整记忆性能以供直接比较，从而量化禁用测试时学习后的性能下降。

**表 16：零样本设定下测试时学习（TTL）任务的性能，并与完整记忆设定比较。所有指标均为任务准确率，数值越高表示性能越好。**

| 模型 | MCC | Recom. | Avg. |
|---|---:|---:|---:|
| **零样本设定（无法访问历史样例）** | | | |
| GPT-4o-mini | 0.6 | 6.1 | 3.4 |
| GPT-4.1-mini | 0.8 | 5.7 | 3.3 |
| Gemini-2.0-Flash | 0.0 | 5.5 | 2.8 |
| **完整记忆设定（可访问历史样例）** | | | |
| GPT-4o-mini w/ full context | 82.0 | 15.1 | 48.6 |

结果证实了我们的核心假设：所有模型在零样本设定下均表现得接近随机水平，两项任务的平均准确率都低于 4%。相比之下，在提供完整历史样例序列时，GPT-4o-mini 的平均准确率达到 48.6%，绝对提升 45.2 个百分点。这一鲜明的性能差距表明，基础 LLM 没有能够开箱即用地解决这些长尾任务的实质性先验知识；完整记忆设定下的全部性能增益，都明确来自智能体从所提供交互历史中学习的能力。这验证了 TTL 基准确测量的是测试时学习能力，而不是预训练知识或虚假的模式匹配。

# I 成本—性能分析

## I.1 方法

为了对每种智能体架构的实际局限作出符合现实的评估，我们在计算成本时假设已启用上下文缓存（Context Caching，OpenAI 等现代 API 的标准功能）；这会显著降低处理共享历史的长上下文（LC）模型的成本。我们比较三种代表性架构：长上下文（LC）模型、RAG 智能体以及智能体式记忆系统 MIRIX。

**定价依据：** 成本依据 OpenAI 截至 2025 年 11 月的定价计算：

- GPT-4o-mini：$0.15 / 1M 输入 tokens，$0.60 / 1M 输出 tokens
- GPT-4.1-mini：$0.40 / 1M 输入 tokens，$1.60 / 1M 输出 tokens

**上下文缓存：** 假设在共享上下文上依次提问，我们对长上下文智能体采用缓存输入价格（GPT-4o-mini 优惠 50%，GPT-4.1-mini 优惠 75%）。

**设定：** RAG 智能体使用 Top-K=10。嵌入索引成本属于一次性支出，故不计入。

## I.2 成本—性能结果

表 17 报告四个代表性数据集上每个查询的摊销推理成本（USD）及相应性能指标（Accuracy/Score）；这些数据集的上下文长度和推理复杂度各不相同。

**表 17：每个查询的估计摊销成本与性能。成本在共享同一上下文的问题集合上摊销。性能分数采用主文定义的指标。**

| 模型/架构 | MH-Doc QA 估计成本（USD） | 性能 | MCC 估计成本（USD） | 性能 | Detective QA 估计成本（USD） | 性能 | FC-SH 估计成本（USD） | 性能 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| GPT-4o-mini | $0.01 | 43.0 | $0.008 | 82.0 | $0.01 | 63.4 | $0.01 | 45.0 |
| GPT-4.1-mini | $0.043 | 66.0 | $0.011 | 75.6 | $0.013 | 56.3 | $0.027 | 36.0 |
| RAG Agents (BM25 + 4o-mini) | <$0.001 | 56.0 | $0.006 | 75.4 | $0.006 | 52.1 | <$0.001 | 48.0 |
| MIRIX (4.1-mini) | $0.016 | 75.0 | $0.010 | 61.0 | $0.011 | 62.0 | $0.019 | 20.0 |

## I.3 关键洞见

1. **RAG 的效率—推理权衡：** BM25 等 RAG 智能体成本高效（每个查询 < $0.001 – $0.006），但在需要全局推理的任务上表现欠佳（例如 Detective QA 得分 52.1，而 LC 模型为 63.4）。这限制了它们在复杂分析场景中的效用。

2. **长上下文扩展的高昂成本：** 长上下文模型虽然强大，但采用更强主干时成本会急剧上升。从 GPT-4o-mini 升级到 GPT-4.1-mini，成本大约增加到四倍（MH-Doc QA 从 $0.010 升至 $0.043）；即便采用缓存，高端长上下文部署仍然昂贵。缓解此类成本障碍的一个有前景方向，是高效的模型构建与知识蒸馏：通过可学习的知识迁移（Shi et al. (2024)）和基于链的蒸馏（Shi et al. (2026)），把大模型能力压缩进更小、更高效的架构。这类技术可让记忆智能体以显著降低的计算成本利用强大的主干推理能力，使长上下文记忆更适合现实部署。

3. **智能体式记忆（MIRIX）是最佳折中：** 一项关键发现是，在 MH-Doc QA 等记忆密集型任务上，MIRIX（使用 GPT-4.1-mini）的成本（$0.016）低于原始 GPT-4.1-mini 长上下文设定（$0.043），同时性能更优（75.0 对 66.0）。这表明，智能体式记忆机制能够成功解除性能与线性上下文成本之间的耦合，为高性能应用提供可扩展方案。

# J 严格计算匹配的比较实验

## J.1 实验设置

为回应预算效应可能混淆架构比较的担忧，我们使用最强主干（GPT-4.1-mini），在 Banking77（TTL）和 Book Summarization（LRU）上开展了严格计算匹配的消融研究。我们定义三个预算级别：

- **低（约 4K tokens）：** 将长上下文（LC）模型限制为截断上下文；把 RAG 和 MIRIX 限制为 Top-K=1 个块，以匹配 token 数。
- **中（约 40K tokens）：** 对所有架构采用中等预算；把 RAG 和 MIRIX 限制为 Top-K=10 个块。
- **高（约 100K+ tokens）：** 令 RAG 和 MIRIX 检索较大 Top-K 的块，以匹配 LC 的完整上下文预算。

为提高计算效率，Book Summarization 使用随机抽取的 30 本书子集评估。我们依据处理的 token 总数比较总体计算负载，而不严格对齐前向传播次数，因为强制智能体式模型仅进行一次前向传播会剥夺其核心推理能力。

**表 18：TTL 与 LRU 任务上计算匹配的实验结果（Accuracy/Score %）。**

| 任务 | 预算设定 | Long-Context (GPT-4.1-mini) | RAG (BM25) | Agentic (MIRIX) |
|---|---|---:|---:|---:|
| TTL (Banking77) | 低（约 4K / Top-K=1） | 74.0 | 83.0 | 52.0 |
| TTL (Banking77) | 中（约 40K / Top-K=10） | 90.0 | 89.0 | 65.0 |
| TTL (Banking77) | 高（约 104K） | 93.0 | 88.0 | 67.0 |
| LRU (Book Summarization) | 低（约 4K / Top-K=1） | 8.2 | 7.9 | 8.4 |
| LRU (Book Summarization) | 中（约 40K / Top-K=10） | 16.4 | 15.8 | 18.7 |
| LRU (Book Summarization) | 高（约 113K） | 39.7 | 38.0 | 38.8 |

关键发现如下：

- **TTL：效率—容量权衡。** 在低（约 4K）预算下，RAG（83.0）显著优于 LC 模型（74.0）；这说明 RAG 能通过精确检索相关样例，以更高的结构效率完成模式匹配，而 LC 模型则受到截断的严重影响。预算增加到中等水平（约 40K）时，性能趋同（约 90.0）。在高预算下，LC 模型提升到 93.0，而 RAG 因检索噪声达到饱和并略降至 88.0。这表明 RAG 虽然高效，但预算允许时，LC 模型具有更高的容量上限。

- **LRU：信息阈值效应。** 对于全局推理任务，性能呈现清晰的阈值行为。在低、中预算下，所有架构均失败（得分 <20.0），说明无论采用何种方法，部分信息都不足以完成全书摘要。只有在高预算（能够访问全文）下，所有模型才取得有意义的性能（约 39.0），且 RAG 几乎追平 LC 模型。这证明，对于 LRU 任务，成功取决于是否达到完整信息阈值，而非架构本身。

# K 提示设计与覆盖策略消融实验

## K.1 提示设计说明

不同于直接输入原始文本的标准长上下文评估，我们将所有输入块包装在模拟的 User-Assistant 对话中，以显式触发智能体的记忆机制。每个输入块前均置入一条记忆指令（如“请记住以下信息，以供后续提问”），确立明确的信息存储意图。对于每个具体数据集，我们都仔细设计任务指令，确保智能体准确理解任务意图并执行所需操作。

尤其重要的是，对于选择性遗忘能力，我们在提示中加入显式护栏。我们明确告知智能体：事实按序号索引，越新的事实序号越大。智能体必须优先采用最新事实来解决冲突（完整提示模板见补充代码仓库）。

需要说明，尽管提示差异会改变结果，我们的基准对所有被评估的智能体（长上下文、RAG 与智能体式智能体）应用统一、标准化的提示模板。这确保所观察到的性能差距（例如 RAG 智能体在多跳选择性遗忘上的失败）应归因于其记忆机制的局限，而非提示不一致。

## K.2 覆盖策略消融实验

为严格检验显式指令能否缓解遗忘/覆盖问题，我们以 GPT-4.1-mini 为基线，使用显式覆盖提示开展了额外消融研究。测试两种策略设定：

- **策略 A（始终优先采用较晚信息）：** “关键规则：把事实视为按时间顺序排列的更新流。如果事实之间存在任何冲突，你必须始终用序号较大的事实覆盖较早的事实。”
- **策略 B（保守/显式否定）：** “关键规则：谨慎更新。只有当序号较大的事实明确否定较早事实，或明确声明先前信息不正确时，才丢弃或覆盖较早事实。”

### K.2.1 消融结果

覆盖策略消融结果见表 19。

**表 19：选择性遗忘任务上的覆盖策略消融结果（Accuracy %）。**

| 模型/设定 | FC-SH | FC-MH | Avg. |
|---|---:|---:|---:|
| GPT-4.1-mini (Baseline) | 36.0 | 5.0 | 20.5 |
| GPT-4.1-mini (Policy A) | 40.0 | 4.0 | 22.0 (+1.5) |
| GPT-4.1-mini (Policy B) | 28.0 | 4.0 | 16.0 |

### K.2.2 关键洞见

1. **激进更新的泛化能力有限：** 策略 A 虽然小幅提升了单跳任务性能（FC-SH 从 36.0 提升至 40.0），却无法泛化到复杂的多跳推理（FC-MH 降至 4.0）。这表明，提示虽然有助于简单事实检索，却无法让更新有效传播过多步推理链。

2. **保守约束导致性能下降：** 策略 B 使平均性能显著下降（-4.5 分）。复杂的条件指令（检查显式否定）增加了模型的认知负荷，并诱发过度谨慎的行为，导致有效更新无法被检索。

这些发现构成一项健全性检查，验证了我们的核心动机：选择性遗忘无法仅靠提示工程解决，而需要专门设计记忆机制。

# L 关于 LLM-as-a-Judge 与输入格式的补充说明

## L.1 LLM-as-a-Judge 评估的有效性

我们承认，基于模型的评估未必适合所有任务情境。不过，为与既有工作保持一致，我们在 LongMemEval 和 ∞Bench-Sum 数据集上采用 LLM-as-a-judge 评分，并从以下方面验证其适用性：

- 对 LongMemEval 而言，问题具有明确、客观的标准答案，主观性极小。Wu et al. (2025) 报告，经过提示工程的 GPT-4o 评判器与人工标注的一致率达到 98.0%，稳定性非常高。
- Yen et al. (2024) 验证，在长上下文摘要任务上，GPT-4o 的判断大体与人工评估一致。

这些发现证实，在本基准所含任务上，我们的 LLM-as-a-judge 设置能够可靠反映人工评估。

## L.2 分块输入格式的依据

我们承认，现实世界的个人助理可能接收流式输入（例如连续的用户交互或实时数据流）。然而在实践中，必须先将现实世界输入量化，再送入语言模型（例如将连续输入累积为用于推理的离散块）。因此，在现实部署中，向智能体输入分块是处理流式输入的一种自然且现实的策略。

此外，分块输入格式模拟了现实世界用户—智能体交互的增量、多轮性质：信息随时间依次到达，而不是作为单个完整文档一次性提供。这一设定对于评估记忆智能体至关重要，因为它们被设计为增量处理信息，而不是处理标准长上下文基准所采用的静态全文输入。


