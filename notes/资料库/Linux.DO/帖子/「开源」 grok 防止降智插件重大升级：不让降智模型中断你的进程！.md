# 「开源」 grok 防止降智插件重大升级：不让降智模型中断你的进程！ 

[原帖链接](https://linux.do/t/topic/2769461/1)

**作者：Ljj**  
**时间：Aug 17, 2026 7:09 pm**  

#### 本帖使用社区开源推广，符合推广要求。我申明并遵循社区要求的以下内容：

- **我的帖子已经打上 [开源推广](https://linux.do/tag/2234-tag/2234) 标签：** 是
- **我的开源项目完整开源，无未开源部分：** 是
- **我的开源项目已链接认可 LINUX DO 社区：** 是
- **我帖子内的项目介绍，AI生成、润色内容部分已截图发出：** 是
- **以上选择我承诺是永久有效的，接受社区和佬友监督：** 是

_以下为项目介绍正文内容，AI生成、润色内容已使用截图方式发出_

---

最近阴间的马斯克在 grokfree 上面塞了一个不思考的 grok，原版的降智插件可以找到他并且保证下一次不降智，但是这一次无法避免。  

而这个模型，它只要发出去这个对话，它就会立刻中断，进程，无论做到哪里。  

导致用起来很烦躁。  

于是我们的插件迎来重大升级：

![image](https://cdn3.ldstatic.com/original/4X/f/d/f/fdf05f19cfa791fe3c0f90c64cc534905e993161.png)  

现在只要识别到由那个不思考的降智模型发出的降智信号，就会立刻掐断他的，不发给用户，立马换号、换出口重试，直到成功为止。重试最多 5 次，如果还是降的话，就通知用户。  

![image](https://cdn3.ldstatic.com/original/4X/b/d/0/bd00e8452d439dbd83216a5f2a0eb7d21850623c.png)  

这样子的话，我们就可以非常好的连续使用不降智的 grok 了。  

![image](https://cdn3.ldstatic.com/original/4X/5/b/d/5bd1e2a5f459b7be6e9ab72e412b2853c5a54596.png)
至于如何使用，再强调一遍：每个守护节点后面不是一个节点，而是一个代理池，要么买家宽要么自己搭建 resin。不要再有人拿着一个连接着 clash代理节点来问我为什么不行已经加上友情链接  

项目地址

  

      [github.com](https://github.com/lij768423-svg/grok2api-egress-enhancements)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/a/0/c/a0c617b251982fbd6c126558849638955a2a9cf7_2_690x344.png)

  ### GitHub - lij768423-svg/grok2api-egress-enhancements: Grok2API & CPA egress quality guard with proxy...

    Grok2API & CPA egress quality guard with proxy recovery, quarantine, migration, and operations UI

  

  
    
    
  

  

注册机,前文，成本，理论知识都可以看这篇

    
  
    
    ![](https://cdn.ldstatic.com/user_avatar/linux.do/lij768423/48/2354638_2.png)
    
      [【开源】grok 防降智插件更新：新的降智模型出现](https://linux.do/t/topic/2756217) [开发调优](https://linux.do/c/develop/4)
    
  

  > 本帖使用社区开源推广，符合推广要求。我申明并遵循社区要求的以下内容：> 我的帖子已经打上 [开源推广](https://linux.do/tag/2234-tag/2234) 标签： 是
> 我的开源项目完整开源，无未开源部分： 是
> 我的开源项目已链接认可 LINUX DO 社区： 是
> 我帖子内的项目介绍，AI生成、润色内容部分已截图发出： 是
> 以上选择我承诺是永久有效的，接受社区和佬友监督： 是
> 以下为项目介绍正文内容，AI生成、润色内容已使用截图方式发出 
> 今天多出现…


新人开发者，可以点个 star 支持一下哈哈哈，这回很多公益站应该会好很多了  

仓库有一键安装提示词

![image](https://cdn3.ldstatic.com/original/4X/e/f/2/ef2c0384d6662fe20a0d3a6d2dc16e5b424a7b41.png)
