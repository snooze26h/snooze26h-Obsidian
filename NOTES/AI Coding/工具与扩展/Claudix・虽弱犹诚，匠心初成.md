# Claudix・虽弱犹诚，匠心初成 

[原帖链接](https://linux.do/t/topic/1166801/1)

**作者：哈雷彗星**  
**时间：Nov 13, 2025 5:21 am**  

[github.com](https://github.com/Haleclipse/Claudix)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/4/6/2/4622b086cebe0e74bcd2a51f12218d3da3fc76da_2_690x344.png)

  ### GitHub - Haleclipse/Claudix: Gorgeous Claude Code Extend for VS Code.

    Gorgeous Claude Code Extend for VS Code.

  

  
    
    
  

  

Old : [Claudex ・也许是未来最好用的 Claude Code for VSCode Extension](https://linux.do/t/topic/1014194)  

Future : [非著名CC扩展之 Jetbrains 版](https://linux.do/t/topic/1237331)

从 [Claudia](https://github.com/winfunc/opcode) 到 [Claudiatron](https://linux.do/t/topic/781100)

再从 解答数 10 到 350, 我在L站好像花了好多好多时间在捣鼓 Anthropic 周边的东东

帮助很多新入坑CC的佬友解决各种奇奇怪怪的使用问题 甚至自费时间主动请求远程协助  

(省去信息沟通过程和时间，因为实在是有太多不是很会描述自己的问题的人了)  

调侃自己是客服哈哈，实际上单纯只是解答问题有瘾 有点点上头

但其实自我觉得是个菜菜 虽然一直热爱编程吧 可现实上每天都会遇到有各种形式的瓶颈  

非常恼人 哪能咋办 只能花更多的时间变着法“啃”它

在这个过程中 我本来是追捧 TUI 那套玩意的  

CC已经在概念上肥肠超前了 以CLI之名无差别侵入各路工作流 一切皆文本 感觉没GUI就像直接扔掉了宿主平台的束缚 那么为什么会突然搞上 Claudia 呢？  

一是当时发现有非常多有问题的佬友完全摸不清楚这个配置配在哪 那个文件怎么放 那么GUI附带指引是解决这个问题的最佳手段。  

喏一看 Claudia 的星星 都10k多了 （没错我就是蛐蛐星星哈哈）。  

他那个代码我都无语了 所有React组件全扔一个文件夹里平级 一看就是CC代工的

而且我不喜欢Tauri（概念很好，效果很拉）  重抄了一版 Electron 的 在配置上这个问题初见端倪 然后同时有很多佬友在 claudia 的基础上各种二改 加诸如中转站配置啊之类的配置页 ，好是都挺好。也越发的显现出一个问题 各种CC-xxxx的工具 接在Tauri类webview的屁股后面弄了一个接一个的“电子小垃圾” 他不够集中啊 所以应该弄一个CC的“大号电子垃圾” 一站式包揽CC各种配置各种能力和信息渠道(市场)

对他就是 Cursor.

有一说一 Cursor二开VSCode弄出来的小细节是真滴多 诶所以兜兜转转 先用VSCode 做一个扩展插件 就能少做很多编辑器方面的工作 一顿捣鼓就有了 Claudex（名字烂大街了恼，现在我就改用回了当初开的现在和Cursor 2.0 Agent很像的一个坑的名字）

![image](https://cdn3.ldstatic.com/original/4X/a/a/c/aac41dbcc68957943690acc06b3c4165af29296f.jpeg)
这是个前后端分离的玩意 到时候也很好出货堆功能 基本上都挺满意的

![image](https://cdn3.ldstatic.com/original/4X/e/5/4/e54965248e040a7756e4e93e5feba97f87f881b0.png)
再配上和Cursor操作逻辑相当的设置页面  

能搬一部分 Cursor 的交互过来（受制于VSCode API对齐）

ACE 也会合并到custom tool

这里要特别感谢 [@WenDavid](https://linux.do/u/wendavid) 以往在思维路线上给予了我很多的帮助 让我这个小笨蛋变成了大笨蛋捏

好了 碎碎念般的故事讲完了（思绪比较乱 见谅）

以及为什么是 [社区孵化](https://linux.do/c/incubator/102) 呢  

我觉得非常符合呐 社区里总结来的需求 诞生于这里  

那我想我需要 什么支持呢 似乎好像没有…  

有了. 留下一句鼓励吧 在我深夜懊恼的时候 看到会感到振奋  

就像我翻出来的之前的一个留言 在我做宝可梦还比较青涩没头脑的时候

![](https://cdn3.ldstatic.com/original/4X/2/2/5/225c9dd24b9fe77bfd9b1aa1a887c1802413dd04.png)  

如果扩展好用的话  

如果不好用的话 那么也很需要你真诚的建议和反馈  

我这人面对夸赞是完全害羞状。。。（大概是自我认知感到太无力了
所以如果认可我的肝努力的话 主页「开发调优」点一个认可吧 谢谢大家~

---

### 修复记录

优化`@`触发引用的 文件搜索结果列表内容 使用Fuse.js解决规则导致结果太过死板的问题

### 待追加功能 记录与指南

- 全面的设置页 - 供应配置商切换（高优先级） - 会话使用Haiku模型渠道自动起名 - bypassPermissions追加Initial Permission Mode
- 文件目录树配合Shift触发拖拽上屏文件引用
- 框选选区代码区段追加•右键菜单和快捷键
- TodoList/FileEditList挂到PromptInputBox上实时追加统计 用做于Code Review的入口
- Plan模式计划完成后权限审批卡住的问题
- SubAgent执行过程增添二级查看页予以包裹
- Edit或Write权限审批时 同时打开编辑器Diff视图标签页以供查看
- ToolMessage diff视图代码高亮着色
- 模型载入初始化数据缺Haiku,需复核问题
- 实施 /rewind 即 checkpoint (较复杂 略慢)
- ACE custom tool 附加
- 快捷键统一追加 部分设定点 例如允许改 Shift+Enter为发送等
- 权限审批提示框寻找更为合适的 UI/UX
- 复核权限审批对于指定模式上对应放行逻辑是否正确

指南tip:

- 调出命令面板(Ctrl/Cmd+P)输入 Claudix 即可找到打开视图
- 仅需聚焦到Claudix上即可使用Shift+Tab切换模式
- @文件列表上下键可导航，PageUp/Down 即可翻页 暂5条
- 模式中 Agent = AcceptEdit ,仅因名字过长予以替换表达
