# 「开源自荐」Grok2api二开优化，新加多agents并行工作的console类模型（内置web+x搜索），16个agents并发搜索助力佬们搜索自由+防403版本的一键部署！（修复fast403） 

[原帖链接](https://linux.do/t/topic/2239921/1)

**作者：久**  
**时间：May 24, 2026 7:06 pm**  

本项目思路来自于[https://linux.do/t/topic/2191735](https://linux.do/t/topic/2191735)  

[https://linux.do/t/topic/2297018](https://linux.do/t/topic/2297018)

  

      [github.com](https://github.com/jiujiu532/grok2api)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/6/f/5/6f50ab8fc86f650ea66c444b662393f88ab9910b_2_690x344.png)

  ### GitHub - jiujiu532/grok2api

    通过在 GitHub 上创建帐户来为 jiujiu532/grok2api 开发做出贡献。

  

  
    
    
  

  

**如果对你有帮助，点个 Star 吧～**

二开增强的部分  

1.console模型的支持  

2.界面添加一些组件  

3.**防封版本的一键部署**  

4.console次数的简单压测（取了保守30次/15分钟，具体可看readme）  

5.console内置x+web搜索，无需安装mcp或者skill

---

| 模型名 | reasoning effort | 说明 |
| --- | --- | --- |
| grok-4.3-console | 用户传入（默认 medium） | 免费账号 |
| grok-4.3-low | low（固定） | 免费账号 |
| grok-4.3-medium | medium（固定） | 免费账号 |
| grok-4.3-high | high（固定） | 免费账号 |
| grok-4.20-0309-console | 默认 | 免费账号 |
| grok-4.20-0309-reasoning-console | 固定 reasoning | 免费账号 |
| grok-4.20-0309-non-reasoning-console | 无 reasoning | 免费账号 |
| grok-4.20-multi-agent-console | 用户传入（默认 medium） | 免费账号，多智能体，agent 数量由 effort 决定 |
| grok-4.20-multi-agent-low | low（固定）→ 4 agents | 免费账号，多智能体 |
| grok-4.20-multi-agent-medium | medium（固定）→ 4 agents | 免费账号，多智能体 |
| grok-4.20-multi-agent-high | high（固定）→ 16 agents | 免费账号，多智能体 |
| grok-4.20-multi-agent-xhigh | xhigh（固定）→ 16 agents | 免费账号，多智能体 |
| grok-build-console | 默认 | 免费账号，Grok Build 0.1 |

---

体验地址  

[https://linux.do/t/topic/2193859](https://linux.do/t/topic/2193859)

前几天在复习+考试，发的有点晚了  

**强烈推荐grok-4.20-multi-agent-xhigh，16个Agents并发工作爽飞了**

![image](https://cdn3.ldstatic.com/original/4X/4/5/b/45b6a41eba9f274c19babc734cdb34f48ee8b58a.png)
![f87abc4cdd3d912c92ef3718d193852a_720](https://cdn3.ldstatic.com/original/4X/e/9/b/e9bb710c0060596773c79757c1a519406703d657.jpeg)
![Screenshot_2026-05-26-09-48-17-519_me.rerere.rikkahub-edit](https://cdn3.ldstatic.com/original/4X/a/a/f/aaf183af21f606ba52a274cadd416c520c522528.jpeg)
**致谢各位佬提的pr，以及提供的思路**  

[@testG](https://linux.do/u/testg) [@Sakur](https://linux.do/u/sakur) [@Tizenry](https://linux.do/u/tizenry) [@Chenyme](https://linux.do/u/chenyme) [@xunxun    
  
      ![two_hearts](https://cdn.ldstatic.com/images/emoji/twemoji/two_hearts.png?v=15)](https://linux.do/u/xunxun)

---

#### 本帖使用社区开源推广，符合推广要求。我申明并遵循社区要求的以下内容：

- **我的帖子已经打上 [开源推广](https://linux.do/tag/2234-tag/2234) 标签：** 是
- **我的开源项目完整开源，无未开源部分：** 是
- **我的开源项目已链接认可 LINUX DO 社区：** 是
- **我帖子内的项目介绍，AI生成、润色内容部分已截图发出：** 是
- **以上选择我承诺是永久有效的，接受社区和佬友监督：** 是

_以下为项目介绍正文内容，AI生成、润色内容已使用截图方式发出_

---
