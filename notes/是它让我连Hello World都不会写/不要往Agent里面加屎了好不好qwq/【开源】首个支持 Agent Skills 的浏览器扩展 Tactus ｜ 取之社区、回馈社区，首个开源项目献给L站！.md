# 【开源】首个支持 Agent Skills 的浏览器扩展 Tactus | 取之社区、回馈社区，首个开源项目献给L站！ 

[原帖链接](https://linux.do/t/topic/1508073/1)

**作者：灿烂甜菜**  
**时间：Jan 23, 2026 8:15 pm**  

v1.0.2 已发布，兼容了硅基流动的GLM-4.7的工具参数解析失败问题（ [@gofreehj    
  
      ![speech_balloon](https://cdn.ldstatic.com/images/emoji/twemoji/speech_balloon.png?v=15)](https://linux.do/u/gofreehj) 佬友反馈）  

硅基流动的GLM-4.7稳定复现，在流式返回嵌套JSON结构时漏了最后一个’}'导致的工具参数解析失败。  

新增tryFixIncompleteJson函数，可在保留常规JSON解析逻辑的同时自动补全缺失的括号。

---

v1.0.1 已发布，添加了国际化的支持。  

目前只支持 English 和简体中文，初次安装自动检测浏览器语言并设置。  

在设置页可自行切换，同时会作为 agent 回复的语言。

---

# 为什么会有这个项目

首先我还是念念不忘站内佬友们帖子的总结场景，可以看到去年8月我分享过`gemini-cli`通过`mcp-chrome`去获取网页，再结合油猴的自动滚动，那我就可以直观的看主帖总结及佬友们的讨论梳理。

在这之后，为了摆脱比较重而且与浏览网页的割裂感，我开始尝试 AI 浏览器和浏览器扩展的方式，但是日渐形成的符合个人品味的总结习惯无法作为提示词配置进去。

随着 Agent Skills 规范的被越来越多的 agent 集成，我意识到我们在浏览器的一些工作流天然适合作为 skills 使用，那么最好的一个载体便是浏览器扩展。

我们可以把自动化流程封装在`scripts`的 js 脚本中，在`SKILL.md`说明使用场景和流程，前文说的总结提示词就可以放在`SKILL.md`中，不满意可以自己调整，还可以把额外的说明放在`references`中。

取之社区、回馈社区，自打去年加入L站，自身的AI信息面各方面展开飞快，也为了感谢站内佬的公益站，首个开源项目献给L站！

  

      [github.com](https://github.com/Castor6/tactus)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/f/7/b/f7b876bf9bbed0f855201758805b0b7b8caa82a9_2_690x344.png)

  ### GitHub - Castor6/tactus: The first browser AI assistant extension to...

    The first browser AI assistant extension to support Agent Skills, enabling AI to perform complex tasks through an expandable skill system. | 首个支持 Agent Skills 的浏览器 AI 助手扩展，让 AI 通过可扩展技能系统执行复杂任务

  

  
    
    
  

  

# 解决的痛点

1. 固定的自动化流程可以被封装为脚本，既稳定又省 tokens；
2. skills 具备可分发性，通用工作流可流传；
3. 定制化自己的工作流，不满意其中的提示词可自行微调；

# 核心功能

## Agent Skills 系统

Tactus 是首个在浏览器扩展中实现 Agent Skills 规范的产品：

- **技能导入** - 支持导入符合规范的 Skill 文件夹，包含指令、脚本和资源文件    ![image](https://cdn3.ldstatic.com/original/4X/a/d/e/adeddd241462f6420763b0d454df1225626ee1d7.png)
- **脚本执行** - 在页面中安全执行 JavaScript 脚本
- **信任机制** - 首次执行脚本需用户确认，可选择信任    ![image](https://cdn3.ldstatic.com/original/4X/7/9/2/7922625b20310384ca0ce29258ef7df151df4db5.png)

## 智能对话

- **OpenAI 兼容** - 支持 OpenAI 兼容的 API 服务商（薄荷佬、魔搭、阿里云经过验证 全是免费）
- **流式响应** - 实时显示 AI 回复，支持思维链展示
- **ReAct 范式** - 内置工具调用循环，AI 可自主决策使用工具

## 页面交互

- **智能提取** - 使用 Readability + Turndown 提取页面核心内容并转换为 Markdown
- **选中引用** - 选中页面文字后一键引用提问带上
- **上下文感知** - AI自行判断是否调用网页提取工具，如果 skill 脚本有提供则不会

![image](https://cdn3.ldstatic.com/original/4X/e/9/9/e9935bb65ead4d80ccd18291a208f4d4263ac42d.jpeg)
## 本地存储

模型配置、skills、对话历史均本地存储，这里要注意的是下载后要解压在固定的目录，如果要更新扩展，解压覆盖后在谷歌浏览器扩展管理中点击重新加载就好了。

# 首发skills

第一个 skill 献给L站，这个 skill 的脚本是利用了 discore 论坛的开放链接获取帖子的主帖及讨论区的完整内容。  

只要有完整内容，佬友们想做什么都可以，如总结还有瓦砾佬的爬楼定位等场景。  

[fetch-linuxdo-post.zip](https://linux.do/uploads/short-url/pzjyshWpgxtXn9lUXEewTL8O0i2.zip) (7.3 KB)  

解压后上传文件夹即可。

# 快速开始

## 1. 下载

从官方 Github [发布页面](https://github.com/Castor6/tactus/releases) 下载最新的 `tactus.zip` 文件。

## 2. 安装

- 在固定目录解压 `tactus.zip` 。
- 在 Chrome 中打开 `chrome://extensions/`
- 启用 `开发者模式`（右上角）
- 点击 `加载未打包的扩展程序` （左上角）
- 选择已解压的 `tactus` 文件夹。

# 项目碎碎念

- scripts脚本比较有限，限于页面操作和网页请求的js实现，也能只能用原生的，因为浏览器环境肯定是没有那些npm的依赖包。
- 不了解js的佬可以等我迭代自动操作和录制回放。
- ps：应该是首个吧 没看到有实现skills规范的扩展

# 下一步计划

- 引入 CDP 自动化作为 agent 的工具，可人工接管介入
- 操作录制一键生成可复用的 skills
- 长时稳定自动化任务挑战

# 致谢致谢

感谢 [@waili    
  
      ![clinking_beer_mugs](https://cdn.ldstatic.com/images/emoji/twemoji/clinking_beer_mugs.png?v=15)](https://linux.do/u/waili) 瓦砾佬的第一个star和一直以来的鼓励与支持，作为我 AI 的领路人，她的开源精神深深鼓舞到我。  

感谢 AionUi 开发者们的开源精神是我的榜样！  

感谢薄荷佬的公益站，gemini-3-flash 用于扩展的 agent 快速实用。  

感谢站内大佬们的羊毛及公益站，对我的 AI 平权帮助很大！  

感谢一位大佬的 UI skills 非常好用！（等我翻下链接，一时半会没找到）  

感谢我上一个帖子大佬们分享的总结L站帖子技巧，让我得以集成到skills中。

---

# 注意事项

1. 下载了扩展一定要解压在固定目录，模型配置、对话历史和 skills 都是和这个目录的扩展id关联的。
2. 如果要更新扩展，解压后替换掉文件就好了，然后在扩展管理中点击重新加载，请务必刷新已打开的页面，才可以使用。    ![image](https://cdn3.ldstatic.com/original/4X/6/f/f/6ff00c365bae8a3382e19b9f33ca1f3c2fc510f5.png)    ![image](https://cdn3.ldstatic.com/original/4X/0/f/a/0fa521312f80d9865c572a426a8cb35541cdd960.png)
