# 最近在站里看见了很多吐槽codex的帖子，我分享一些自己的使用经验给各位参考 

[原帖链接](https://linux.do/t/topic/1427711/3)

**作者：heidi**  
**时间：Jan 10, 2026 12:05 pm**  

我是从codex cli出来就开始使用的，现在工作上完全使用codex。我觉得codex并没有说的这么差而且只要用好基本能解决所有问题。  

下面是一些基本情况大家了解一下  

1、机器是macboompro m1 max  

2、使用语言为java开发都是web相关的项目(最近在学rust)  

3、从今年a÷出过公开敌对中国事件后我就没使用claude code了，账号我也发邮件让他们删除了  

4、codex我只使用cli，插件没用过，一般都是在vs中删除多余的会话  

看过贴吧很多帖子与问题示例后觉得很多人使用不好的原因是对模型缺乏一个基本的了解，所有我先给大家介绍一些模型的基本知识  

一、模型的数据是哪里来的
> 公开可获取的互联网文本、技术博客、论坛、文档网站、百科内容、说明文档、博客文章、开源代码(github)、额外授权的书籍、文档、人工清晰构造的高质量样本(比如在强化学习和标注数据阶段模型公司就会出一些问题然后邀请专家来为这个问题编写高质量的回答


二、token是什么
> 模型内部用于计算的最小文本单元，token 的切分由 tokenizer 决定，与自然语言的词法规则不完全一致(大家可以通过这个网站去体验一下[https://tiktokenizer.f2api.com](https://tiktokenizer.f2api.com))


三、一个模型产生的大致阶段
> 1、预训练阶段  
> 这一阶段是训练成本最高的，大模型训练的费用大部分都是花费在这个上面  
> 预训练使用的是大量弱结构数据，包括自然语言与代码混合语料。数据不以任务、指令、问答为单位而是以连续文本流的形式输入。模型看到的只是token序列，这一阶段模型主要训练结构建模能力(语言语法、代码语法、嵌套结构、长程依赖关系)、 统计语义关联(哪些概念常一起出现，哪些模式在特定上下文中更高概率成立) 、跨域泛化能力(不同语言、不同编程语言、不同写作风格之间的共享模式)。预训练模型不知道自己在回答问题不具备帮助用户解决问题不区分真实与虚假，只区分哪些token常见与罕见  
> 2、标注训练  
> 使用人工编写的高质量样本对模型进行监督训练，这些样本通常以指令、问题、代码需求等形式出现，并配有明确的理想回答。具体可参考openai的这篇论文[https://arxiv.org/pdf/2203.02155](https://arxiv.org/pdf/2203.02155) (3.4章节)  
> 这一阶段的核心作用是约束模型的输出行为，让模型从单出的对互联网数据回忆续写文本转变为尝试按人类指令完成任务。标注训练本身并不显著增加新知识，而是让模型用已有知识解决用户的具体问题。  
> 3、强化学习  
> 在标注训练基础上，通过让人类对多个模型回答进行排序，训练奖励模型，再用强化学习算法调整主模型的输出概率分布。强化学习降低胡编概率、提升回答一致性，并强化安全边界与拒答行为用于调整输出倾向


上面这些内容了解一下就行，我自己也是懂了个皮毛所以里面肯定有很多错误。具体信息可以去看Andrej Karpathy的视频

![](https://cdn3.ldstatic.com/original/4X/5/5/2/5521bd1bee2dc0f18f7c27bec15b711b2e8e979f.jpeg)
      
    
    
      
        [Deep Dive into LLMs like ChatGPT](https://www.youtube.com/watch?v=7xTGNNLPyMI)

b站的飞天闪客也可稍微看一下  

了解了这些后我们可以简单将大模型的一个会话抽象理解成一个持续输出的一维的token数组，你在上下文的输入会影响这次会话中模型的输出而且这个影响会发生的很快，当你发现模型的输出开始出现问题或者风格不是需要的最好检查一下你输入了什么，当然你也可以在发现问题时直接纠正，将他改造成你喜欢的样子。推荐在工作的工程中尽量减少语气化的输入(你认为我在跟你嘻嘻哈哈么？我看起来现在是想跟你搞七捻三么)  

毁灭吧我累了，猜猜我按下这个按钮会发生什么 ![image](https://cdn3.ldstatic.com/original/4X/e/8/2/e82509826a24a529e47b75e729dd163b1b847531.png)
了解了这些你大概就明白为什么你的模型总是不办事或者办事办的乱七八糟的，当然和模型本身的能力也是有关系的  

接线来给大家介绍一下codex cli
> 这是一个code agent你可以在本地或者ide插件中去使用，它能够使用内置工具帮助你从0开始完成一些编码任务  
> 工程结构  
> **模型层**  
> 负责理解指令、分析上下文、生成计划与代码，本质是一个经过对齐的代码模型。  
> **执行层**  
> 负责把模型输出转化为可执行动作，例如：读文件、改代码、运行命令、调用工具，并记录完整执行轨迹。  
> **环境层**  
> 即你当前的本地仓库或云环境，包括文件系统、git 状态、依赖环境、网络权限等。  
> Codex 启动时会构建一个指令链，不是简单拼一段 prompt。这个指令链在一次会话（CLI session / exec run）开始时构建完成，之后整个执行过程都基于它运行。  
> AGENTS.md 是配置规则的核心入口，Codex 会在执行任何任务前，自动搜索并加载 AGENTS.md 文件。这些文件不需要你在 prompt 中显式提到，只要存在，就会被自动纳入上下文窗口。


1. **全局规则（Global）**

- 目录：`~/.codex/`
- 优先级： - `AGENTS.override.md` - `AGENTS.md`
- 只读取第一个非空文件

1. **项目规则（Project）**

- 从项目根目录开始
- 一直向下走到当前工作目录
- 每一层目录最多加载一个文件，顺序为： - `AGENTS.override.md` - `AGENTS.md` - 备用文件名（如 TEAM_GUIDE.md）

1. **合并规则**

- 所有命中的文件会按目录顺序拼接
- **越靠近当前目录的规则优先级越高**
- 后出现的规则可以覆盖前面的约定

1. **窗口大小与rule截断**

- 所有规则文件会被拼接进模型上下文
- 有最大字节限制（默认 32KB）
- 超过上限后，后续规则不会被继续加载

以上是它默认的执行链，可以自己去配置加载的路径。官方文档中有说我就不详细说了

  
      ![](https://cdn3.ldstatic.com/original/4X/4/e/6/4e6c215b527d954515a2695f3754a6bce3aa56f7.png)

      [developers.openai.com](https://developers.openai.com/codex/guides/agents-md)
  

  
    ![](https://cdn3.ldstatic.com/optimized/4X/0/d/8/0d8af75cd9ff650778ba8b0e03c3542979f0bdcc_2_690x259.jpeg)

### Custom instructions with AGENTS.md

  Give Codex extra instructions and context for your project

  

  
    
    
  

  

接下来讲讲MCP
> MCP 是 模型 用来接入外部工具和上下文的统一协议。  
> 它解决的问题是：模型如何在不内嵌能力的前提下，安全、可控地使用外部系统。  
> 我一般将它理解为微服务架构中的一个微服务  
> 我只装了这三个mcp，  
> **context7**  
> 用来查开源项目的最新文档，模型工作时优先使用的是训练数据有些数据和技术可能太老。  
> [GitHub - upstash/context7: Context7 MCP Server -- Up-to-date code documentation for LLMs and AI code editors](https://github.com/upstash/context7)  
> **drawio**  
> 用来画图，效果不错，只尝试过一次。  
> [GitHub - lgazo/drawio-mcp-server: Draw.io Model Context Protocol (MCP) Server](https://github.com/lgazo/drawio-mcp-server)  
> **database**  
> 模型在工作时最好能了解你的数据模型，使用这个就可以让他在实现需求时结合数据库中的数据与表结构思考减少因为信息不完整时的错误实现，这个mcp只包含查询功能  
> [MCP - 数据库查询MCP](https://modelscope.cn/mcp/servers/socialman/database-mcp)
> ![image](https://cdn3.ldstatic.com/original/4X/4/f/7/4f7741096dc0ff799ee74ed80ec9465c700f130d.png)


接下来说说skill
> 这个东西在我的印象中好像出了很久了，但是站里还是有相当一部分再问，我觉得有点困惑 。  
> skill本质就是将原本需要在agents.md中编写的一些规则抽象成一个单独的文档让模型在执行任务时可以自己根据description判断是否需要读取这些内容。它解决的是agents.md中内容爆炸的问题，agents.md中的内容是第一次启动时才会构建，这样随着上下文的延长他就会离当前的上下文距离越远。skill可以随时重新让模型读取，离当前上下文越近模型执行执行力就越高(当时使用atlas看官方文档时的那个会话一直嘴硬跟我说使用必须手动用命令告诉模型使用`$skill-name`,我后面的测试中是不用的。模型会自己判断加载，当然如果你发现它不遵守也可手动在输入中告诉他遵守这个skill。所以skill的描述不要太接近，相同的内容放在同一个skill中就行了，输入命令如果前方有你的输入记得加空格，命令如果是在最前面就不用)
> ![image](https://cdn3.ldstatic.com/original/4X/8/5/8/858d014c4f6598411c1c41e3fafd08da6b2c4ac2.png)  
> ![image](https://cdn3.ldstatic.com/original/4X/0/7/d/07de8a28997844d72bbd982ac0a626f8ffc4c89d.png)


然后讲讲cli中一些命令
> 我使用过的只用init、review、resume 。其他的好像用不到啊  
> **init**  
> 就是用来初始化你的项目生成agents.md的  
> 这个东西我一般把他放在项目根目录初始化时直接告诉告诉他根据 agent-md-example/agent-init.md 中的规则初始化项目
> ![image](https://cdn3.ldstatic.com/original/4X/4/2/5/425eb8d9da0623133e993e126df3dae8487e3d8d.png)  
> [v2-2025-12-19.zip](https://linux.do/uploads/short-url/u8V3VfIn7AMeCRM0IYUfftLIupE.zip) (25.9 KB)  
> 这个google-java-style-guide.md我是在网上找的然后整理成md的，是不是google的我也不确定，随便啦。  
> [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)  
> **resume**  
> 用来恢复会话的  
> **review**  
> 可以让来对比提交、分支、未提交的代码对比检验模型的实现是否正确的  
> ![image](https://cdn3.ldstatic.com/original/4X/4/c/f/4cf0ae72cc6738194f60c3e799539f9bdb5fd6e5.png)  
> 一般都是用第4个自定义指令，让他明确的只校验当前功能相关的代码也减少一些token消耗，现在review也有额度了，元旦过后就加了。说实话其余的我真没用到过，大家如果需要可以去它官方文档中自行查询  
> [Codex](https://developers.openai.com/codex)


接下来讲讲一些chatgpt的使用经验吧，说实话现在不用gemini的部分原因就是他的其他产品特别好用，比如会话记忆、自定义风格、自定义gpt。5.2更新后我感觉gpt风格变得越来越油腻和谄媚了，动不动就罗里吧嗦的说一堆废话，还特别喜欢一句话总结。我总结你xxxx **自定义gpt**

![image](https://cdn3.ldstatic.com/original/4X/c/d/8/cd8dda12dc64f4a54118ae8215e3132eec30228f.png)  

这个功能创建和修改目前只支持web，app端可以使用创建好的但是不能修改或创建，他的作用就相当于你自定义一个带系统提示词的会话，这样就不用每次开新会话想让他做一些固定事情是还要跟它解释半天了，你把他理解成一个自定义skill就行比如翻译  

![image](https://cdn3.ldstatic.com/original/4X/8/a/4/8a4b8a464d71e700b5a8a25205b944e49ddf7514.png)  

这是我自定义的一个用来专门翻译的，以后我用这个gpt创建一个会话时直接将英文文档给他就行了  

![image](https://cdn3.ldstatic.com/original/4X/b/5/4/b5411b99e80e23fc8d7a73ebfae065790327d170.png)  

**自定义风格**  

这个我就把我自己的个性化定义提示词给大家参考一下  

![image](https://cdn3.ldstatic.com/original/4X/c/e/5/ce584107615ced1ec566e062eff0813fab56bfb6.png)> Language & Tone  
> Default language: Chinese  
> Use technical, engineering-oriented language  
> State facts and conclusions only  
> No rhetorical or evaluative language
> Default Output Rules (Strict)  
> Answer directly  
> No preamble, no small talk, no evaluation, no opening remarks  
> Do not restate the question  
> Do not use phrases like “this is a good question”, “let’s first”, “in conclusion”, etc.  
> No explanations by default  
> Maximum 5 lines unless explicitly requested
> Explanation Rules  
> Explanations are allowed only if the prompt explicitly includes keywords:  
> “why”, “explain”, “reason”, “principle”, “details”, “expand”  
> If none of these keywords appear, treat the request as answer-only
> Content Constraints  
> No tutorial-style writing  
> No background, history, or conceptual introductions  
> Include only information strictly required to answer the question  
> Prefer precise, verifiable technical statements
> Optional Trigger Keywords  
> “answer only”: output conclusion only  
> “engineering perspective”: allow implementation details and trade-offs  
> “expand”: allow detailed explanation
> Failure Condition  
> Any preamble, evaluative language, or unnecessary explanation counts as a failed response


**知识库**  

有些论文或者书籍你可以在这里新建一个知识库，然后让模型将内容总结给你，提升信息的接收速度。这里一个知识库后面是一个沙箱环境的linux主机，你所有上传的文件都会保存在这个沙箱环境中

![image](https://cdn3.ldstatic.com/original/4X/d/8/d/d8d61e1d5c92311b10eedb6426de2faa7149032b.png)  

![image](https://cdn3.ldstatic.com/original/4X/f/2/6/f26e3ff1c48e85b9aa7d591da0d2e1bccd9ad067.jpeg)
兄弟们燃尽了，一滴都没有了。其中说的有些不对的地方麻烦大家给我指正一下。写了几个小时整理这个文档，很多地方怕写不对写了又删好累啊。  

这个内容太长了SKILL的分享我就放评论区，都只有SKILL.md没有其他的比如

- scripts
- references
- assets    因为都不是工具类的skill，然后给大家分享一个获取skill的网站    [https://skillsmp.com](https://skillsmp.com)    我觉得在ai中自己写是最符合自己需求的，需要什么就创建什么    ![image](https://cdn3.ldstatic.com/original/4X/4/4/7/447ce15db00bb31b9a98cfd991a09124c9bf2bcc.png)    目前我感觉我的配置codex会存在几个问题，有时他不会遵守先计划再编码的规则直接就开始编码的。还有就是代码的结构有时会有点过度封装

然后就是我基于我目前的skill与配置给大家演示一下效果。从0开始简单的copy一个开源项目

  

      [github.com](https://github.com/springwolf/springwolf-core?tab=readme-ov-file#about)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/1/8/e/18e7457eeae3435fd95101bb878f60a6e308e217_2_690x344.png)

  ### GitHub - springwolf/springwolf-core: Automated documentation for event-driven...

    Automated documentation for event-driven applications built with Spring Boot

  

  
    
    
  

  

看看token消耗  

codex简单复现项目地址，后端完成时间18分钟，前端完成时间为10分钟。前端我没有任何skill或者agents.md规则效果可能会差一点

  
      ![](https://cdn3.ldstatic.com/original/4X/b/a/d/bad3e5f9ad67c1ddf145107ce7032ac1d7b22563.svg)

      [github.com](https://github.com/littleheid/spring-wolf-demo/tree/main)
  

  
    ### GitHub - littleheid/spring-wolf-demo

  [main](https://github.com/littleheid/spring-wolf-demo/tree/main)

  Contribute to littleheid/spring-wolf-demo development by creating an account on GitHub.

  

  
    
    
  

  

token消耗前后对比

![codex-limit](https://cdn3.ldstatic.com/original/4X/c/5/0/c5077bd394bb764482708c653fa7e6f80df4d2a1.png)  

![codex-limit2](https://cdn3.ldstatic.com/original/4X/7/0/b/70b18e6b06ee0e968dea408a4072edd6c859642b.png)
吐槽一下，codex感觉元旦后token消耗变高了，我一周只用5天都快感觉不够用了。以前还能剩下百分之50，现在都只能剩下25了。还有上下压缩的问题，有时看剩余明明有30，它就开始压缩了。不过压缩效果很好，目前看来没有丢失过信息，就是压缩会很慢。

## 收藏楼层（#3）

**作者：heidi**  
**时间：Jan 10, 2026 12:15 pm**  
**原帖楼层：[查看 #3](https://linux.do/t/topic/1427711/3)**  

接下来分享一些skill，都是java和项目结构相关的，关于agents.md中的指令与skill最好还是使用英文编写，因为模型训练时数据使用英文对于英文的规则遵守肯定更高。虽然gpt中文语言能力还行
**ask-questions-if-underspecified**
这个是codex负责人Tibo分享的，当模型犹豫不决时将问题抛给用户 :smiley: 
```md
---
name: ask-questions-if-underspecified
description: Clarify requirements before implementing. Do not use automatically, only when invoked explicitly.
---

# Ask Questions If Underspecified

## Goal

Ask the minimum set of clarifying questions needed to avoid wrong work; do not start implementing until the must-have questions are answered (or the user explicitly approves proceeding with stated assumptions).

## Workflow

### 1) Decide whether the request is underspecified

Treat a request as underspecified if after exploring how to perform the work, some or all of the following are not clear:
- Define the objective (what should change vs stay the same)
- Define "done" (acceptance criteria, examples, edge cases)
- Define scope (which files/components/users are in/out)
- Define constraints (compatibility, performance, style, deps, time)
- Identify environment (language/runtime versions, OS, build/test runner)
- Clarify safety/reversibility (data migration, rollout/rollback, risk)

If multiple plausible interpretations exist, assume it is underspecified.

### 2) Ask must-have questions first (keep it small)

Ask 1-5 questions in the first pass. Prefer questions that eliminate whole branches of work.

Make questions easy to answer:
- Optimize for scannability (short, numbered questions; avoid paragraphs)
- Offer multiple-choice options when possible
- Suggest reasonable defaults when appropriate (mark them clearly as the default/recommended choice; bold the recommended choice in the list, or if you present options in a code block, put a bold "Recommended" line immediately above the block and also tag defaults inside the block)
- Include a fast-path response (e.g., reply `defaults` to accept all recommended/default choices)
- Include a low-friction "not sure" option when helpful (e.g., "Not sure - use default")
- Separate "Need to know" from "Nice to know" if that reduces friction
- Structure options so the user can respond with compact decisions (e.g., `1b 2a 3c`); restate the chosen options in plain language to confirm

### 3) Pause before acting

Until must-have answers arrive:
- Do not run commands, edit files, or produce a detailed plan that depends on unknowns
- Do perform a clearly labeled, low-risk discovery step only if it does not commit you to a direction (e.g., inspect repo structure, read relevant config files)

If the user explicitly asks you to proceed without answers:
- State your assumptions as a short numbered list
- Ask for confirmation; proceed only after they confirm or correct them

### 4) Confirm interpretation, then proceed

Once you have answers, restate the requirements in 1-3 sentences (including key constraints and what success looks like), then start work.

## Question templates

- "Before I start, I need: (1) ..., (2) ..., (3) .... If you don't care about (2), I will assume ...."
- "Which of these should it be? A) ... B) ... C) ... (pick one)"
- "What would you consider 'done'? For example: ..."
- "Any constraints I must follow (versions, performance, style, deps)? If none, I will target the existing project defaults."
- Use numbered questions with lettered options and a clear reply format

1) Scope?
a) Minimal change (default)
b) Refactor while touching the area
c) Not sure - use default
2) Compatibility target?
a) Current project defaults (default)
b) Also support older versions: <specify>
c) Not sure - use default

Reply with: defaults (or 1a 2a)


## Anti-patterns

- Don't ask questions you can answer with a quick, low-risk discovery read (e.g., configs, existing patterns, docs).
- Don't ask open-ended questions if a tight multiple-choice or yes/no would eliminate ambiguity faster.
```

**java-coding-style**
```md
---
name: java-coding-style
description: Enforce repository-wide Java conventions for formatting, naming, error handling, logging, nullability, and documentation. Spring Web layering and API rules are defined in spring-web-api-architecture-protocol.
---

# Java Coding Style (SOP)

## Scope (MUST)

Use this skill when implementing or reviewing Java code in this repository.

Out of scope:
- Spring Web external API layering / DTO boundaries / validation / unified error envelope (use `spring-web-api-architecture-protocol`)
- Over-architecture and structural minimalism rules (use `java-code-structure-minimalism`)

## Formatting (MUST)

- Code must be auto-formatable and consistently formatted.
- Follow Google Java Style.
- Use `google-java-format` when possible; do not rely on manual alignment.

## Naming Semantics (MUST)

- Package/class/field/method names must reflect **business semantics**.
- Forbidden naming patterns:
  - Generic or ambiguous names: `tmp`, `data`, `info`, `process`, `handle`, `doThing`, `manager`, `helper`, `util` (unless the name is the established domain term)
  - Technical-implementation names at business boundaries: `Impl`, `Base`, `Common`, `Misc` (unless required by framework conventions)
- Names must be stable under refactors: avoid encoding transient implementation details.

## Nullability (MUST)

- Keep nullability explicit where relevant.
- Prefer `Optional` only for return values where absence is a valid state; do not use `Optional` for fields.

## Library & Utility Usage (MUST)

- Do NOT create new utility methods or classes when equivalent functionality exists in:
  - JDK
  - Spring Framework
  - Apache Commons Lang3 / Collections
- Thin wrappers around existing library calls are forbidden.
- Generic utility classes (`*Util`, `*Helper`, `*Common`) are forbidden.

Exception:
- A helper is allowed only if it encodes **business semantics** (not generic logic) and is reused.

## Error Handling and Validation (MUST)

- Never swallow exceptions.
- Preserve the original cause when rethrowing.
- Classify errors explicitly:
  - Domain errors: represent business rule violations
  - Infrastructure errors: IO/DB/remote calls/timeouts
- Do not use exception messages as program logic.
- Enforce invariants at module boundaries.

## Logging and Observability (MUST)

- Do not log secrets, credentials, tokens, or PII.
- Logs must carry a correlation identifier using a consistent MDC key, e.g. `traceId`.
- Log levels:
  - `ERROR`: unexpected failures requiring attention
  - `WARN`: recoverable anomalies or degraded behavior
  - `INFO`: business-relevant state transitions (avoid hot paths)
  - `DEBUG/TRACE`: local debugging only
- Avoid noisy logs in hot paths; throttle logs in loops/retries.

## Comments and Javadoc (MUST)

- Documentation language defaults to **Simplified Chinese** unless explicitly required otherwise.
- Write concise Javadoc for public classes and public methods when the behavior is not obvious.
- Use inline comments only for non-obvious branches or invariants.
- Relax Javadoc for simple test-only code.

### Class-level Javadoc Header (MUST when adding a new public class)

Include:
- `@author heidi`
- `@date yyyy/MM/dd`
- `@describe <简短的描述>`
- `@since 1.0`

## Review Checklist (MUST)

- [ ] Code is formatted by `google-java-format` (or equivalent automated formatter)
- [ ] Names are business-semantic and unambiguous
- [ ] Nullability is explicit where it matters
- [ ] Exceptions are not swallowed; causes preserved
- [ ] Logs exclude secrets/PII and include `traceId`
- [ ] Comments/Javadoc are Simplified Chinese by default
```
**large-scope-design-documentation-protocol**
这个是写代码时怎么写计划文档

```md
---
name: large-scope-design-documentation-protocol
description: Defines mandatory documentation structure, location rules, and update constraints for large-scope or long-running design changes, ensuring recoverable context for humans and agents.
---

## Directory Structure (MUST)

docs/<feature>/
├── plan.md       # Feature background and implementation plan (entry point)
├── model.md      # API / data model / schema / contract definitions
├── task.md       # Dynamic task tracking and current execution state
├── changelog.md  # Time-ordered implementation record

## `plan.md` — Design Context and Plan (MUST)

Purpose:

- Provide a single entry point for humans or agents
- Explain why the feature exists and what is being changed

Must include:

- Problem statement and motivation
- Scope and non-goals
- Design boundaries and constraints
- High-level implementation plan

Must NOT include:

- API field-level details
- Step-by-step implementation logs
- Task status updates

---

## `model.md` — API and Data Models (MUST if applicable)

Purpose:

- Serve as the single source of truth for external or internal contracts

Must include:

- API endpoints or messages
- Request/response schemas
- Data models and relationships
- Validation rules and invariants

Must NOT include:

- Business logic explanations
- Implementation steps
- Task or progress information

---

## `task.md` — Task Tracking (MUST)

Purpose:

- Track current execution state during long-running or interrupted work
- Enable fast recovery after context loss

Must include:

- Current phase or milestone
- Completed items
- In-progress items
- Blockers and open risks
- Next actionable steps

Rules:

- This is a dynamic document and may be frequently overwritten
- Only the current state matters

Must NOT include:

- Historical logs
- Design rationale
- Detailed code diffs

---

## `changelog.md` — Implementation Record (MUST)

Purpose:

- Provide a compact, append-only history of what changed over time

Must include (per entry):

- Timestamp or version
- Files or modules modified
- Short description of behavior added or changed

Rules:

- Append-only
- One entry per meaningful change

Must NOT include:

- Task status
- Future plans
- Speculative design ideas

---

## Usage Rules (MUST)

- Documentation content defaults to **Simplified Chinese** unless explicitly required otherwise
- `plan.md` must exist before or alongside any large-scope design change
- `model.md` must be updated in the same change as any API / data / contract modification
- `task.md` must always reflect the latest execution state during active work
- `changelog.md` must be updated incrementally as implementation progresses

---

## Trigger Matrix (MUST)


| Change Type                  | plan.md | model.md      | task.md  | changelog.md |
| ---------------------------- | ------- | ------------- | -------- | ------------ |
| New feature / large refactor | MUST    | IF APPLICABLE | MUST     | MUST         |
| API or data model change     | SHOULD  | MUST          | SHOULD   | MUST         |
| Long-running implementation  | SHOULD  | OPTIONAL      | MUST     | MUST         |
| Small localized code change  | NO      | NO            | OPTIONAL | OPTIONAL     |

---

## Minimal Templates (MUST)

### `plan.md`

**示例（简体中文）**

- Background / Problem：本功能用于解决订单在跨系统同步时状态不一致的问题
- Scope：仅覆盖订单创建与取消流程
- Non-goals：不重构历史订单数据
- Constraints：需保持与旧 API 的向后兼容
- High-level Plan：新增状态机层，逐步切换流量

### `model.md`

**示例（简体中文）**

- Entities / Schemas：Order、OrderStatus
- Fields and Types：status:string，updated_at:timestamp
- Contracts (API / Events)：POST /orders，ORDER_STATUS_CHANGED
- Invariants：订单状态只能单向流转

### `task.md`

**示例（简体中文）**

- Current Phase：实现中
- Done：设计文档完成，基础 schema 已定义
- In Progress：订单状态机代码实现
- Blockers：下游系统状态回调未确认
- Next Steps：实现状态变更事件发布

### `changelog.md`

**示例（简体中文）**

- Date or Version：2026-01-05
- Changed Files / Modules：order_service/, api/orders
- Behavior Summary：新增订单状态机与状态校验逻辑

---

## Documentation Location Rules (MUST)

- If the change affects **only a single module**, store documentation under:

  `/<module-root>/docs/<feature>/`
- If the change affects **multiple modules** or introduces cross-module behavior, store documentation under:

  `/docs/<feature>/` at the project root

Rules:

- The location decision is based on **impact scope**, not code ownership
- Do not duplicate the same feature docs in multiple locations
- A feature documented at project root must not have parallel copies in module folders

---

## When to Create `docs/<feature>/` (MUST)

Create (or significantly update) `docs/<feature>/` when any of the following are true:

- New or changed **API / contract** (HTTP/gRPC/events, payloads, error model, idempotency)
- New or changed **data model / persistence** (schema, migration, transaction boundaries)
- New or changed **configuration** (keys, defaults, required secrets/vars)
- New **cross-module orchestration** (retries, ordering, partial failure handling, async flows)
- **Operational risk** increases (rate limits, irreversible writes, backward compatibility risks)
- Work is **multi-step / long-running** (likely to be interrupted, handed off, or context-compressed)

Do NOT create a new folder for:

- Pure refactors with no behavior/contract change
- Local code cleanup, formatting, renames with no external impact
- Tiny changes that are fully obvious from code and tests

---

## Change-Type → Required Docs Matrix (MUST)

Rules are additive.

- API/contract change → `model.md` + `plan.md` + `changelog.md`
- Data/persistence change → `model.md` + `plan.md` + `changelog.md` (+ migration notes inside `model.md` if needed)
- Config change → `plan.md` + `model.md` (config section) + `changelog.md`
- Cross-module behavior change → `plan.md` + `task.md` + `changelog.md` (and `model.md` if it affects contracts)
- Operationally risky change → `plan.md` + `task.md` + `changelog.md` (include rollback/verification in `plan.md`)
- Long-running initiative → all four files

---

## When Only `task.md` Update Is Allowed (MUST)

You may update **only** `task.md` when:

- No contract/model/config changes are introduced in the current step
- The work is strictly implementation progress (e.g., wiring, refactor inside the boundary)
- You are resuming after interruption and need to re-establish execution state

If any contract/model/config changes occur, you MUST also update `model.md` and add a `changelog.md` entry in the same change.

---

## `model.md` Contract Format (MUST)

Use one of the following formats, pick the simplest that fits:

- Table format (recommended)
  - Field | Type | Required | Constraints | Notes
- JSON schema snippets (allowed)
  - Keep examples minimal

Rules:

- Keep `model.md` **factual** (no rationale)
- Include at least one example payload for each public contract
- Record compatibility notes when changing defaults or making backward-incompatible changes

---

## `plan.md` Required Operational Slots (MUST)

Add these two sections when risk exists:

- Verification: minimal steps to prove the change works (tests, metrics, endpoints)
- Rollback: how to revert safely (feature flag, config revert, migration strategy)

---

## `changelog.md` Entry Format (MUST)

Use this structure per entry:

- Date: YYYY-MM-DD
- Summary: one line
- Touchpoints: files/modules changed
- Behavior: what changed (observable)

Keep entries short.

---

## Review Checklist (MUST)

- Documentation language defaults to Simplified Chinese unless required otherwise
- `plan.md` exists and scope/non-goals are explicit
- Any contract/model/config change is reflected in `model.md`
- `task.md` reflects current state and next steps
- `changelog.md` has a new entry for meaningful behavior changes
- No secrets/PII in any doc
- No duplicated content across files (each file keeps its single responsibility)

---

## Non-goals

This skill does NOT define:

- Code comment standards
- Repository-wide documentation policy
- User-facing documentation formats

```
**pr-change-summarizer**
这个是用来生成pr的
```md
---
name: pr-change-summarizer
description: Generate concise, module-oriented, outcome-focused Pull Request titles and descriptions based on code changes, removing redundant details and standardizing PR summaries for review.
---

## Language and Writing Rules

- The PR **must be written in Chinese**.
- **Technical terms, class names, library names, configuration keys, and APIs** should remain in English when translation would reduce clarity.
- Use **concise, technical language**. Avoid narrative or explanatory prose.

---

## Overall Structure

- Do **not** include branch names, issue IDs, or links.
- Do **not** include headings like “Summary”, “Description”, or “Changes”.
- The PR body is organized **only by modules**.

### Single-module PR

<module-name>
<title>
  <content>

### Multi-module PR

module-a
title
  content

module-b
title
  content


- Module blocks are separated by a blank line.
- Titles and contents are separated only by line breaks and indentation, not labels or prefixes.

### Title Rules
- One line only.
- Describe the primary outcome or intent, not the implementation steps.
- Use imperative or declarative technical phrasing.
- Avoid mentioning files, commands, or tools.

#### Examples
- 下沉 Spring Security 依赖至子模块
- 移除模块中遗留的 Security 配置
- 调整模块依赖边界以降低耦合

### Content Rules
- Use bullet points with - .
- Each bullet must describe a resulting state or constraint, not how it was achieved.
- Prefer what changed semantically, not where or how it was edited.

#### Information Density Constraints
- Do not describe implementation procedures
- ❌ 修改 pom.xml
- ❌ 执行 mvn dependency:tree
- Do not repeat the same fact in different wording
- One module = one core change, unless multiple outcomes are strictly independent

#### Dependency Changes
- Describe dependency intent, not mechanics.
- Examples:
- 父模块移除对 Spring Security 的隐式依赖
- 子模块显式引入 spring-boot-starter-security 以保证编译与运行期一致性
- 仅保留 spring-security-crypto 支持 BCryptPasswordEncoder

#### Configuration Changes
- Describe configuration impact, not file edits.
- Examples:
- 清理无效的 spring.security.user 与 management 相关配置
- 移除不再生效的 Actuator Security 配置

### Test and Verification Rules
- Do not include test statements for:
  - 纯依赖调整
  - 纯配置清理
  - 编译期可见性修复
- Only mention tests if:
  - Runtime behavior changes
  - New logic paths are introduced
  - Existing behavior is intentionally altered

When required, keep it minimal:
- 已通过现有集成测试验证
- 已验证相关接口行为未发生变化

### Git Execution Constraint
- This skill does not execute git commands.
- Do not include git commit, git push, or similar commands in the PR content.
- The skill’s responsibility ends at drafting PR title and description text.

### What Not to Include
- No change logs
- No step-by-step explanations
- No “why we did this” background
- No validation checklists
- No screenshots or logs

### Output Expectation

When invoked, produce only:
- A PR title
- A PR description body following the structure above

No additional commentary or explanation.

**spring-web-api-architecture-protocol**
这个是spring框架的
---
name: spring-web-api-architecture-protocol
description: Defines mandatory architecture, layering, data model boundaries, validation, exception handling, and framework-level checks for building external APIs using the Spring framework.
---
## Scope (MUST)

This skill applies when:
- Building or modifying **external-facing APIs** using Spring (Spring MVC / Spring Boot)
- Defining controllers, services, repositories, DTOs, or API error models
- Introducing validation, exception handling, or framework-level behavior

This skill does **not** define internal-only Java structure rules.
Internal structure is governed by `java-code-structure-minimalism`.

## Layering Rules (MUST)

- Allowed layers:
  - Controller → Service → Repository

### Controller
- Handles request parsing and response mapping only
- Must NOT contain business logic
- Must NOT access repositories directly
- Must remain stateless

### Service
- Owns business logic
- Owns transaction boundaries
- Coordinates repositories and external systems

### Repository
- Persistence access only
- No business logic

Additional layers (Facade / Manager / Adapter / Orchestrator) are **forbidden** unless:
- Coordinating ≥2 services, or
- Coordinating ≥2 external systems

## Data Model Boundaries (MUST)

### DTO
- Used only at API boundaries
- Must NOT be reused as domain entities
- Must NOT contain business logic

### Entity / Domain Model
- Must NOT be exposed in controllers
- Must NOT be serialized directly to API responses

### Mapping Rules
- Mapping logic belongs in the Service layer
- A dedicated mapper class is allowed **only if mapping logic is reused**
- Do NOT introduce mapper layers by default

Existence of DTOs does NOT justify additional abstraction layers.

## Validation Rules (MUST)

- Input validation must occur at the API boundary
- Use Bean Validation consistently (`@Valid`, `@NotNull`, etc.)
- Validation failures must map to a unified API error response
- Business rule validation must live in the Service layer

## Exception Handling (MUST)

- Controllers must NOT catch business exceptions
- Use centralized exception handling (`@ControllerAdvice`)
- API must expose a stable error model, containing at least:
  - `errorCode`
  - `message`
  - optional `details`
- Internal exception types and stack traces must NOT leak to API responses

## Spring-Specific Checks (MUST)

- `@Transactional` is allowed only on Service layer
- Controllers must never declare transactions
- Controllers must never access repositories
- Jackson configuration (unknown fields, date formats, naming) must be centralized
- API behavior must be deterministic under default Spring configuration

## Compatibility & Evolution (MUST)

- API changes must be backward-compatible unless explicitly declared
- Field removal or semantic change requires versioning or fallback handling
- Default values and optional fields must remain stable
- Compatibility rules apply regardless of internal refactoring

## Relationship to Other Skills

- API structure and framework rules are governed by this skill
- Internal Java code structure is governed by `java-code-structure-minimalism`
- Documentation obligations are governed by `design-change-documentation-protocol`

## Non-goals

This skill does NOT define:
- Code formatting or style conventions
- Performance optimizations
- Non-Spring framework behavior
```
