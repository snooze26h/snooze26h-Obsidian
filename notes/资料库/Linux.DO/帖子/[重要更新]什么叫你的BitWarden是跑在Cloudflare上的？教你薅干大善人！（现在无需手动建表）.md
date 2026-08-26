# [重要更新]什么叫你的BitWarden是跑在Cloudflare上的？教你薅干大善人！（现在无需手动建表） 

[原帖链接](https://linux.do/t/topic/1232950/1)

**作者：Sora**  
**时间：Nov 28, 2025 1:36 am**  

# 前言

一直见有佬友在纠结密码管理器应该跑在哪。是用官方来的稳定，还是自建来的可控，亦或是各种托管平台上来的廉价（免费）？那如果我说有一个既稳定、又免费的自建方案呢？ 把Rust编译成WASM放浏览器里跑，想必各位佬已经非常熟悉了，其实它也可以放到Worker里跑的。哎你说巧不巧，服务端有个现成的用Rust写的[vaultwarden](https://github.com/dani-garcia/vaultwarden)，bitwarden的绝大多数功能都是可以整成无状态的，Cloudflare还正好提供了D1数据库，这还能不薅它？  

虽说Cloudflare前几天才崩过，但CF都崩了，还有几个网站登录得上呢 … 当然这么牛的思路肯定不是我能想出来的，也不是我找到的（看了二叉树树的视频才知道有这么个神必项目），但这并不妨碍我化身为API对齐的牛马，目前是打算长期维护基础功能的。

[原项目](https://github.com/deep-gaurav/warden-worker)的端点实现缺的有点多，已经不兼容新版客户端了，批量导入也有点问题，而且好像作者也没有要更新的意思，我这里就放上我二开的版本了。

  

      [github.com](https://github.com/qaz741wsd856/warden-worker)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/3/7/e/37e053643cc7058d7e40b7e0e16e2e1d8e2badda_2_690x344.png)

  ### GitHub - qaz741wsd856/warden-worker: A Bitwarden-compatible server for Cloudflare...

    A Bitwarden-compatible server for Cloudflare Workers

  

  
    
    
  

  

**友情提醒**: 本项目不支持ws，因此用不了send功能。目前只实现基础功能，暂不考虑网页前端。网页版已通过 [Static Assets](https://developers.cloudflare.com/workers/static-assets/) 的方式集成了 [bw_web_builds](https://github.com/dani-garcia/bw_web_builds) ，静态资源不消耗Worker额度，感谢 [rainisa](https://linux.do/u/rainisa)佬的建议，现已经不再需要单独部署Page了。

---

项目已更新，详见：

    
  

![](https://linux.do/user_avatar/linux.do/qaz741wsd856/48/312935_2.png) qaz741wsd856:

[](https://linux.do/t/topic/1232950/50)
  > 重要安全更新

---

# 简易部署教程

该项目的部署方法与二叉树树的教程一致: [你可曾想过，直接将BitWarden部署到Cloudflare Worker？](https://2x.nz/posts/warden-worker)  

因此我在此只做简述并强调关键步骤，详细图文流程请参考上面的博客，**记得要fork我的项目**就行。

## 前期准备

你需要准备：

- 一个 Cloudflare 账户
- 一个 Github 账户
- 一个没有被阻断的域名

## 第一步：设置 Github 项目

首先fork[我的仓库](https://github.com/qaz741wsd856/warden-worker)到你的账户（要是能顺手点个星星就好啦），进入Action页面，启用Action。  

点击 `settings` - `Secrets and variables` - `Actions`，准备添加 `Repository secrets`，接下来我们一共要添加三个secrets。

然后进入 Cloudflare 控制台，[https://dash.cloudflare.com/](https://dash.cloudflare.com/)  

在浏览器地址栏中复制你的account id  

![image](https://cdn3.ldstatic.com/original/4X/a/6/5/a65d50c53bd48ba1bae671f8db1cb262803ad594.png)

回到github页面，点击`New repository secret`，名称写`CLOUDFLARE_ACCOUNT_ID`，值就写刚才复制的id。

接着访问[https://dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens)  

选择编辑Worker的模板，然后给它再**添加D1的编辑权限**，再创建api令牌，回到github页面添加为`CLOUDFLARE_API_TOKEN​`。

最后去Cloudflare控制台的`存储与数据库` - `D1 sql 数据库`，创建一个数据库，并复制它的id。

![image](https://cdn3.ldstatic.com/original/4X/a/f/e/afe1d50c6b4d1c060f85e86ffe028326d84773ce.png)  

回到github，添加为`D1_DATABASE_ID`。
### 附件支持（可选）

本项目的附件功能依赖R2，虽然Cloudflare提供了慷慨的免费额度，但仍可能产生非预期的扣费，参考[Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/)，因此该功能默认关闭。

如果你想启用R2支持，需要先去Cloudflare控制台创建一个R2储存桶，记下它的名称。

在github仓库中添加一个名为`R2_NAME`的secret，值就填刚才创建的桶名称。

---

至此在github上的配置就完毕了，点击上方的`Actions`选项卡，选到Build工作流点运行即可。

## 第二步：建表

在Action运行的时候也不用干等着，可以先去CF控制台里把表建了。

打开刚才创建的数据库，点`Explore Data`，然后在Query窗口中**分别**粘贴并运行以下sql建表语句(定义在[sql/schema.sql](https://github.com/qaz741wsd856/warden-worker/blob/main/sql/schema.sql)中)，一次只执行一条：

```
CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY NOT NULL,
    name TEXT,
    avatar_color TEXT,
    email TEXT NOT NULL UNIQUE,
    email_verified BOOLEAN NOT NULL DEFAULT 0,
    master_password_hash TEXT NOT NULL,
    master_password_hint TEXT,
    password_salt TEXT, -- Salt for server-side PBKDF2 hashing (NULL for legacy users pending migration)
    key TEXT NOT NULL, -- The encrypted symmetric key
    private_key TEXT NOT NULL, -- encrypted asymmetric private_key
    public_key TEXT NOT NULL, -- asymmetric public_key
    kdf_type INTEGER NOT NULL DEFAULT 0, -- 0 for PBKDF2, 1 for Argon2id
    kdf_iterations INTEGER NOT NULL DEFAULT 600000,
    kdf_memory INTEGER, -- Argon2 memory parameter in MB (15-1024), NULL for PBKDF2
    kdf_parallelism INTEGER, -- Argon2 parallelism parameter (1-16), NULL for PBKDF2
    security_stamp TEXT,
    totp_recover TEXT, -- Recovery code for 2FA
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

```
CREATE TABLE IF NOT EXISTS ciphers (
    id TEXT PRIMARY KEY NOT NULL,
    user_id TEXT,
    organization_id TEXT,
    type INTEGER NOT NULL,
    data TEXT NOT NULL, -- JSON blob of all encrypted fields (name, notes, login, etc.)
    favorite BOOLEAN NOT NULL DEFAULT 0,
    folder_id TEXT,
    deleted_at TEXT,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (folder_id) REFERENCES folders(id) ON DELETE SET NULL
);
```

```
CREATE TABLE IF NOT EXISTS folders (
    id TEXT PRIMARY KEY NOT NULL,
    user_id TEXT NOT NULL,
    name TEXT NOT NULL, -- Encrypted folder name
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

```
CREATE TABLE IF NOT EXISTS attachments (
    id TEXT PRIMARY KEY NOT NULL,
    cipher_id TEXT NOT NULL,
    file_name TEXT NOT NULL,
    file_size INTEGER NOT NULL,
    akey TEXT,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    organization_id TEXT,
    FOREIGN KEY (cipher_id) REFERENCES ciphers(id) ON DELETE CASCADE
);
CREATE INDEX IF NOT EXISTS idx_attachments_cipher ON attachments(cipher_id);
```

```
CREATE TABLE IF NOT EXISTS attachments_pending (
    id TEXT PRIMARY KEY NOT NULL,
    cipher_id TEXT NOT NULL,
    file_name TEXT NOT NULL,
    file_size INTEGER NOT NULL,
    akey TEXT,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    organization_id TEXT,
    FOREIGN KEY (cipher_id) REFERENCES ciphers(id) ON DELETE CASCADE
);
CREATE INDEX IF NOT EXISTS idx_attachments_pending_cipher ON attachments_pending(cipher_id);
CREATE INDEX IF NOT EXISTS idx_attachments_pending_created_at ON attachments_pending(created_at);
```

```
CREATE TABLE IF NOT EXISTS twofactor (
    uuid TEXT PRIMARY KEY NOT NULL,
    user_uuid TEXT NOT NULL,
    atype INTEGER NOT NULL,
    enabled INTEGER NOT NULL DEFAULT 1,
    data TEXT NOT NULL, -- JSON data specific to the 2FA type (e.g., TOTP secret)
    last_used INTEGER NOT NULL DEFAULT 0, -- Unix timestamp or TOTP time step
    FOREIGN KEY (user_uuid) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE(user_uuid, atype)
);
```

## 第三步：配置环境变量与自定义域名

当github的Action执行完毕后，你就能在CF的Workers中找到刚才创建的` warden-worker`了，进入这个Worker的设置页面，在`变量和机密`栏添加以下三个密钥：

| 名称 | 说明 |
| --- | --- |
| ALLOWED_EMAILS | 允许注册的完整邮箱，用,分隔（例：your-name@example.com） |
| JWT_SECRET | 随机长字符串 |
| JWT_REFRESH_SECRET | 随机长字符串 |

**可选环境变量**

- `IMPORT_BATCH_SIZE`：用于控制导入密码库时，每次批操作的数据条数，默认为30，设为0代表一次批操作导入所有数据（不推荐）；如果你需要导入的库特别大，可以适当调大这个值。
- `TRASH_AUTO_DELETE_DAYS`: 配置回收站中的项目多少天后会被清理，默认为30，设为0代表永不清理。
- `DISABLE_USER_REGISTRATION`: 用于控制是否表明服务器不支持注册，默认为true，设为false可以让客户端显示注册按钮；该选项不影响实际注册功能。
- `AUTHENTICATOR_DISABLE_TIME_DRIFT`: 控制是否允许TOTP的时间±1偏移，默认为true。
- `ATTACHMENT_MAX_BYTES`: 单个附件的大小限制，单位字节，默认不限制，填`104857600`则代表100MB。
- `ATTACHMENT_TOTAL_LIMIT_KB`: 单个用户的总附件大小限制，单位KB，默认不限制，填`1048576`代表1GB。
- `ATTACHMENT_TTL_SECS`: 附件的上传下载链接的有效时间，默认五分钟。

然后在`域和路由`中添加一个路由，区域选择你的域名，路由写你分给这个Worker的子域名 + `/*`（如`bitwarden.example.com/*`）。

最后别忘记去给这个子域名添加一条dns记录。  

如果你想优选IP，那就不开小黄云，把它CNAME到一个优选过的域名。  

要是不想优选，那就打开小黄云，目标随便填。
> 建议去域名配置页的 `安全性` - `安全规则` 里给 `/identity/*` 和 `/api/accounts/*` 添加速率限制规则。


## 第四步：创建账户并导入数据

这步其实没啥好说的了，如果已有bitwarden库，在电脑上把它导出为JSON即可。

由于没有前端，只能在手机APP上创建账户，然后在电脑上导入即可。 现在可以了

**注意**: 创建账户时用的密码必须记住，没有任何方式能找回。

---

以下开始为我自己加的可选步骤，建议整一下。

## 第五步：配置数据库同步S3

我这里是整了一个自动导出D1并上传到S3的Action，虽然也能用私有仓库备份，但有点怕判定为滥用，所以还是创一个**私有的**S3吧。

**注意**：

- 千万别像我一样傻fufu的用跟Worker同一个Cloudflare账户的R2来备份，不然你号一被封就全没了；
- 千万别用登录或者获取密钥需要依赖Bitwarden的账户，你得自己记着，不然就套娃了；它的恢复邮箱的密码你也得记着。

如果没什么特殊需求，可以试试 [backblaze](https://www.backblaze.com/)，虽然额度非常少但用来备份也够了，胜在完全免费不绑卡。  

此处略过注册账户和创建桶的操作。
> 请确保你之前添加的 `CLOUDFLARE_API_TOKEN` 至少有D1的读取权限。


回到你fork的github仓库，继续添加以下secrets:

| Secret | Required | Description |
| --- | --- | --- |
| S3_ACCESS_KEY_ID | yes | S3 access key ID |
| S3_SECRET_ACCESS_KEY | yes | S3 secret access key |
| S3_BUCKET | yes | 桶名称 |
| S3_REGION | yes | S3 区域，有就正常填，不知道就直接填auto |
| S3_ENDPOINT | no | 用AWS就不用填，其他填https://your-s3-domain.com的形式，带协议不带路径 |
| BACKUP_ENCRYPTION_KEY | no | 额外加密密钥，不填就不加密，填了就一定要记住 |

然后在Action页面中选到`Backup D1 Database to S3`，手动触发一次，等待它运行完成，然后检查你的S3中是否有备份文件。成功后，每天都会自动备份一次，每个备份默认保存30天。

至于恢复操作还是请各位到时去看readme吧，希望这辈子不会用到。

---

### 额外说明

其实 Cloudflare D1 提供了回滚到30天内的任意时刻的功能：[Cloudflare D1 Time Travel documentation](https://developers.cloudflare.com/d1/reference/time-travel/)

```
# 查看当前bookmark
# DATABASE_NAME换成你在Cloudflare控制台里设置的D1名称
wrangler d1 time-travel info DATABASE_NAME

# 回退到指定时间 (ISO 8601)
wrangler d1 time-travel restore DATABASE_NAME --timestamp=2024-01-15T12:00:00Z
    
# 回退到指定bookmark
wrangler d1 time-travel restore DATABASE_NAME --bookmark=<bookmark_id>
```

通过这个操作，就算库出了问题也能快速回滚，再配上S3备份，是不是感觉这一通操作不那么灵车了？

---

# 其他说明

本项目不提供设备管理功能，但其相关API均提供了空实现，目前用着没啥问题。如果真有客户端登录完还会拉一遍设备列表确认自己在不在里面的话，请提个issue。

# TOTP功能投票

我让AI帮我搓了一个totp，我正常用着好像是没啥问题，但我对这玩意的标准和实现完全不了解，也不知道对不对，暂时放在[`totp`](https://github.com/qaz741wsd856/warden-worker/tree/totp)分支下了。至少目前的实现中，要想将移除totp是很容易的，删掉totp表里的数据就行。各位佬友对这个功能有需求么？

  
    
      
            
  - Votes

  
      - 61%           有，大不了我自己来review一下
- 39%           无，纯粹自己防自己

  

      
  
  
  
    
      31
      voters
    
  

  

    
    
  Display tally

  
  

---

看来佬友们对TOTP的呼声很高啊，TOTP功能已在main分支中支持。
