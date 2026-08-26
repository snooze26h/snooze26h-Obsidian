# 昨天做的Office插件（Claude in Word！）今天打磨得差不多了，还是不选择闭源了吧，没什么意思 

[原帖链接](https://linux.do/t/topic/1544743/1)

**作者：Cimix**  
**时间：Jan 30, 2026 1:01 am**  

书接上文：

    
  
    
    ![](https://linux.do/user_avatar/linux.do/cimix/48/748494_2.png)
    
      [今天没活，又来分享纯Vibe小玩具，这次是Word Add-on](https://linux.do/t/topic/1540624) [开发调优](https://linux.do/c/develop/4)
    
  

  > 因为发现可在商店里下载的 Add-on 不是要付费服务就是太卡太重，于是决定自己写个轻便好用的，直接使用Office API来获取用户操作和格式信息，快捷地使用右键菜单和侧边栏操作。 > 基础功能完成计时：2 Hours 
>  ![image](https://cdn3.ldstatic.com/original/4X/4/f/2/4f2290eaf1354a9bbbfc1e1d6e89cc842273d357.png) 
> 我弄的Gemini首字延迟太高了，Gif裁掉了几百帧，所以可能会闪烁 
> [1] 
> UI设计想要尽量贴合Microsoft的设计风格，重构了一版，让其不会有太多…


强兼OpenAI / Anthropic / Gemini 格式API，推荐使用OpenAI格式

功能演示：

### 1、跟随word启动（暂时不选择开机启动，照顾开机进程敏感型人群，但是启动会有点慢，因为需要处理运行环境，也就是说你不需要电脑上安装NodeJS也可以用）

```
--install-startup // 需要注销一下 / 重启电脑 才能生效
--wait-for-word // 立即生效

// 建议两个命令都执行一下

WriteBot.exe --install-startup
wscript.exe WriteBot.vbs --wait-for-word
```

![image](https://cdn3.ldstatic.com/original/4X/9/8/e/98e8f09ffc044bde8f2197820fd6adc4f08bd86e.png)

### 2、自动捕获用户使用鼠标选择的文字内容

- 我有鼠标乱动症

![11](https://cdn3.ldstatic.com/original/4X/1/8/9/189e707b4729ba417286a5474bebfcd1bfaf7407.gif)

### 3、辅助排版，伴随着分块处理和全局撤销功能

*还有一些瑕疵不过现在差不多了  

![12](https://cdn3.ldstatic.com/original/4X/0/f/1/0f173aa2cd758c3ad98e3386b527ccf000e4cc44.gif)

### 4、与Office风格统一的UI设计

- I tried my best…

### 仓库地址

- README的打包流程尚有优化的空间，就暂时只保留使用教程了

  

      [github.com](https://github.com/funkpopo/writebot)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/6/d/5/6d569a8646f85f1b0142c036030268c5005424d4_2_690x344.png)

  ### GitHub - funkpopo/writebot

    通过在 GitHub 上创建帐户来为 funkpopo/writebot 开发做出贡献。

  

  
    
    
  

  

2026.01.31  00：02

    
  

![](https://linux.do/user_avatar/linux.do/cimix/48/748494_2.png) Cimix:[](https://linux.do/t/topic/1544743/78)
  > 尝试了多种方案，最后还是决定提供一个渠道将引用写入后台服务。> v1.0.1版本，执行
> ```
> WriteBot.exe --install-service
> ```
> 即可将应用作为自启服务加入本机的系统服务列表
> ![image](https://cdn3.ldstatic.com/original/4X/e/d/f/edf59cd35b2f42beee97175ce919b2a2a9ac9f45.png)
> 如果需要移除，可以执行
> ```
> WriteBot.exe --uninstall-service
> ```
> 如果不需要服务形式，而是前台运行，可以双击执行exe，但是会有一个命令行窗口存在，由于我尚未开始设计GUI，所以会比较影响非服务类型的PC使用体验

> 感谢佬友反馈！
