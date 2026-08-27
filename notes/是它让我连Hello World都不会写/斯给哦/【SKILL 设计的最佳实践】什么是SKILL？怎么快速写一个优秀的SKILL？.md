# 【SKILL 设计的最佳实践】什么是SKILL？怎么快速写一个优秀的SKILL？ 

[原帖链接](https://linux.do/t/topic/1424073/1)

**作者：Nick Hugo**  
**时间：Jan 9, 2026 2:40 am**  

最近需要写一个Skill用于拓展之前grok-search的相关功能，希望能做到写代码的时候进行事实核查以及代码demo，技术路线，API版本校验的功能  

[【多模型MCP协同新实践】突破Claude Code 官方 Web_Search 的安全内容审查 — 检索增强的 Grok MCP ( 补全 Vibe Coding 的最后一块拼图 - 开发调优 - LINUX DO](https://linux.do/t/topic/1356321)  

在学习怎么写Skill的过程中发现，如果想要发挥Skill的作用，撰写一个好的SKILL.md非常重要。本来想找一下站里有没有相关信息，不过好像没有看到，看到好像之前也有佬也有问题  

[claude的skills，有没有佬研究过，求个最佳实践的教程 - 开发调优 - LINUX DO](https://linux.do/t/topic/1058513)

所以专门研究了一下官方文档，然后加上自己的理解诞生了这个帖子，下面是大纲，核心原则是根据官方文档+自己的理解补充的。

- [Too Long Don’t Read 省流版:](#too-long-dont-read-%E7%9C%81%E6%B5%81%E7%89%88)
- [What’s SKILL](#whats-skill)
- [SKILL设计的核心原则](#skill%E8%AE%BE%E8%AE%A1%E7%9A%84%E6%A0%B8%E5%BF%83%E5%8E%9F%E5%88%99)   - [一、 命名是起点 （Naming is First Step）]() - [二、 描述即路标 （Description Is the Signal）]() - [三、 简洁是关键（Concise is Key）]() - [四、 自由度要匹配（Match the Degrees of Freedom）]()
- [如何生成卡通图的补充]()

## Too Long Don’t Read 省流版:

为了节省佬们的时间，主要内容以一图流的形式方便佬们理解跟收藏，感谢伟大的谷歌大善人，Gemini还是太好用了 嘻嘻！

![summary](https://cdn3.ldstatic.com/original/4X/8/1/6/816e8f779384fe8b841b9dcb991c4a16a235a288.jpeg)
## What’s SKILL

目前大模型瓶颈主要卡在上下文长度上，加载过多mcp的情况下，过长的系统提示词，MCP tools的功能描述会使得模型需要**花更长的时间来分析工具的调用**，还会**弱化模型的指令遵循能力**。

为此 Claude 提出了**Skill**，SKILL设计的出发点就在于使用**渐进式披露**的方式，使用尽可能少的上下文实现功能的感知，真正实现有效加载上下文！

![compare](https://cdn3.ldstatic.com/original/4X/e/4/e/e4e358fbd4e01b1d76d353f5830c3fae72a3e055.jpeg)
因此一个优秀的SKILL.md应当同时具备:
> - **较短的概览实现SKILL的调用**
> - **明确的细节解释**来规划工作流
> - 补充**合适的技术细节完成最后的具体实现**


而这些就诞生了 SKILL的最佳实践 Best Practice, 其实Claude官网对于Skill.md的创作提供了一个最佳实践 [Skill authoring best practices - Claude Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

本文呢是结合个人经验对官方文档的二次总结&提炼，如果想了解更详细的信息，还是推荐佬们看第一手官方文档信息

## SKILL设计的核心原则

SKILL文档主要分为4种组件:
> - 元信息（YAML元数据）： 包括name以及decription唯一标识符，启动时默认加载，需要保证简洁避免冗余
> - 主体内容（Body）： 相当于目录，提供概述、入门指导和资源引用，按需加载所需资源，减少上下文注入
> - 参考文件(Reference File)：详细指南 API参考 以及使用示例，当主体明确链接到参考文件时实现动态加载
> - 脚本文件（Scription）： 提供可执行脚本用于获取输入输出，保证一致性执行


一图流：

![processing](https://cdn3.ldstatic.com/original/4X/6/0/9/60924b70282921edf6b8891c639f78b9e0c248f2.jpeg)
### 一、 命名是起点 （Naming is First Step）

启动SKILL时只会预加载所有SKILL的**名称（name）**和**描述（description）**，因此设计合适精炼的名称以及描述时成功调用SKILL的第一步。

SKILL的命名需要清晰地描述技能所提供的活动或能力，推荐使用**动词ing+对象**的命名习惯，避免模糊或者通用描述，导致模型紊乱
> - 正例：processing-pdfs； analyzing-spreadsheets； managing-databases
> - 反例：helper； documents； utils


### 二、. 描述即路标 （Description Is the Signal）

**Description** 用于查找技能，应包括技能的功能和使用时机。

务必具体，并包含关键术语 。不仅要**说明技能的功能**，还要说明**何时使用该技能的具体触发条件/ 上下文** 。

由于大多数情况下 Claude 根据description来实现工具的调用，因此SKILL的description必须做到有区分，使得Claude知道何时选择该技能。避免模糊表述或者冗余

正例：

```

description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.s

```

反例（模糊表述）：

```

description: Process pdf files.

```

反例（冗余表述）：

```

description: This skill is designed to help users work with PDF documents by providing comprehensive functionality for extracting text content and tabular data from PDF files, as well as filling out form fields in PDF documents, and merging multiple PDF documents together into a single file. It can be used whenever you need to process PDF files, or when users explicitly mention that they want to work with PDFs, or when they talk about forms, or when document extraction is needed, or in any situation where PDF manipulation might be helpful or beneficial to the task at hand.

```

### 三、 简洁是关键（Concise is Key）

现在我们正式进入主体内容（body）的撰写，这一部分将以目录的方式组织SKILL更具体的参考资源，执行逻辑以及上下文加载,由于主体部分在加载后会直接参与Claude的上下文中，与其他内容存在竞争关系，因此在主体部分中 – “简洁是关键”，这里Claude 官方提出一个基本假设：

**“Claude已经足够聪明，不需要补充常识，只需要补充Claude不了解的背景信息。”**

正例：

```

## Extract PDF text

Use pdfplumber for text extraction:

```python

import pdfplumber

with pdfplumber.open("file.pdf") as pdf:

text = pdf.pages[0].extract_text()

```

```

反例：

```makdown

## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains

text, images, and other content. To extract text from a PDF, you'll need to

use a library. There are many libraries available for PDF processing, but we

recommend pdfplumber because it's easy to use and handles most cases well.

First, you'll need to install it using pip. Then you can use the code below...

```

**特别的，官方建议正文部分不要超过500行**

### 四、 自由度要匹配（Match the Degrees of Freedom）

由于Claude本身具有很高的自由度，有的时候这种自由度可以极大的提升模型的创造能力，但对于部分特殊任务，我们希望通过严格的流程或者具有唯一确定的解决办法，又或者是我们需要满足特定约束，因此我们需要在SKILL中对于不同操作赋予不同自由度，根据自由度的区别主要可以使用下面的方式来进行规范以及约束：

| 自由度等级 | 任务特征 | 指令设计建议 | 典型示例 |
| --- | --- | --- | --- |
| High | - 多种方法都可能成功
- 决策高度依赖具体上下文
- 可以通过启发式而非硬规则指导 | - 给出总体目标和方向性指导
- 使用原则、检查清单或启发式约束
- 信任模型根据上下文选择最佳路径 | 代码审查（最佳方式取决于上下文） |
| Mid | - 存在推荐的执行模式
- 允许一定范围内的变体
- 行为会受到配置或参数影响 | - 固定整体结构或流程
- 通过参数或配置开放可变部分
- 平衡可控性与灵活性 | 使用参数控制行为的自动化脚本 |
| Low | - 只有一种安全或正确的执行路径
- 错误代价高，容错空间极小
- 执行顺序或步骤错误会造成严重后果 | - 提供明确的护栏和强约束
- 给出精确步骤和固定执行顺序
- 避免让模型自行推断关键决策 | 必须按顺序执行的数据库迁移 |

下面给出官方文档中 对于三种不同自由度的示例

### 高自由度（使用文本描述）：

```

## Code review process

1. Analyze the code structure and organization

2. Check for potential bugs or edge cases

3. Suggest improvements for readability and maintainability

4. Verify adherence to project conventions

```

### 中自由度(使用伪代码或带参数的脚本)

```

## Generate report

Use this template and customize as needed:

```python

def generate_report(data, format="markdown", include_charts=True):

# Process data

# Generate output in specified format

# Optionally include visualizations
```

### 低自由度(特定脚本+少参数+文本限制)

```

## Database migration

Run exactly this script:

scripts/migrate.py --verify --backup

Do not modify the command or add additional flags.
```

其实目前比较好的工作流是使用claude自己生成一份初始版的SKILL.md然后自己根据上面的原则进行优化，因为实际使用的过程中发现好像Claude有的时候自己也不完全遵守这些最佳实践，感觉需要一个专门写SKILL的SKILL（递归了属于是），后面有空看看能不能整个。
> 感谢 [@JoJoJotarou](https://linux.do/u/jojojotarou)  [@eguhz99](https://linux.do/u/eguhz99) 佬的补充，官方提供了一个用于构造SKILL的SKILL，而且好像在使用claude构建SKILL的时候也会自动调用该工具，链接如下：  
> [skills/skills/skill-creator at main · anthropics/skills · GitHub](https://github.com/anthropics/skills/tree/main/skills/skill-creator)  
> 不过如果了解，本文可以帮助佬们在更好的实现自己的SKILL的时候能起到一点点小小的帮助，那在下还是很开心的，嘻嘻~


关于SKILL针对具体场景还有其他的优化模式，这在官方文档中也有说明，不过这个帖子就先到这吧，后面有时间再搬运以及总结。

快周末了，就祝佬们周末愉快~

## 关于生图的补充

> 好像好多佬想知道图怎么生成的，想要提示词，就在这统一回复了，笔者使用的是中转站的Gemini banana pro + blog内容 + 风格化图 作为输入生成的风格的关键词是“手账”，具体的操作以及页面可以参考评论 59楼以及64楼，理论上只要是使用gemini应该都可以生成类似的，我也不介意佬们直接用我的图作为仿图，顺便一提里面的卡通人物是 疯狂动物城的狐尼克 也是笔者的头像，算是笔者的一点小彩蛋吧 哈哈哈
