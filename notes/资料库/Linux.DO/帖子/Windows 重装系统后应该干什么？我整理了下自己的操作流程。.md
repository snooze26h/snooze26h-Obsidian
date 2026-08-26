# Windows 重装系统后应该干什么？我整理了下自己的操作流程。 

[原帖链接](https://linux.do/t/topic/1361194/1)

**作者：ℑ𝔤𝔫𝔦𝔰**  
**时间：Dec 25, 2025 12:05 am**  

## 安装驱动

建议重装完成后离线安装驱动，需要**提前下载**好电脑所需要的驱动。

- **芯片组驱动（主板 / 笔记本官网）**
- **显卡驱动（[NVIDIA](https://www.nvidia.com/en-us/geforce/drivers/) / [AMD](https://www.amd.com/en/support/download/drivers.html) / [Intel](https://www.intel.com/content/www/us/en/download-center/home.html) 官网）**
- **下载全量的 [Snappy Driver Installer](https://sdi-tool.org/)**

如何跳过联网？

- 方法一：连接到网络界面 Shift + F10 输入：`oobe\bypassnro` 重启
- 方法二：连接到网络界面 Shift + F10 输入   ``` reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\OOBE /v BypassNRO /t REG_DWORD /d 1 /f ```   - 再输入 `shutdown /r /t 0` 重启
- 方法三：使用 PE 工具安装系统，选择跳过启动设置

**查看设备驱动：**

- 右键「此电脑」→ 管理 → 设备管理器
- 找到 或 的设备

**缺失的驱动：**

- 如果还有未安装的驱动可以在设备管理器 **右键**或 的设备→**属性**→**详细信息**→选择 **硬件Id**（例如`PCI\VEN_1022&DEV_15E2&CC_0480`）→**复制问AI**这是什么设备需要下载什么驱动

---

## 激活 Windows

如果重装的系统版本不是家庭版，可能会掉激活

可以在联网后 **以管理员运行 Powershell** 执行以下代码选择激活方式【[Microsoft Activation Scripts | MAS](https://massgrave.dev/)】

```
irm https://get.activated.win | iex
```

---

## 解决缺失 DLL & C++ Redistributable

- [DirectX 修复工具增强版](https://www.zysoftware.top/post/10.html)
- [Microsoft.Net](https://dotnet.microsoft.com/zh-cn/download/dotnet)

---

## 系统优化设置

根据个人电脑使用习惯，推荐 2 款开源工具（有_清理，暂停更新，恢复 Win10 资源管理器，Win10右键_等功能）

- [GitHub - Chuyu-Team/Dism-Multi-language: Dism++](https://github.com/Chuyu-Team/Dism-Multi-language)
- [GitHub - ZyperWave/ZyperWinOptimize: ZyperWin++](https://github.com/ZyperWave/ZyperWinOptimize)

---

## 关闭安全中心

- 关闭Windows自带的安全中心是为了防止误删软件（之前下载一些红队相关软件被无情删除）
- 不要让电脑裸奔，最好安装一个杀毒软件，还是看个人喜好，免费的有[火绒](https://www.huorong.cn/)、[360](https://weishi.360.cn/)，付费的有[卡巴斯基](https://www.kaspersky.com.cn/)等
- 把信任的文件或者文件夹放进信任区

**建议：**

- 有良好使用习惯
- 安装可靠杀软

---

## 文件夹分类（推荐结构）

- 个人推荐在非 C 盘建立统一结构
- 文件夹层级越多，找起来越麻烦
- 我个人的文件夹结构大概如下，有多个盘的佬也可以根据自己的想法分类

```
D：
├── WindowsApp # 放置一些常用软件，大部分软件都可以放这里
│   ├── JetBrains
│   │   ├── IDEA
│   │   ├── PyCharm
│   │   └── ...
│   ├── qBittorrent
│   ├── IDM
│   └── ...
├── DevKit # 一些软件开发工具包和脚本可以放这里
│   ├── Python
│   ├── Java
│   ├── Cygwin64
│   ├── ...
├── Tools # 工具类软件（也可以和WindowsApp合并）
│   ├── Revo Uninstaller Pro
│   ├── 图吧工具箱
│   └── ...
├── Projects # 项目、代码
│   ├── VSCodeProjects
│   ├── PyCharmMiscProject
│   └── ...
├── Games
├── Work
└── Backup
```

**避免 C 盘臃肿？移动“个人文件夹”（文档 / 下载 / 桌面等）** 的几种方法（可选）

- 打开资源管理器 → 右键例如【文档】 → 属性 → 切换到【位置】选项卡 → 点击【移动】 → 选择 D 盘或其他盘的新文件夹 → 确认并允许移动文件
- 设置 → 系统 → 存储 → 高级存储设置 → 新内容保存位置
- 注册表`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList ` → 修改 ProfilesDirectory 值`%SystemDrive%\Users` → 重启

---

## 安装软件

接下来就是软件安装环节了

最好要和自己的文件结构对应上

下面是我认为大部分佬都能用到的软件

**推荐**使用

- [UniGetUI](https://www.marticliment.com/unigetui/)进行软件管理

| 分类 | 软件名 | 说明 |
| --- | --- | --- |
| 浏览器 | Google Chrome | 主流浏览器，兼容性和扩展生态优秀 |
|  | Mozilla Firefox | 开源浏览器，注重隐私 |
| 压缩 / 解压 | 7-Zip | 轻量开源，高压缩率 |
|  | Bandizip | 界面友好，支持多格式 |
|  | WinRAR | 老牌压缩工具 |
| 编辑器 | Notepad3 | 文本编辑器 |
| 输入法 | 小狼毫（ 雾凇拼音） | 高度可定制输入法 |
| 媒体播放 | PotPlayer | 功能强大的本地播放器 |
|  | VLC Media Player | 开源全格式播放器 |
| 办公软件 | Microsoft Office | 商业办公套件 |
|  | WPS Office | 国产办公软件 |
| PDF / OCR | ABBYY FineReader | 专业 OCR 与 PDF 处理 |
| 写作 / 笔记 | Typora | 所见即所得 Markdown 编辑器 |
|  | Notion | 知识管理与协作 |
| 系统工具 | Everything | 极速本地文件搜索 |
|  | Dism++ | Windows 系统维护 |
|  | Revo Uninstaller Pro | 深度卸载清理残留 |
|  | ZyperWin++ | 系统优化与精简 |
|  | 图吧工具箱 | 硬件检测与装机工具合集 |
|  | DiskGenius Pro | 磁盘管理与数据恢复 |
| 下载 / 网络 | FDM | 免费多线程下载加速 |
|  | qBittorrent Enhanced Edition | 开源 BT 下载工具 |
|  | LocalSend | 局域网跨平台传输 |
| 文件工具 | File Converter | 右键菜单文件格式转换 |
|  | OpenHashTab | 文件哈希校验 |
| 安全软件 | 火绒安全 | 轻量级安全防护 |
| 开发工具 | Visual Studio Code | 轻量代码编辑器 |
|  | JetBrains IDEs | IDEA / PyCharm 等 IDE |
|  | Visual Studio IDE | .NET / C++ 开发 |
|  | TortoiseGit | Git 图形化客户端 |
|  | Cygwin | 类 Linux 命令行环境 |
|  | Android platform-tools | adb / fastboot 工具 |
| 运行环境 | nvm-windows | Node.js 版本管理 |
|  | Java（Oracle JDK） | Java 开发与运行环境 |
|  | Python | 通用编程语言 |
| 容器 | Docker Desktop | 容器化平台 |
| 辅助 | STranslate | 划词翻译工具 |

## 系统修复（不到万不得已不要重装，修复下系统还能接着用）

如果不是硬件问题，可以尝试微软官方的 [Windows 安装媒体](https://go.microsoft.com/fwlink/?linkid=2156295)

这个软件可以识别到自己系统的版本并下载相同的镜像 **（不支持 LTSC 版本）**

运行后选择制作ISO镜像 → 完成后双击镜像文件进行修复（选择保留设置！选择保留设置！选择保留设置！）

可以解决：

- Windows 更新报错
- 蓝屏 / 黑屏报错
- 启用或关闭 Windows 功能出现错误
- 重置系统部分设置
