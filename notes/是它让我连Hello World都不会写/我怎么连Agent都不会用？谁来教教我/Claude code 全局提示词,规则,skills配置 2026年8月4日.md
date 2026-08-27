# Claude code 全局提示词,规则,skills配置 2026年8月4日 

[原帖链接](https://linux.do/t/topic/1167907/1)

**作者：猫南北**  
**时间：Aug 4, 2026 7:33 pm**  

#### 本帖使用社区开源推广，符合推广要求。我申明并遵循社区要求的以下内容：

- **我的帖子已经打上 [开源推广](https://linux.do/tag/2234-tag/2234) 标签：** 是
- **我的开源项目完整开源，无未开源部分：** 是
- **我的开源项目已链接认可 LINUX DO 社区：** 是
- **我帖子内的项目介绍，AI生成、润色内容部分已截图发出：** 是
- **以上选择我承诺是永久有效的，接受社区和佬友监督：** 是

_以下为项目介绍正文内容，AI生成、润色内容已使用截图方式发出_

---

我重新更改了这个主题帖,原因是:

1. 大量的文本导致我编辑帖子时电脑几乎卡死,风扇在咆哮
2. 由于claude code 的不断更新, 这些配置文件也会随时发生变动,
3. 我不是随时能更新此主题帖,所以我想了一个办法,就是把配置通过`mklink`的方式写入一个空白项目,以此来做到实时同步配置

我建立了一个仓库, 地址是:

  

      [github.com](https://github.com/bd-dxg/skills)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/2/8/d/28dc28eb3542d2dff9b9e1b4627b50a629d3856b_2_690x344.png)

  ### GitHub - bd-dxg/skills: Bddxg's curated collection of agent skills.

    Bddxg's curated collection of agent skills.

  

  
    
    
  

  

内容包含了全局提示词,全局skills

![d789cca6393d61ee600be55ccd678a48](https://cdn3.ldstatic.com/original/4X/f/c/2/fc2405e76fcfd83b72d4039f0341d033e8a9dcfa.jpeg)
本次更新了自己常用的skill ,同时已经转入 pi Agent 阵营

其核心特点是没有了 claude code 超级长的系统提示词,节省了大量token

在接入deepseek的背景下 claude code消耗 0.5 元的问题, pi 只需要 0.03 元

大家可以看看我的新帖:

[https://linux.do/t/topic/2683015](https://linux.do/t/topic/2683015)

---

谢谢L友的点赞!方便的话能给个认可吗? 点击我的头像, 选择**认可** 在 [开发调优](https://linux.do/c/develop/4) 谢谢L友
