# 【教程】2026版 小白也能看懂的自建Cloudflare临时邮箱教程（域名邮箱） 

[原帖链接](https://linux.do/t/topic/1666961/1)

**作者：小黄**  
**时间：Aug 22, 2026 12:42 am**  

从 [【教程】小白也能看懂的自建Cloudflare临时邮箱教程（域名邮箱）](https://linux.do/t/topic/316819) 继续
> 你可以通过右边的目录快速定位你想查看的模块~


# 文档如果没有需要修改的错误和需要新补充的，还请各位佬友不要随意改标题和内容！

### 总结佬友们补充的教程（2026年3月5日更新）

有很多热心的佬友出了别的版本的教程，例如这位佬友写了用Claude Code + CLI的形式去部署，其实项目是支持CLI的，只是说对于新手来说有点门槛~但是这位佬友的思路是交给了Claude Code，写的很详细！感兴趣的佬友可以去学习学习！

[https://linux.do/t/topic/1692459](https://linux.do/t/topic/1692459)

### 佬友迭代出来的自动部署脚本（2026年3月21日更新）

更新为使用 CF 提供的 API 完成全自动化部署，但目前 macOS 版本可能有点问题，等周一解决。 无法复现可能出现的 bug，有没有好心人帮忙解决一下。 修复 mac 版本可能出现的 bug

添加新功能，支持配置子域名邮箱。以及可能解决了 macOS 的 bug 问题。

[https://linux.do/t/topic/1783188](https://linux.do/t/topic/1783188)

### 佬友迭代出来的Windwows版自动部署脚本（2026年3月23日更新）

[https://linux.do/t/topic/1801403](https://linux.do/t/topic/1801403)

### 碎碎念

明天就是入站两周年了，之前一直鸽发邮件教程，一直没写，这几天会把教程补全，在明天入站两周年之际，先把这个主要的教程发出来 感谢佬友们的陪伴，两年，学到了很多，无论是技术上的，还是为人方面的，非常感谢 这是一个充满着友善的社区，一个拥有者庞大能量的社区，感谢L站佬友们~

### 文档共建说明

[文档共建](https://linux.do/c/wiki/42)

写在了文档共建话题，意味着各位佬友都可以编辑文档，如果发现错误的地方，佬友们可以直接帮我编辑修复哈当然也可以评论告诉我，我来修！，如果有新增补充的内容，请不要破坏文档结构，可以按文档结构来继续添加，对了，如果做了文档的修改，可以在你修改的部分著名一下，@一下自己，这样就知道是谁修复或者补充的内容啦~

### 写在前面

我记得写这篇教程的时候是2024年11月或者12月的时候，期间帖子多次编辑后，所以忘了是啥时候了，一晃一年多过去了，当初用来写教程邮箱都过期几个月了 ，很显然上次的教程给很多佬友都带来了帮助，这次我打算翻新一下教程，因为Cloudflare的很多页面更新了，而且临时邮箱项目都迭代好多个版本了！这次打算：

- 翻新Cloudflare原本的教程（之后不会再翻新啦，因为都差不多的，这次翻新也是有佬友说找不到UI位置啦）
- 写发送邮件的配置教程，走你：[为Cloudflare临时邮箱项目（cloudflare_temp_email）添加发邮件功能使用 Resend](https://linux.do/t/topic/1667361)
- 写对接TG机器人的教程，走你：[为Cloudflare临时邮箱项目（cloudflare_temp_email）添加Telegram机器人](https://linux.do/t/topic/1667519)
- 建立问题反馈帖（根据我的空闲待定）（TODO）
- 对接LINUX DO 登录

### 注意

此为临时邮箱，如果你会给你的域名一直续费，那么你可以一直用你自己的域名邮箱，如果你只是年抛域名（用了一年直接抛弃）**那么请不要拿它注册重要的平台，因为我知道很多域名二次续费很贵，各位佬友在购买域名的时候，也请看看续费域名的价格，当然如果是年抛，就无所谓了**

### 教程看前说明

目前这篇教程只会分成两部分

- 第一部分，是教你如何把域名托管到Cloudflare并且配置如何用自己的域名接收邮件，转发到自己常用的邮箱
- 第二部分，会教你如何在Cloudflare上部署并配置该项目[GitHub - dreamhunter2333/cloudflare_temp_email: CloudFlare free temp domain email 免费收发 临时域名邮箱 支持附件 IMAP SMTP TelegramBot · GitHub](https://github.com/dreamhunter2333/cloudflare_temp_email)

### 域名购买

域名的购买，你可以在腾讯云、阿里云或者能提供域名注册的云服务商购买，我这里还是用腾讯云，因为我的实名信息在上面已经有了，买起来方便，这次起个域名`quickbox.cloud`，**如果你要长期使用该域名，可要看清楚续费价格了哦**~

我这里就挑便宜的，买个年抛域名来写教程，域名购买我可不教了，佬友实在不会就查查吧！ ![image](https://cdn3.ldstatic.com/original/4X/b/5/c/b5c2674f9a7017df68e6f117c62477fac295c7c7.jpeg)
### 开始前的准备

#### 域名准备

一个域名，我刚买的`quickbox.cloud`

#### Cloudflare账号

如果你没有，就请去赛博大善人这注册一个账号，注册没什么验证的，很方便

  
      ![](https://cdn3.ldstatic.com/original/4X/2/e/d/2ed1eba49329ae2c63932f73202c0441c81ff476.png)

      [cloudflare.com](https://www.cloudflare.com/)
  

  
    ![](https://cdn3.ldstatic.com/optimized/4X/7/b/5/7b5b801e67e3dce6eecb2f776e24ece12ca4499c_2_690x394.png)

### Cloudflare: Build for the agent era

  Welcome to Cloudflare - Powering the next generation of applications

  

  
    
    
  

  

#### cloudflare_temp_email 项目地址

  

      [github.com](https://github.com/dreamhunter2333/cloudflare_temp_email)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/e/6/d/e6d83fc4941b3b412a3f6e886b9df1cbf2edab83_2_690x344.png)

  ### GitHub - dreamhunter2333/cloudflare_temp_email: CloudFlare free temp domain email 免费收发 临时域名邮箱 支持附件...

    CloudFlare free temp domain email 免费收发 临时域名邮箱 支持附件 IMAP SMTP TelegramBot

  

  
    
    
  

  

项目原作者：[@awsl    
  
      ![bili_071](https://cdn3.ldstatic.com/original/3X/9/6/96c5a75226af5cd6a775a52bb10801b92b65260d.png?v=15)](https://linux.do/u/awsl)

项目原文档：[临时邮箱文档](https://temp-mail-docs.awsl.uk/zh/)

**请记得给项目点个star！**

### 第一部分，将域名托管到Cloudflare

第一部分，是教你如何把域名托管到Cloudflare并且配置如何用自己的域名接收邮件，转发到自己常用的邮箱

#### 将域名交给Cloudflare托管（教程以腾讯云为例，其它云服务商大同小异）
> 在这之前，请登录Cloudflare并把你的语言设置成简体中文与教程同步，在右上角


![image](https://cdn3.ldstatic.com/original/4X/1/1/3/113b1cbba11454a7355eabe800db9e524c9c108b.png)
![image](https://cdn3.ldstatic.com/original/4X/5/2/8/5280ba7553d3ea63580f613613ba6795d0761032.png)> 把你自己的域名填上，这里我填的是刚刚买的域名


![image](https://cdn3.ldstatic.com/original/4X/7/7/5/7754b3b3f1385f7c6ee2da0a3f7a6bebff232607.png)
![image](https://cdn3.ldstatic.com/original/4X/1/3/e/13e9d40a1e98c25ab8599284503d7a7a3994adc2.jpeg)
![image](https://cdn3.ldstatic.com/original/4X/3/4/2/3428332bd8bc668d701687b878e6364cb241ee5c.png)
![image](https://cdn3.ldstatic.com/original/4X/4/1/5/4158ead07f04efafab700f30b31d1f0821bd5308.png)> 这里你可以复制，也可以等会来复制，接下来我们要打开云服务商平台了，我这里是以腾讯云为例哈，其它云服务商都差不多的


![image](https://cdn3.ldstatic.com/original/4X/d/2/0/d20bff66b42ce68797ca37214e5f8aee34a8a00f.png)
![image](https://cdn3.ldstatic.com/original/4X/c/6/d/c6d0ec5575c06af5b96ddc68c8f3e70ed4107687.png)
![image](https://cdn3.ldstatic.com/original/4X/a/1/1/a11fa78fdd96bf123c1f89848cd3bb53471caeb5.png)
![image](https://cdn3.ldstatic.com/original/4X/1/c/9/1c95c0dc94f100a0efdf66e44591d86dfe969bb3.png)> 这里就粘贴刚刚Cloudflare给的两个DNS服务器地址


![image](https://cdn3.ldstatic.com/original/4X/f/9/b/f9bda8091297c03db1cc3c15ab17d1da5b7abeb9.png)> 回到Cloudflare点击试试，如果没有反应，那就等十分钟左右~


![image](https://cdn3.ldstatic.com/original/4X/0/7/1/071ba95d5cb8987a5b435d0393b3312aa038a467.png)
![image](https://cdn3.ldstatic.com/original/4X/4/2/a/42a76164babdc819919c851ece1270f42f9dfa87.png)> 如果你跟我一样，那就耐心等等~


![image](https://cdn3.ldstatic.com/original/4X/6/e/8/6e80c7b32119d5f01f89b07eb01506aaa39b9869.png)> 大概过了7~8分钟，如果提示这样，就代表OK啦！


![image](https://cdn3.ldstatic.com/original/4X/1/0/a/10a7247502cdbe2074f235a18f5a649c03fdc9ae.png)
![image](https://cdn3.ldstatic.com/original/4X/c/3/c/c3c3a2e81e2c73d029d44ddbf09e0a43c10bc574.png)
#### 配置电子邮件路由

在此说明，如果你不想部署临时邮箱项目，你只是想拥有一个自己的临时域名邮箱，自己用，只做接收邮件即可的话，那么按接下来的配置完成后，就不用继续往下了，你就可以用你的邮箱去注册你想注册的平台，它会把**邮件转发至你配置的常用邮箱**，举个例子，例如我的域名`quickbox.cloud`：

- xiaohuang@quickbox.cloud
- linuxdo@quickbox.cloud
- bbb@quickbox.cloud
- neo@quickbox.cloud

我的常用邮箱是：xxxx@gmail.com

那么`xiaohuang@quickbox.cloud、linuxdo@quickbox.cloud、bbb@quickbox.cloud、neo@quickbox.cloud`都会转发到`xxxx@gmail.com`

这就意味着，无论前缀是什么`*@quickbox.cloud`**（*号代表所有）**，都会转发到你的常用邮箱，如果你只是想这样，那么按接下来的配置完即可~

![image](https://cdn3.ldstatic.com/original/4X/5/7/b/57b13231bf84b3b3a176dba3084930337e192cf8.png)
![image](https://cdn3.ldstatic.com/original/4X/3/4/9/34963a637b71d9464590261852eee50e73082f5c.png)> 【目标】输入框里请填写你自己的邮箱地址（可以收到邮件的邮箱地址），填写后点击【创建并继续】接下来你会在你的邮箱里收到一封验证邮件，打开地址就可以完成验证了，我这里没截到图！


![image](https://cdn3.ldstatic.com/original/4X/0/7/c/07c759134b4318050d82db9fc7cab678fb9a5a02.png)
![image](https://cdn3.ldstatic.com/original/4X/7/8/7/7870e0dbd6d95da7975a6d4cb567923d1c45808f.png)
![image](https://cdn3.ldstatic.com/original/4X/8/2/a/82ac198ab8d98137e538cada1b038c2a792ffade.png)
![image](https://cdn3.ldstatic.com/original/4X/5/5/c/55c88a4190f8dff8bd025800c4ecbff355d63a0b.png)
![image](https://cdn3.ldstatic.com/original/4X/0/b/0/0b03e5a7917e97efcdd2bdac651ebebad3b6b2fb.png)
![image](https://cdn3.ldstatic.com/original/4X/d/c/4/dc4638add27dced24c09a4a5755b0c2ec676ba74.png)> 至此就完成了第一部分的所有配置，下面是测试


![image](https://cdn3.ldstatic.com/original/4X/a/8/0/a803d5c681aece0dd102da4e857ab80a0a266994.png)
### 第二部分，在Cloudflare上部署 cloudflare_temp_email 项目

为什么能收到邮件了，还要部署`cloudflare_temp_email`项目呢？  

三级帖里好东西太多了，妙处可太多了，这是其一，第二就是你可以分享给你的朋友一起使用~

#### 创建D1数据库
> 回到首页


![image](https://cdn3.ldstatic.com/original/4X/8/f/6/8f6edf26afd5afe562187b12f9a9a5ef24c05c18.png)> 名称自己随便起个就OK


![image](https://cdn3.ldstatic.com/original/4X/a/0/b/a0bcb17355c97a1f106a3f04121778ff76414086.png)> 打开项目的 `db/schema.sql` 文件复制SQL


直达：[cloudflare_temp_email/db/schema.sql at main · dreamhunter2333/cloudflare_temp_email · GitHub](https://github.com/dreamhunter2333/cloudflare_temp_email/blob/main/db/schema.sql)

![image](https://cdn3.ldstatic.com/original/4X/9/5/0/950c62f5411bd8294d36e2da8e916b64b6f91df3.png)> 从Github复制过来后，粘贴到输入框内，点击【执行】


![image](https://cdn3.ldstatic.com/original/4X/b/e/4/be4a2fa77240390fdb33a53f7e88d8a3613a6958.png)> 这样就代表执行OK了！


![image](https://cdn3.ldstatic.com/original/4X/2/a/0/2a09f2cc8433395201aa9b8bc9497ba46a621107.png)> 回到概述刷新一下，如果表数量是10就代表成功~


![image](https://cdn3.ldstatic.com/original/4X/4/f/2/4f2479819606115ba166527f064281d5fa659ee5.png)
#### 配置KV缓存

![image](https://cdn3.ldstatic.com/original/4X/6/0/b/60b6c3057450d93ff27c23ed8315663f54636745.png)> 名字依旧自己起个名字，也可以跟图片里一样哈


![image](https://cdn3.ldstatic.com/original/4X/7/f/a/7fa39c6c43c0e24d7ee5b68f3c428452e0efc2b5.png)> 先创建，我们一会会用的上


#### 创建 Workers 部署 cloudflare_temp_email 后端

![image](https://cdn3.ldstatic.com/original/4X/c/c/b/ccb085b5b33ca5e69aa38ced219e07d3ca5d3638.png)
![image](https://cdn3.ldstatic.com/original/4X/1/6/2/1621d78c4a26dc8e8a6e8fa3f522f834dcbed717.png)> 名字依旧自己随便填哈


![image](https://cdn3.ldstatic.com/original/4X/5/b/3/5b38e2efde74d6f220ec2228bea034a8b3d06363.png)
##### 绑定DB数据库和KV缓存

![image](https://cdn3.ldstatic.com/original/4X/a/d/5/ad5b45ca686bbc99a862e894e3c6c2438f213fee.jpeg)
![image](https://cdn3.ldstatic.com/original/4X/3/6/1/36101c6e0d92a270e1e92a6f8d519e15e4cebd8a.png)> 这里变量名称请一定要填 **DB**可不能自己随意填了哈！


![image](https://cdn3.ldstatic.com/original/4X/2/4/2/2427011ea57c3dc7a65164522c52f75f54e8edef.png)
![image](https://cdn3.ldstatic.com/original/4X/6/9/8/698cc12acb9cd181968a7704d5a2a90c00ec0ad5.png)
![image](https://cdn3.ldstatic.com/original/4X/8/d/e/8deb4c8ffee7b863c7ae142d1c759ae76cdb38d2.png)> 这里变量名称也是一样，请保持**KV**和图片一致！！！


![image](https://cdn3.ldstatic.com/original/4X/4/e/6/4e6b251b0fe0050672cddf8cf65c8a6bcd967723.png)
#### 配置变量参数

我看了有非常非常多的参数，但不是所有参数都要配，我在教程里给出的都建议配置！

**可以一次配置多个，你可以一次性配置好然后点击部署！建议直接复制粘贴！**

![image](https://cdn3.ldstatic.com/original/4X/2/8/3/2831c0ae558180fc818bb4b98834152ce39a5fe2.png)
![image](https://cdn3.ldstatic.com/original/4X/6/2/2/62218ec4aa51c4157897be8513516db3f6f29950.png)
![image](https://cdn3.ldstatic.com/original/4X/b/a/0/ba0f6b07fe749c87599a943d57001b984a97c85f.png)
**类型记得改成教程里说的参数类型，不然可能失效哦！**

#### 以下是参数名和参数示例以及解释

##### DOMAINS

参数类型：JSON

单个域名示例 **推荐**

```
[
    "quickbox【这只是示例记得改成你自己的域名】.cloud"
]
```

多个域名示例

```
[
    "awsl.uk",
    "example.com"
]
```

解释：临时邮箱域名列表，比如我只有一个，我就填一个就行，多个就以JSON数组的方式添多个

---

##### DEFAULT_DOMAINS

参数类型：JSON

留空示例（未登录用户什么都没得用）**推荐**

```
[]
```

给一个域名（未登录用户也可以以这个域名创建邮箱地址）

```
[
    "你的域名.com"
]
```

解释：直接留空，未登录的用户或者无角色的用户可用的域名列表，直接为空就行，如果你想给未登录的用户有域名用的话，就配置域名

---

##### DISABLE_ANONYMOUS_USER_CREATE_EMAIL

参数类型：文本

```
true
```

解释：设为 `true` 后，未登录的匿名用户无法创建邮箱，必须登录才能创建

---

##### JWT_SECRET

参数类型：文本

在线生成一个：[Credentials Generator | LibreChat](https://www.librechat.ai/toolkit/creds_generator)

![image](https://cdn3.ldstatic.com/original/4X/0/5/b/05b32e4ec5fd2617905dad66425e0382206c37b1.png)
```
1c3fa6d797b84a01e65e8a37710f2e0fd5acb91ad993a798544e57161a44f944不要用这个不要用这个，自己去生成，不要直接复制这个！
```

解释：JWT签名密钥，用于生成登录凭证和鉴权

---

##### ADMIN_PASSWORDS

参数类型：JSON

可以多个也可以单个

```
[
    "mypassword123"
]
```

```
[
    "mypassword123",
    "aaaabbbblinuxdo"
]
```

解释：Admin管理后台的登录密码，不配置的话无法登录后台管理

---

##### ENABLE_USER_CREATE_EMAIL

参数类型：文本

```
true
```

解释：是否允许用户创建邮箱地址，不配置默认不允许，两个值`true`、`false`填`true`就行

---

##### ENABLE_USER_DELETE_EMAIL

参数类型：文本

```
false
```

解释：是否允许用户删除邮件消息，我一般都是默认false随自己的想法配置哈

---

##### USER_ROLES

参数类型：JSON

举例，例如我想让vip使用`xxx.love`，admin使用`aaa.love`，这样就可以隔离不同的角色用不同的域名了

```

[
    {
        "domains": [
            "xxx.love"
        ],
        "prefix": "",
        "role": "vip"
    },
    {
        "domains": [
            "aaa.love"
        ],
        "prefix": "",
        "role": "admin"
    }
]

```

不过在我们这肯定都是一样的，我们没有配置多个域名，**推荐**

```

[
    {
        "domains": [
            "quickbox.cloud"
        ],
        "prefix": "",
        "role": "vip"
    },
    {
        "domains": [
            "quickbox.cloud"
        ],
        "prefix": "",
        "role": "admin"
    }
]
```

解释：配置用户的角色，及角色可以使用的域名列表

---

##### ADMIN_USER_ROLE

参数类型；文本

```
admin
```

解释：可访问`admin`管理后台的角色名，也就是说用户被赋予这个角色名后，登录就有了管理后台的权限

---

##### ENABLE_AUTO_REPLY

参数类型：文本

```
false
```

解释：否允许自动回复邮件，这个直接false就行

---
> 请按我上面给的参数配置好！我给出来的请都配好！


![image](https://cdn3.ldstatic.com/original/4X/4/5/7/4577d57f7ce38ce2726d1d5af81cf32624405fea.png)> 配置好后一共是10个参数，数数哈


#### 部署代码文件
> 先配置一下兼容性标志


![image](https://cdn3.ldstatic.com/original/4X/3/8/6/386bcf24b4f130e17dcd6638600a24d2fb2cdfa3.png)
![image](https://cdn3.ldstatic.com/original/4X/2/c/9/2c99eb8ce142edd3f3e768fe5267187112b7e022.png)> 直接在这里复制粘贴过去哈


```
nodejs_compat
```
> 从 Github 的 Releases 中下载最新版本的 Worker.js


**记得给作者点star！**

项目 Releases 直达：[Releases · dreamhunter2333/cloudflare_temp_email · GitHub](https://github.com/dreamhunter2333/cloudflare_temp_email/releases)

![image](https://cdn3.ldstatic.com/original/4X/5/1/9/519d38d9e2b9d5b81b0d8ff680841fe3ed871ae0.png)
![image](https://cdn3.ldstatic.com/original/4X/c/2/f/c2fb501448d5cf4c3a201b0bf769027a89ce2ffc.png)> 直接往里拖


![image](https://cdn3.ldstatic.com/original/4X/e/8/e/e8e19a3997d0904a912b66d9231c4e4064c73d8f.jpeg)
![image](https://cdn3.ldstatic.com/original/4X/d/c/4/dc4edec6ef4913fbf975b9e6d2c39b15140b04c3.png)> 或者如果拖进去失败的话，你可以打开代码文件，直接全部复制，粘贴过来


![image](https://cdn3.ldstatic.com/original/4X/3/b/7/3b738635d79400d1794e0e3d01b2931bad721e14.jpeg)
![image](https://cdn3.ldstatic.com/original/4X/6/4/9/6490d266dd30a5947945b55cf5db402d23165349.jpeg)
![image](https://cdn3.ldstatic.com/original/4X/b/5/2/b52ddefb539c44d95adfa4ef274ad66712812136.jpeg)> 如果刷新后是`ok`，恭喜你，后端部署成功，但还没有结束，还需要配置自定义域！


![image](https://cdn3.ldstatic.com/original/4X/7/1/7/71791d0dc011b4558df16b1e3a9aa0be9277529d.jpeg)
#### 配置自定义域

![image](https://cdn3.ldstatic.com/original/4X/3/a/d/3ad35b737383a7ab168cc5c8292adb8cf360b632.png)> 你可以直接按我给的来，当然域名改成你自己的！


```
apimail.你的域名.com
```

![image](https://cdn3.ldstatic.com/original/4X/8/0/0/800b9f435f59610e490264e727f61dc6712ce80d.png)> 如果访问没有显示`ok`，那就晚一点再去试试！


![image](https://cdn3.ldstatic.com/original/4X/5/c/8/5c86ad195d8b74708f7b6528b215cd33717b9a94.png)
#### 重新配置域名电子路由

**如果你之前配置了`Catch-all`到你的常用邮箱的话，这一步是必定要配置的！否则你的域名邮箱站点是收不到任何邮件的！**
> 回到首页


![image](https://cdn3.ldstatic.com/original/4X/2/1/e/21e24a3da3ef9e1495e12a175d7dd1d09366db00.png)
![image](https://cdn3.ldstatic.com/original/4X/2/e/3/2e35b0454f14e0fb5776a8baab7b25be90b5c8b8.png)
![image](https://cdn3.ldstatic.com/original/4X/6/9/f/69f0b2dbb350802c43ea80e3d00e4ed4d2f1a661.png)
![image](https://cdn3.ldstatic.com/original/4X/a/a/a/aaa3bdfdb890f4164df2cf90f991c514d68e99da.png)
#### 部署前端页面

我们在上一步不是配置了自定义域吗？这样我们就可以通过自己的域名访问到后端接口了，而不是Cloudflare提供的域名，以我的为例，配置的是`apimail.quickbox.cloud`

那么我们接下来生成前端代码的要填的就是

```
https://apimail.quickbox.cloud 这是我的后端地址！请填自己的后端地址，并且记得用https！不要复制用我的！用你自己搭建的！
```

生成前端页面代码地址：[Cloudflare Pages 前端 | 临时邮箱文档](https://temp-mail-docs.awsl.uk/zh/guide/ui/pages)

![image](https://cdn3.ldstatic.com/original/4X/8/d/e/8dedbb63dd87843a85fb54d579c69b5005befd1d.jpeg)
![image](https://cdn3.ldstatic.com/original/4X/9/d/d/9dd71ba177514f48dd789330cce631c258954b73.png)> 回到Cloudflare首页


![image](https://cdn3.ldstatic.com/original/4X/c/b/b/cbbb8cb9ea391c843e550a8ab529b67cfb633d62.png)
![image](https://cdn3.ldstatic.com/original/4X/5/6/3/563d169d6318c1e082d88aa35d17458d0c822145.png)> 把刚刚下载的前端代码文件拖进去


![image](https://cdn3.ldstatic.com/original/4X/f/9/8/f985650a75ec0528dcc0228b709e2e437a962e1f.jpeg)> 这里的未找到处理改成`single-page-application`，这样刷新就不会出现404了！一定要配！名字记得随便给个~


![image](https://cdn3.ldstatic.com/original/4X/2/4/6/24697107a8c2442e9fb32dc679a9aba1e1881896.png)
![image](https://cdn3.ldstatic.com/original/4X/3/8/d/38d27ab30c23b076491a1d8b8f08ac7281208ca7.png)
![image](https://cdn3.ldstatic.com/original/4X/7/1/d/71dea69077c0dfa68d818070fd6b6b0af81cef1b.png)> 老样子添加自定义域


![image](https://cdn3.ldstatic.com/original/4X/7/1/3/7130f5f190238d33aa68bb3f77944e2cdc2fe343.png)
![image](https://cdn3.ldstatic.com/original/4X/e/6/1/e616b3f5857a04549c84091f25305ec096bc900b.png)
![image](https://cdn3.ldstatic.com/original/4X/1/3/e/13e681d8434868462e1c00efb21e2d092f35ee4e.png)
```
mail.你的域名.com
```
> 试了下要单击5次了！ ![image](https://cdn3.ldstatic.com/original/4X/3/5/e/35e2f01ea6e7115a688bcc5c783113faf4fb74d2.png)

![image](https://cdn3.ldstatic.com/original/4X/8/5/c/85c8231a3bdff425bda0e80309213a1343663017.png)> 如果成功登录，那么恭喜你！部署成功！


![image](https://cdn3.ldstatic.com/original/4X/c/a/9/ca9b2e8f95475d291f5ab01c82fad9313bfe1a35.png)
### 基本的使用

#### 给你那不成气的兄弟或者姐妹建一个用户给TA用！

![image](https://cdn3.ldstatic.com/original/4X/d/e/d/dede7576d5bae7d9fc4aa4d54b860c02460fb87d.png)
![image](https://cdn3.ldstatic.com/original/4X/4/d/9/4d94eaf72643e7e2e8cfbbb548e5e96d394c1d14.png)
![image](https://cdn3.ldstatic.com/original/4X/5/3/d/53dbf0a2bc77a6243c7591bdfacb4d572c5e27e3.png)
![image](https://cdn3.ldstatic.com/original/4X/c/e/6/ce6daf02174ada87dfa56e8714bce725417d9202.png)
![image](https://cdn3.ldstatic.com/original/4X/e/b/6/eb638a6726478f70aae778359f3306a5285a743f.png)
![image](https://cdn3.ldstatic.com/original/4X/8/6/1/8612853d6da77dadc1386105d8a73152c65b979a.png)> 之后把前端地址发给TA，然后给他登录邮箱和密码登录就可以创建邮箱了


![image](https://cdn3.ldstatic.com/original/4X/0/1/e/01ea7376a9a2465aba3eeeed56cf4f9ad79b72f0.png)
![image](https://cdn3.ldstatic.com/original/4X/e/9/0/e90f7c6620ea1c368edf0fe8572d756831c6e69a.png)
![image](https://cdn3.ldstatic.com/original/4X/4/f/9/4f900a95aa7ac4e8bb7887470992c2a9deec441e.png)
![image](https://cdn3.ldstatic.com/original/4X/e/3/f/e3fee52e2acd2787c49f57def03ce34303871a27.png)> 至此就可以用这个邮箱地址去使用了~


![image](https://cdn3.ldstatic.com/original/4X/2/1/0/2102d13fa8aab36f8cd0948c70a70b2a159f987e.png)
![image](https://cdn3.ldstatic.com/original/4X/b/b/f/bbf3b301c35d3f2b5e1f175b52db352ff1734100.png)
### 对接LINUX DO Oauth2登录给可爱的佬友们用！

在配置之前，我们需要在后端增加一个新的变量`USER_DEFAULT_ROLE`

我尝试配置`USER_DEFAULT_ROLE`发现没有效果，很奇怪，所以我们还是改`DEFAULT_DOMAINS`和`DISABLE_ANONYMOUS_USER_CREATE_EMAIL`吧

回顾一下参数作用

#### DEFAULT_DOMAINS

参数类型：JSON

改成给佬友们使用的域名列表

```
[
    "quickbox.cloud"
]
```

解释：这里一般是留空的，因为我没打算把自己的域名给未登录的用户使用，这里是对接LINUX DO Outh2登录，所以添加了域名，这样通过LINUX DO登录后就可以使用这个域名创建邮箱了~

---

#### DISABLE_ANONYMOUS_USER_CREATE_EMAIL

参数类型；文本

**这个还是保持true，如果你之前改的是true那这里就不用调整这个参数了！**

```
true
```

解释：设为 `true` 后，未登录的匿名用户无法创建邮箱，必须登录才能创建

---

**这样效果就达成了，佬友们登录后有域名用，然后未登录的用户还是没有域名用！只不过在后台管理新建的用户没指定角色也有域名用了，所以自己看着配置去就行~**

我这里不贴怎么配置参数了啊，上面都有！不会了就翻上去看！**注意是后端**！

也就是我们之前所说的`mailapi`

![image](https://cdn3.ldstatic.com/original/4X/9/6/f/96f07cfc7b58ec5fde302283e80e8a33c364a6c8.png)

---

#### 开始配置Oauth2

![image](https://cdn3.ldstatic.com/original/4X/f/a/3/fa39eeb59360c93877e7262f3b9dfcfc7302688f.png)
![image](https://cdn3.ldstatic.com/original/4X/5/d/5/5d5f9f18f6c8f71ca597d5227ed954788e93c251.png)> 接下来打开LINUX DO的connect


[https://connect.linux.do/](https://connect.linux.do/)

![image](https://cdn3.ldstatic.com/original/4X/c/2/2/c2201649af84796b5c4ea876ec1036d9f26cab48.png)
![image](https://cdn3.ldstatic.com/original/4X/b/e/1/be1051a41d9b95f19ff63b32b7b902764e463500.png)> 回调地址看以下操作，我这里等级给了1级，也就是说1级及以上的佬友都可以用


![image](https://cdn3.ldstatic.com/original/4X/5/3/b/53b793482404442748d8b661483bca7474bbf9dd.png)
![image](https://cdn3.ldstatic.com/original/4X/c/d/8/cd8dcbbe11e33ef70d5145ee167a59257dbf0910.png)
![image](https://cdn3.ldstatic.com/original/4X/7/8/d/78d19ddfbaa163d0c26bf0b92d45f71c0f628c48.png)> 回填`Client ID`和`Client Secret`


![image](https://cdn3.ldstatic.com/original/4X/1/2/5/1250963cd37ff815a23c53a1e3abece582b7e5cd.png)
![image](https://cdn3.ldstatic.com/original/4X/9/9/e/99ed16013a3de5b6c849b0e034fa90fe1982ab52.png)> 测试是否成功


![image](https://cdn3.ldstatic.com/original/4X/4/f/b/4fb09583d93511a40aa57dd8540f6dd2d234e04a.png)
![image](https://cdn3.ldstatic.com/original/4X/6/d/0/6d05935c00d3993080d66a6929e3dab220c31fb5.png)
![image](https://cdn3.ldstatic.com/original/4X/7/a/a/7aa88828536901e48f0f136b946355eca949a574.png)> 未登录的用户无法使用


![image](https://cdn3.ldstatic.com/original/4X/4/a/4/4a40a45cafe7588aef27a7bbfb974bdddf4c9cc4.png)
### 教程结束

又是从下午13点30开始，写到晚上20点38分，比自己做别的项目开发还费时间 所以佬友们，目前是搭建好了，也接了Outh2登录，你们可以使用这个临时邮箱服务，但请不要拿来注册奇怪的平台！危险的平台！我在后台都能看到！ 其它的教程，最晚是后天补齐，还请佬友们耐心等待，会单独开新帖~

## 也不要拿来注册重要的平台，这是临时邮箱（最近可能停掉）

地址：[https://mail.quickbox.cloud/](https://mail.quickbox.cloud/) **即将停用！**

    
  
    
    ![](https://cdn.ldstatic.com/user_avatar/linux.do/xiaohuang/48/338864_2.png)
    
      [关于临时邮箱域名quickbox.cloud停用说明](https://linux.do/t/topic/2163435) [搞七捻三](https://linux.do/c/gossip/11)
    
  

  > 写在前面> 原因是 [@yorecx](https://linux.do/u/yorecx) 这位佬友在临时邮箱教程中[【教程】2026版 小白也能看懂的自建Cloudflare临时邮箱教程（域名邮箱）](https://linux.do/t/topic/1666961) 用了我提供给佬友们用的临时域名注册了重要的账户，于是在评论区找到了我，评论如下： 
> 处理方案
> 竟然如此，我打算直接把这个域名（quickbox.cloud）转给这位佬友，但不是马上就一刀切了，在此告知使用该域名邮箱的佬友们： 
> 域名将在下个月6月12日…


### 搬运说明
> 如果觉得教程不错，可以进行搬运，但请标明教程的出处！否则就是私自搬运！这是可耻的！


- 之前第一个帖子刚发没多久就被搬运到CSDN了…

# 请不要在文档里写无关教程的内容！
