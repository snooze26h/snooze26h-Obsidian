# 《以LLM推理的速度ship代码》 

[原帖链接](https://linux.do/t/topic/1537104/1)

**作者：哈雷彗星**  
**时间：Jan 28, 2026 7:09 am**  
> 这是一篇我好早就想转的博文
> 当时看到作者说的 你就叽里咕噜等模型出东西，先别管先出东西，拿量堆货再评效果。觉得是有一些道理在的，最近也有在尝试多开几个项目并行，习惯和脑子还是转不过来强迫症还是爱盯实施细节，然后这几天不是Clawdbot炒火了嘛 我都没在意，结果看到不知道哪贴的图 贡献者第一个的头像是他 哈哈 乐歪了 才想起来这篇博文本来就有提到 遂花点时间补上校对并分享


发布于：2025年12月28日 • 阅读时间约18分钟

原文：[Shipping at Inference-Speed | Peter Steinberger](https://steipete.me/posts/2025/shipping-at-inference-speed)

---

## 自五月以来有什么变化

vibe coding今年进步得太快了。大概五月份的时候，我还会惊叹于_某些_ prompt 能直接生成可用的代码，而**现在这已经是我的基本预期**了。我现在能产出代码的速度简直不可思议。从那时起我[烧掉了大量的 token](https://x.com/thsottiaux/status/2004789121492156583)。是时候更新一下了。

这些 agent 的工作方式很有意思。几周前有个争论说，[必须亲自写代码才能感受到架构的好坏](https://x.com/steipete/status/1997380251081490717)，用 agent 会产生一种脱节感——现在我**完全不同意**。当你和 agent 相处足够久，你会非常清楚某件事应该花多长时间，当 Codex 回来却没能一次搞定时，我就已经开始怀疑了。

现在我能开发软件的数量主要**受限于推理时间和深度思考**。说实话——大多数软件根本不需要深度思考。大多数应用不过是把数据从一个表单里搬成另一种形式，可能存到某个地方，然后以某种方式展示给用户。最简单的形式是文本，所以默认情况下，不管我想构建什么，都从 CLI 开始。Agent 可以直接调用它并验证输出——从而形成闭环。

## 模型的转变

真正让我能[像工厂一样高效产出](https://github.com/steipete/)的是 GPT 5。发布后我花了几周才意识到这一点——等 Codex 追上 claude code 的功能，再花点时间学习和理解它们的差异，然后我开始越来越信任这个模型。**现在我基本不怎么读代码了。** 我看着输出流，偶尔看看关键部分，但说实话——大部分代码我不读。我确实知道各个组件在哪、整体结构如何、系统是怎么设计的，通常这就够了。

现在重要的决策是**语言/生态系统和依赖项**。我的首选语言是：TypeScript 用于 Web 相关的东西，Go 用于 CLI，如果需要用 macOS 特性或有 UI 就用 Swift。Go 在几个月前我压根没考虑过，但后来我试了试，发现 agent 写 Go 确实很厉害，而且它简单的类型系统让 lint 很快。

在 Mac 或 iOS 上开发的朋友们：现在你们不太需要 Xcode 了。[我甚至不用 xcodeproj 文件](https://github.com/steipete/clawdis/tree/main/apps/ios)。Swift 的构建基础设施现在对大多数事情来说已经够用了。Codex 知道怎么运行 iOS 应用、怎么处理模拟器。不需要特别的配置或 MCP。

## Codex vs Opus

我写这篇文章的时候，Codex 正在跑一个巨大的、长达数小时的重构任务，清理 Opus 4.0 之前留下的烂摊子。Twitter 上经常有人问我 Opus 和 Codex 之间最大的区别是什么，为什么基准测试那么接近还有区别。我觉得基准测试越来越难信任了——你得两个都试试才能真正理解。不管 OpenAI 在 post-training 上做了什么，Codex 被训练成在开始之前会读**大量**代码。

有时它会**默默读文件读 10、15 分钟**才开始写代码。一方面这很烦，另一方面这太棒了，因为这大大增加了它修对东西的概率。相比之下 Opus 更急——适合小改动——但对于大功能或重构就不太行，它经常不读完整个文件或者漏掉一些部分，然后交付低效的结果或者遗漏什么。我注意到虽然 Codex 有时候比 Opus 慢 4 倍完成类似的任务，但我反而更快，因为我不用回去修修补补，这在用 Claude Code 的时候感觉很正常。

Codex 还让我卸掉了很多用 Claude Code 时必须玩的把戏。不用"**plan mode**"，我直接[**和模型开始对话**](https://x.com/steipete/status/1997412175615246603)，问个问题，让它搜索，探索代码，一起制定计划，当我对看到的满意了，我就写"build"或者"write plan to docs/*.md and build this"。Plan mode 感觉像是一个 hack，对于那些不太擅长遵循 prompt 的老模型是必要的，所以我们不得不拿走它们的编辑工具。我有一条[被严重误解的推文](https://x.com/steipete/status/2001228002953158928)还在流传，这让我意识到大多数人不明白 [plan mode 并不神奇](https://lucumr.pocoo.org/2025/12/17/what-is-plan-mode/)。

## Oracle

从 GPT 5/5.1 到 5.2 的飞跃是巨大的。大约一个月前我做了[**oracle **](https://github.com/steipete/oracle)——这是一个 CLI，允许 agent 运行 GPT 5 Pro 并上传文件 + prompt，管理会话以便稍后获取答案。我做这个是因为很多次当 agent 卡住时，我会让它把所有东西写到一个 markdown 文件里然后自己去查询，这感觉是重复浪费时间——也是一个闭环的机会。指令在[我的全局 AGENTS.MD](https://github.com/steipete/agent-scripts/blob/main/AGENTS.MD) 文件里，模型有时候卡住了会自己触发 oracle。我每天用好几次。这是一个**巨大的突破**。Pro 在跑遍约 50 个网站然后深度思考方面强得离谱，几乎每次都能给出正确答案。有时很快只要 10 分钟，但我也有跑了一个多小时的情况。

现在 GPT 5.2 出来了，我需要它的情况少多了。我自己有时还用 Pro 做研究，但让模型"ask the oracle"的情况从每天好几次变成了每周几次。我对此并不沮丧——做 oracle 超级好玩，我学到了很多关于浏览器自动化、Windows 的知识，终于花时间研究了 skills，之前我一直不屑于这个想法。这确实说明了 5.2 在很多实际编程任务上进步了多少。它**几乎能一次搞定**我扔给它的任何东西。

另一个巨大的优势是**知识截止日期**。GPT 5.2 到八月底，而 Opus 停在三月中——差了大约 5 个月。当你想用最新的工具时，这很重要。

## 一个具体的例子：VibeTunnel

再给你一个模型进步多大的例子。我早期投入很多精力的一个项目是 [VibeTunnel](https://vibetunnel.sh/)。一个终端多路复用器，让你可以在路上写代码。今年早些时候我几乎把所有时间都投进去了，两个月后它好到我发现自己和朋友出去玩的时候还在手机上写代码…然后决定这是我应该停下来的事情，主要是为了心理健康。那时我试图把多路复用器的一个核心部分从 TypeScript 重写，老模型一直让我失望。我试过 Rust、Go…见鬼，甚至 zig。当然我本可以完成这个重构，但那需要很多手动工作，所以在我把它放下之前一直没完成。上周我重新拿起它，给 Codex 一个**两句话的 prompt** 来[把整个转发系统转换成 zig](https://github.com/amantus-ai/vibetunnel/compare/6a1693b482fa4ef0ac021700a9ec05489a3a108f...a81b29ee3de6a2c85fd9fa41423d968dcc000515)，它跑了 5 个多小时和多次压缩，一次性交付了可用的转换。

你可能会问为什么要重新拿起它？我目前的重点是 [Clawdis](https://clawdis.ai/)，一个 AI 助手，**完全可以访问**我[所有电脑](https://x.com/steipete/status/2005213014778409280/photo/1)上的一切、[消息](https://imsg.to/)、[邮件](https://github.com/steipete/gogcli)、[智能家居](https://www.openhue.io/cli/openhue-cli)、[摄像头](https://camsnap.ai/)、灯光、[音乐](https://sonoscli.sh/)，见鬼它甚至能控制我[床的温度](https://eightctl.sh/)。当然它也有[自己的声音](https://github.com/steipete/sag/)、[发推的 CLI](https://github.com/steipete/bird) 和自己的 [clawd.bot](https://clawd.bot)。

Clawd [可以看到和控制我的屏幕](https://www.peekaboo.boo/)，有时会发点俏皮话，但我也想让他能够检查我的 agent，而获取**字符流**比看图片高效多了…这能不能行得通，走着瞧！

## 我的工作流程

我知道…你来这里是想**学习怎么更快地构建**，而我只是在给 OpenAI 写营销文案。我希望 Anthropic 正在憋 Opus 5，让风水轮流转。竞争是好事！同时，我_喜欢_ Opus 作为通用模型。我的 AI agent 如果跑在 GPT 5 上不会有一半那么有趣。Opus 有一些[特别的东西](https://soul.md/)让它用起来很愉快。我大部分电脑自动化任务都用它，当然它也驱动着 Clawd​。

自从[我十月份那次分享](https://steipete.me/posts/just-talk-to-it)以来，我的工作流程没有太大变化。

- 我通常同时做[**多个项目**](https://x.com/steipete/status/2005083410482733427/photo/1)。根据复杂度可能是 3-8 个。上下文切换会很累，我只有在家、安静、专注的时候才能做到。要同时 shuffle 很多心智模型。幸运的是大多数软件都很无聊。创建一个[查看外卖配送状态的 CLI](https://ordercli.sh/) 不需要太多思考。通常我的重点是一个大项目，卫星项目在旁边慢慢跑。当你做够多的 agentic engineering，你会对什么简单、模型可能在哪里卡住有感觉，所以经常我就放一个 prompt 进去，Codex 会跑 30 分钟然后我就得到我需要的。有时需要一点调整或创意，但通常事情很直接。
- 我大量使用 Codex 的**队列功能**——当我有新想法时就加到 pipeline 里。我看到很多人在尝试各种多 agent 编排系统、邮件或自动任务管理——到目前为止我没看到太大需求——通常我才是瓶颈。我构建软件的方式非常迭代。我做一个东西，玩玩它，看看"感觉"如何，然后得到新想法来完善它。我很少脑子里有完整的图景。当然我有个大概的想法，但通常在探索问题域的时候会发生巨大变化。所以那些把_完整想法_作为输入然后交付输出的系统对我不太行。我需要玩它、摸它、感受它、看它，这就是我如何演进它。
- 我基本**从不 revert** 或用 checkpoint。如果有什么不是我想要的，我让模型改。Codex 有时会重置一个文件，但通常它只是 revert 或修改编辑，很少需要完全回退，而是我们换个方向走。构建软件就像爬山。你不是直线往上走，你绕着它转弯，有时走错路需要往回走一点，不完美，但最终你会到达你需要去的地方。
- 我直接**提交到 main**。有时 Codex 觉得太乱了会自动创建一个 worktree 然后把改动合并回来，但这很少，我只在特殊情况下才 prompt 这个。我发现要想着项目里不同状态的认知负担没必要，更喜欢线性演进。大任务我留到分心的时候——比如写这篇文章的时候，我在跑 4 个项目的重构，每个大约需要 1-2 小时完成。当然我可以在 worktree 里做，但那只会造成很多 merge conflict 和次优的重构。注意：我通常独自工作，如果你在大团队里这个工作流显然不行。
- 我已经提到了我规划功能的方式。我一直**交叉引用项目**，特别是如果我知道我在别的地方已经解决过某件事，我会让 Codex 看 ../project-folder，通常这就够它从上下文推断出要看哪里。这对节省 prompt 非常有用。我可以写"look at ../vibetunnel and do the same for Sparkle changelogs"，因为那里已经解决了，99% 的情况下它会正确地复制过来并适配到新项目。这也是我如何搭建新项目的。
- 我见过很多系统是为了引用过去的会话。这是我从不需要或使用的另一件事。我在每个项目的 **docs 文件夹**里维护子系统和功能的文档，并在我的全局 AGENTS 文件里用[一个脚本 + 一些指令](https://github.com/steipete/agent-scripts/blob/main/scripts/docs-list.ts)强制模型读取某些主题的文档。项目越大这越有价值，所以我不是到处都用，但它对保持文档更新和为我的任务工程化更好的上下文很有帮助。
- 说到 context。我以前很勤快地为新任务重启会话。**用 GPT 5.2** 后这不再需要了。即使上下文更满，性能也极好，而且通常因为模型已经加载了很多文件所以更快。显然这只在你序列化任务或者让改动足够分开以至于两个会话不太互相触碰时才有效。Codex 没有"这个文件改了"的系统事件，不像 claude code，所以你需要更小心——另一方面，Codex 在上下文管理上**好得多**，我感觉一个 Codex 会话比 claude 能做 5 倍的事。这不只是客观上更大的上下文大小，还有其他东西在起作用。我猜测 Codex 内部思考非常精炼以节省 token，而 Opus 很啰嗦。有时模型搞砸了，[它的内部思维流泄漏给用户](https://x.com/steipete/status/1974108054984798729)，所以我见过好几次。真的，[Codex 的措辞方式](https://x.com/steipete/status/2005243588414931368)我觉得很有趣。
- Prompt。我以前用语音输入写很长很精心的 prompt。用 Codex 后，我的 **prompt 变短了很多**，我经常又开始打字了，而且很多时候我加图片，特别是迭代 UI 的时候（或者 CLI 的文字拷贝）。如果你给模型展示什么是错的，只需要几个词就能让它做你想要的。是的，我就是那个拖一个 UI 组件的截图进去写"fix padding"或"redesign"的人，很多时候这要么解决了我的问题要么让我走得足够远。我以前引用 markdown 文件，但用了我的 docs:list 脚本后就不需要了。
- Markdown。很多时候我写"**write docs to docs/*.md**"然后让模型自己选文件名。你为模型训练的内容设计越明显的结构，你的工作就越轻松。毕竟，我设计代码库不是为了方便我导航，我是工程化它们以便 agent 能高效工作。和模型较劲通常是浪费时间和 token。

## 工具和基础设施

- **什么还是很难？** 选择正确的依赖和框架是我花相当多时间的事情。这个维护得好吗？peer dependencies 怎么样？它流行吗 = 有足够的世界知识让 agent 容易处理？同样，系统设计。我们通过 web socket 通信吗？HTML？什么放服务端什么放客户端？数据怎么流、从哪流到哪？这些通常是比较难向模型解释的事情，研究和思考是值得的。
- 因为我管理很多项目，经常我让 agent 在我的项目文件夹里跑，当我发现一个新模式时，我让它"**找到我最近所有的 go 项目**并在那里也实现这个改动 + 更新 changelog"。我每个项目都在那个文件里提升了 patch version，当我回访时，一些改进已经在等我测试了。
- 当然我**自动化一切**。有个 skill 用于注册域名和改 DNS。一个用于写好的前端。我的 AGENTS 文件里有个关于我 tailscale 网络的 note，所以我可以直接说"go to my mac studio and update xxx"。
- 说到**多台 Mac**。我通常用两台 Mac 工作。MacBook Pro 接大屏幕，另一个屏幕上是 Jump Desktop 会话连到我的 Mac Studio。有些项目在那边跑，有些在这边。有时我在每台机器上编辑同一个项目的不同部分然后通过 git 同步。比 worktree 简单因为 main 上的 drift 容易调和。还有个好处是任何需要 UI 或浏览器自动化的我可以移到 Studio，它不会用弹窗烦我。（是的，Playwright 有 headless mode 但有足够多的情况那不行）
- 另一个好处是任务在那边**持续运行**，所以每当我旅行时，远程变成我的主工作站，即使我合上 Mac 任务也继续跑。我过去尝试过真正的异步 agent 比如 Codex 或 Cursor web，但我想念那种可操控性，最终工作以 pull request 结束，这又给我的设置增加了复杂性。我更喜欢终端的简单。
- 我以前玩过 slash commands，但从没觉得特别有用。Skills 替代了一部分，剩下的我继续写"**commit/push**"因为和 /commit 花一样的时间而且永远有效。
- 过去我经常花专门的日子来**重构和清理**项目，我现在更多是即时处理。每当 prompt 开始花太长时间或者我在代码流里看到丑的东西飞过，我会马上处理。
- 我试过 linear 或其他 **issue tracker**，但没有一个坚持下来。重要的想法我马上试，其他的我要么记得要么就不重要。当然我有公开的 bug tracker 给用我开源代码的人报 bug，但当我发现一个 bug，我会立即 prompt 它——比写下来然后之后再切换上下文回来快多了。
- 不管你做什么，**先从模型和 CLI 开始**。我脑子里有个[总结 YouTube 的 Chrome 扩展](https://x.com/steipete/status/2005320848543298009)的想法很久了。上周我开始做 summarize，一个把任何东西转成 markdown 然后喂给模型做总结的 CLI。先把核心做对，一旦那个工作得很好我就用一天做了整个扩展。我很喜欢它。跑在本地、免费或付费模型上。本地转录视频或音频。和一个本地 daemon 通信所以超级快。[试试看！](https://github.com/steipete/summarize/releases/latest)
- 我的首选模型是 **gpt-5.2-codex high**。还是 KISS。xhigh 除了慢得多没什么好处，我不想花时间想不同的模式或"ultrathink"。所以几乎一切都跑在 high 上。GPT 5.2 和 Codex 足够接近以 至于换模型没意义，所以我就用那个。

## 我的配置

这是我的 `~/.codex/config.toml`:

```
model = "gpt-5.2-codex"
model_reasoning_effort = "high"
tool_output_token_limit = 25000
# Leave room for native compaction near the 272–273k context window.
# Formula: 273000 - (tool_output_token_limit + 15000)
# With tool_output_token_limit=25000 ⇒ 273000 - (25000 + 15000) = 233000
model_auto_compact_token_limit = 233000
[features]
ghost_commit = false
unified_exec = true
apply_patch_freeform = true
web_search_request = true
skills = true
shell_snapshot = true

[projects."/Users/steipete/Projects"]
trust_level = "trusted"
```

这让模型一次能读更多，默认值有点小可能限制它看到的内容。它静默失败，这很痛苦，他们最终会修复的。还有，web search 居然还不是默认开启的？`unified_exec` 取代了 tmux 和我老的 `runner` 脚本，其他的也不错。还有别怕压缩，自从 OpenAI 换到新的 /compact endpoint 后，这工作得足够好以至于任务可以跨多次 compact 运行并完成。它会让事情变慢，但经常像是一次 review，模型再看代码时会发现 bug。

就这样了，目前。我计划多写一些，脑子里有不少想法积压，只是[**太享受**](https://codexbar.app/)[**构建东西**](https://x.com/steipete/status/2005393881395835045)了。如果你想听更多关于在这个新世界里如何构建的碎碎念和想法，[在 Twitter 上关注我](https://x.com/steipete)。
