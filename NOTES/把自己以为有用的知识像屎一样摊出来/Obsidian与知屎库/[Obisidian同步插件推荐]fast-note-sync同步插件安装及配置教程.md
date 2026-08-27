# [Obisidian同步插件推荐]fast-note-sync同步插件安装及配置教程 

[原帖链接](https://linux.do/t/topic/1518847/1)

**作者：无花僧**  
**时间：Jan 26, 2026 4:57 pm**  

## 写在前面

obsidian我已经用用停停好几次，很早之前一直纠结于其同步问题，从最早的 **Remotely Save**，到前段时间刚尝试部署的**self-hosted-livesync**同步方案，但是总感觉并不好用，Remotely Save只能同步文档和附件，最早我是用one drive进行同步，然后又尝试了nas开放webdav进行同步，可惜这套方案对于手机端使用很不友好，同步效果并不出色。  

基于这些原因，我后面弃用了，改用siyuan，siyuan笔记很多点挺适合国人写作习惯，但是同步问题也是一项，配置同步功能缺失，手机端与pc端差异较大。最近发现了self-hosted-livesync方案，于是我又重新配置起来，但是这个方案实际使用的时候，配置项目太多了，反而抬高了使用门槛，而且用了几天后，一直出现错误，于是我继续寻找，终于又找到了**fast-note-sync**方案，于是昨天晚上赶紧测试了下，发现确实好用，而且配置简单，同步速度也挺快，同时支持空文件夹&设置、插件、主题等一起同步。  

为此，特意推荐给大家，让大家增加一个选择。

## 插件介绍

快速、稳定、高效、任意部署的 Obsidian 笔记同步&备份插件  

可私有化部署，专注为 Obsidian 用户提供无打扰、丝般顺滑、多端实时同步的笔记同步&备份插件，支持 Mac、Windows、Android、iOS 等平台，并提供多语言支持。

- **极简配置**：无需繁琐设置，只需粘贴远端服务配置即可开箱即用。
- **笔记实时同步**：自动监听并同步 Vault (仓库) 内所有笔记的创建、更新与删除操作。
- **附件全面支持**：实时同步图片、视频、音频等各类非设置文件。 > **注意**：需要 v1.0+，服务端 v0.9+。请控制附件文件大小，大文件可能会导致同步延迟。
- **配置同步**：提供配置同步功能，支持多台设备的配置同步, 告别手动给多端设备拷贝配置文件的痛苦。 > **注意**：需要 v1.4+，服务端 v1.0+。目前还在测试阶段，请谨慎使用。
- **服务端版本查看**： 显示服务器的版本信息，方便了解服务器的版本状态。
- **多端同步**：支持 Mac、Windows、Android、iOS 等平台。
- **笔记历史**：提供笔记历史功能，您可以插件端、服务端WebGui，查看笔记的所有历史修改版本， 您可以查看修改详情或者复制历史版本内容。
- **离线设备笔记冲突优化**：对离线设备的笔记修改，增加冲突解决策略，避免因只保留最新更新，导致的笔记内容丢失。
- **版本检测**：提供版本检测功能，你可以快速的获取 插件端/服务端 最新的版本信息，方便快速升级。
- **附件云预览**：提供附件在线预览功能，附件无需同步到本地设备，从而节省本地存储空间。配合插件的排除设置，可对某类附件直接使用第三方资源库(例如 WebDav)而不通过服务端上传。

## 使用教程

### 部署 docker 服务

来源：[GitHub - haierkeys/fast-note-sync-service](https://github.com/haierkeys/fast-note-sync-service)

#### 一、一键脚本（适合服务器/VPS 部署）

自动检测系统环境并完成安装、服务注册。

```
bash <(curl -fsSL https://raw.githubusercontent.com/haierkeys/fast-note-sync-service/master/scripts/quest_install.sh)
```

脚本主要行为：  

自动下载适配当前系统的 Release 二进制文件。  

默认安装至 /opt/fast-note，并在 /usr/local/bin/fast-note 创建快捷指令。  

配置并启动 Systemd 服务 (fast-note.service)，实现开机自启。  

管理命令：`fast-note [install|uninstall|start|stop|status|update|menu]`

#### 二、Docker 部署（适合 NAS 部署）

1. Docker Run

```
# 1. 拉取镜像
docker pull haierkeys/fast-note-sync-service:latest
# 2. 启动容器
docker run -tid --name fast-note-sync-service \
    -p 9000:9000 -p 9001:9001 \
    -v /data/fast-note-sync/storage/:/fast-note-sync/storage/ \
    -v /data/fast-note-sync/config/:/fast-note-sync/config/ \
    haierkeys/fast-note-sync-service:latest
```

1. Docker Compose

```
version: '3'
services:
  fast-note-sync-service:
    image: haierkeys/fast-note-sync-service:latest
    container_name: fast-note-sync-service
    restart: always
    ports:
      - "9000:9000"  # API 端口
      - "9001:9001"  # WebSocket 端口
    volumes:
      - ./storage:/fast-note-sync/storage  # 数据存储
      - ./config:/fast-note-sync/config    # 配置文件
```

#### 三、后端配置

1. 访问管理面板： 在浏览器打开 http://{服务器IP}: 9000
2. 初始化设置： 首次访问需注册账号。首次注册后记得关闭注册功能（系统设置-允许用户注册-取消勾选）
3. 新建仓库：新建一个仓库，作为笔记存储
4. 准备配置信息：在仓库中点击 viewConfig（小齿轮），准备复制配置信息    ![image](https://cdn3.ldstatic.com/original/4X/f/2/7/f2745b0b2873f301a3b0f1d66d158e10b1ea8153.jpeg)

### PC 端安装插件 (二选一)

1. 手动安装: 访问 [Releases · haierkeys/obsidian-fast-note-sync · GitHub](https://github.com/haierkeys/obsidian-fast-note-sync/releases) 下载安装包, 解压到 Obsidian 插件目录下 .obsidian/plugin
2. 使用 BRAT 安装 ( 支持手机安装 ): 在 Obsidian 插件社区市场, 搜索并安装 BRAT 插件, 进入插件设置界面, 点击 Add beta plugin 并粘贴 [GitHub - haierkeys/obsidian-fast-note-sync: Can be privately deployed, focusing on providing Obsidian users with a seamless, distraction-free note synchronization plugin with real-time sync across multiple platforms, supporting Mac, Windows, Android, iOS, and offering multilingual support.可私有化部署，专注为 Obsidian 用户提供无打扰、丝般顺滑、多端实时同步的多平台笔记同步插件。](https://github.com/haierkeys/obsidian-fast-note-sync)

### 手机端安装插件

建议使用 BRAT 插件去安装本插件，不过需要魔法环境

### 插件配置

1. 连接服务端： 打开 Obsidian 插件设置页面，粘贴后端服务器地址（如果有证书，使用 SSL 更佳）
2. 配置服务端：粘贴前面复制的配置信息进行同步    ![image](https://cdn3.ldstatic.com/original/4X/1/4/4/144ec9449692dd8367214d811b054421043e3854.jpeg)
3. 插件配置：针对自己的使用场景，设置对应同步配置    ![image](https://cdn3.ldstatic.com/original/4X/f/e/8/fe8331e6cf2143882261e30079ac209b7fc3eed7.jpeg)
