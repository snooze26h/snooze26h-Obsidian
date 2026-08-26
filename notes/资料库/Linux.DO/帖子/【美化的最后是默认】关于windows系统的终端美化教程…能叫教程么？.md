# 【美化的最后是默认】关于windows系统的终端美化教程…能叫教程么？ 

[原帖链接](https://linux.do/t/topic/1504035/1)

**作者：连朝雨**  
**时间：Jan 22, 2026 8:03 pm**  

# 序言
> Attention
> 我更新到claude code 2.1.17以后出现了些问题，我不知道是ccline的问题还是美化文件的问题，在排查 > **更新 20260123：** 哈雷佬的补丁只能在npm方式下载的claude code里面生效，因为这种方式才有目标文件cli.js，所以最好还是用npm下载cc 瞎折腾什么呢 A/


我是一个颜值至上的人，因为尝试到了 ccg 的好用，所以最近一直在用 cli。然而对着黑漆麻鸡一片丑的批爆的 powershell，我实在是忍不了了，所以就开启了美化之旅。为了给自己留个足迹方便以后使用其他电脑方便，也是因为看到站内有别的佬友遇到了同样的需求，所以写这么一个文档记录一下。  

还有一件事（老爹脸），我是在 windows 下折腾的，macOS 我直接使用 iterm2 了，没折腾这个。  

另外，后面我会尽量提供 github 仓库的链接，并且能在 L 站内找到作者的都会圈一下，希望大家也去支持一下作者们，感谢前辈们的辛勤付出研究，我们才可以有这么多选择。 # 准备

## 终端软件

既然是美化，那我们就得选择一个好用、高自定义程度的终端软件，我这里使用的是 **WezTerm**，我觉得这个配置也好写，美化出来也不赖。  

仓库链接如下： [GitHub - wezterm/wezterm: A GPU-accelerated cross-platform terminal emulator and multiplexer written by @wez and implemented in Rust](https://github.com/wezterm/wezterm)

- 安装方式： - 从 github 仓库下 - 也可以去官网 Download 页面下载： [Windows - Wez's Terminal Emulator](https://wezterm.org/install/windows.html)

## 提示符渲染

已经使用了 WezTerm 的小朋友可能会疑惑：哎？为什么我的 wezterm 提示符还是和 powershell 里面没有区别啊？那我下这个有什么用？实际上，提示符是有一个单独的美化工具。这里大家其实有很多选择，譬如 **oh-my-posh**。但我这里使用的是 **starship**，原因是 `omp` 好像会出现编码错误的情况？我在站内刷到有人提起过，而 **starship** 则相对更轻便、快速。并且，最好的一点是 **starship** 只需要一个配置文件就可以完成所有的配置项，而且还有很多预设主题可以选择。所以我最终选择使用 **starship**。  

仓库链接如下： [GitHub - starship/starship: ☄🌌️ The minimal, blazing-fast, and infinitely customizable prompt for any shell!](https://github.com/starship/starship)

- 安装方式（windows）： - Winget：`winget install --id Starship.Starship`
- 配置方式： - 首先在 powershell 中输入 `$PROFILE`，会自动打开 powershell 的配置文件。 - 在配置文件中输入 `Invoke-Expression (&starship init powershell)` 并保存即可。
- 配置文件绝对路径：`C:\Users\{username}\.config\starship.toml` （如果没有自己建一个即可）
- 预设网站： [Presets | Starship](https://starship.rs/presets/)

## 字体

有些佬友可能对自己的字体蛮满意的，但是发现在终端发现提示符里出现了乱码（因为一开始我字体就没问题，其实我没遇到，不知道什么情况）。其实是因为提示符里有一些特殊符号，但是正在使用的字体没有相关符号导致的。这种情况需要用 **nerd** 字体。我个人使用的是 **Maple Mono NF CN** 这个字体，貌似还是等宽的，我很喜欢。  

仓库链接如下： [GitHub - subframe7536/maple-font: Maple Mono: Open source monospace font with round corner, ligatures and Nerd-Font icons for IDE and terminal, fine-grained customization options. 带连字和控制台图标的圆角等宽字体，中英文宽度完美2:1，细粒度的自定义选项](https://github.com/subframe7536/maple-font)

## CCometixLine（CCLine）

许多佬友使用终端应该有很多是在用 claude code 吧？我昨天刚从佬友那里发现了一个 status bar 的工具？可以这么说吧，是本站哈雷佬搞的工具。在这里也是要感谢一下哈雷佬 [@Haleclipse    
  
      ![dizzy](https://cdn.ldstatic.com/images/emoji/twemoji/dizzy.png?v=15)](https://linux.do/u/haleclipse) ！  

仓库链接如下： [Releases · rere43/CCometixLine · GitHub](https://github.com/rere43/CCometixLine/releases)

- 配置方式： - 其实哈雷佬有非常详细的文档说明（点赞！），我这里就不画蛇添足了，链接贴上： [CCometixLine/README.zh.md at master · rere43/CCometixLine · GitHub](https://github.com/rere43/CCometixLine/blob/master/README.zh.md)
> Note
> 哈雷佬这两天刚根据cc最新版本2.1.15发现了一个bug，并做出了修复，具体文章链接为：[太不容易了 终于修好了 | CC状态栏字符残留&花屏](https://linux.do/t/topic/1502888)


# WezTerm 的美化

上面的步骤都搞完，其实就已经很好看了，但是我作为一个重度颜值控晚期，怎么能局限于此呢？于是我在研究了前人的仓库之后，进行二次改造（感谢 Claude Code），稍微改造了一点。  

首先提供一下前人的模板，我的其实是稍微修改了一点点地方。原仓库链接： [GitHub - KevinSilvester/wezterm-config: My WezTerm Config](https://github.com/KevinSilvester/wezterm-config)

- 安装方式： 把仓库下载下来放到 `C:\Users\{username}\.config\wezterm` 文件夹下就可以了，我记得是直接自动读取了。

我自己的仓库： [GitHub - sty-sun/WezTerm-Config: 基于前人改造的WezTerm终端美化配置文件](https://github.com/sty-sun/WezTerm-Config?tab=readme-ov-file)  

先给大家看看效果图吧：

![image](https://cdn3.ldstatic.com/original/4X/9/9/c/99ced59c1a5b9cda6235fa0a5617e2f805342db2.jpeg)  

分栏的效果：  

![image](https://cdn3.ldstatic.com/original/4X/8/2/7/827355c6628f3a1df9c78027064cc3b3ef7f8f17.jpeg)
和之前的仓库相比，做出的改变主要有：

- 状态栏：新增了tab数、分栏数的统计
- 还有一些别的改变，但是没啥太大用处也不直观我就不列明了 （后续还要继续改，但是先留痕一下吧，不知道什么时候有空了 ）

# 总结

虽然有句话说美化的最后是默认，但是我还是愿意走一走这条路，欣赏一下沿途的风景的。  

感谢前人的付出，佬友的贡献与指导。  

如果哪里有问题，还请指正！
