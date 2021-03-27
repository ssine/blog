---
title: "自建聊天服务器、机器人与多平台桥接"
date: 2021-03-27T00:36:39+08:00
---

# 前言

利用开源解决方案自建聊天服务器，端到端加密，没有恼人的 EULA，自己控制数据，可以和其他的服务器组成联邦，甚至还能通过桥接与其它聊天软件的个人和群组聊天——看起来十分的完美。 当然，整个系统需要一点时间来搭建，我一开始只是想做一个稳定的聊天机器人，后来才发现自建聊天服务器带来的的更多便利。

![img](result.png)

本文是在 Docker 的层面上手动搭建的，对于细节控制或是扩展性要求不高的同学可以直接参考 [Ansible Playbook](https://github.com/spantaleev/matrix-docker-ansible-deploy) 一键部署。

# Matrix 与 Synapse

首先解释一下这两个关键概念：

-   [Matrix](https://matrix.org/) 是一个及时聊天的协议，定义了 Matrix 服务器与服务器 ( federation ) 、服务器与客户端之间的信息传递方式。
-   [Synapse](https://github.com/matrix-org/synapse) 是对 Matrix 协议中服务器端的官方实现。

<center>
<img src="matrix-architecture.png" width=500px />
</center>

如上图 ( [来源](https://arxiv.org/pdf/1910.06295.pdf) ) 所示，每个 Homeserver 有自己的域名 ( xxx.com ) ，每个 Matrix 用户属于一个 Homeserver，因此有一个唯一的 ID `@user:xxx.com` ，不同的 Homeserver 之间可以互相沟通信息，使得所有用户之间可以互相交流。 Matrix 没有区分私聊和群聊，所有聊天都是在聊天室 Room 中进行的，私聊就是只有两个人的聊天室。 Room 也是属于某个 Server `#room:xxx.com` 的，不过 Homeserver 负责维护所有所属 User 所在的 Room，也要从别的 Homeserver 同步 Room 的信息过来。

对于个人来说，搭建一个 Homeserver ( 对应一个 Synapse 实例 ) 就足以满足日常的聊天等需求了。

整个目录结构如下：

```text
.
├── admin.sh
├── bridge-discord.sh
├── bridge-telegram.sh
├── bridge-wechat.sh
├── bridge-whatsapp.sh
├── data
│   ├── bridges
│   │   ├── discord
│   │   │   ├── config.yaml
│   │   │   ├── database.db
│   │   │   └── discord-registration.yaml
│   │   ├── telegram
│   │   │   ├── config.yaml
│   │   │   ├── mautrix-telegram.db
│   │   │   └── registration.yaml
│   │   ├── wechat
│   │   │   ├── config.yaml
│   │   │   ├── package-lock.json
│   │   │   ├── room-store.db
│   │   │   ├── user-store.db
│   │   │   └── wechaty-registration.yaml
│   │   └── whatsapp
│   │       ├── config.yaml
│   │       ├── mautrix-whatsapp.db
│   │       └── registration.yaml
│   ├── homeserver.db
│   ├── homeserver.yaml
│   ├── <domain>.log.config
│   ├── <domain>.signing.key
│   ├── media_store
│   └── synapse.log
├── gen.sh
└── run.sh
```

上面的 sh 文件都是 docker 的启动命令，data 文件夹包含了所有容器的数据，mount 在 synapse 容器里，data/bridges/xxx 是对应的桥接容器 mount 数据的地方。

首先要给 Homeserver 准备一个域名，假设是 `matrix.qq.com` ，并且 data 目录路径为 `/home/user/data` 。 那么首先生成配置文件：

```bash
docker run -it --rm \
  -v /home/user/data:/data \
  -e SYNAPSE_SERVER_NAME=matrix.qq.com \
  -e SYNAPSE_REPORT_STATS=yes \
  matrixdotorg/synapse:latest generate
```

之后会生成一个 `homeserver.yaml` 配置文件，需要修改的地方 ( 行数不固定，请自行搜索 ) ：

```yaml
server_name: "matrix.qq.com"
enable_registration: true
```

为了让桥接用的容器和 synapse 容器能够互相通讯，建立一个 docker network：

```bash
docker network create matrix
```

之后启动服务器并连接到 network ( 注意端口映射 ) ：

```bash
docker run -d --name synapse \
  -v /home/user/data:/data \
  -p 80:8008 \
  --restart unless-stopped \
  --network matrix \
  matrixdotorg/synapse:latest
```

，如果一切正常的话，直接访问 matrix.qq.com 就能看到 Synapse 成功启动的页面，可以在 Web 客户端 https://app.element.io/ 上注册一个账号了。Element 的其它客户端也比较好用。

# 聊天机器人

Matrix 自带 Bot API，用 SDK 可以更方便的开发。这里用了比较熟悉的 Typescript，详细信息参考 [官方教程](https://matrix.org/docs/guides/usage-of-matrix-bot-sdk) ，简单来说就是先通过 Web 端给 bot 注册一个账号，获取 token，然后调 SDK 就好。 小 Tip：可以用 ts-node 来直接运行 typescript 文件，体验更佳。

跟 qq 微信不同，这里的一切都是设计好的，不需要 Hack，因此很轻松就可以跑起来。 重点反而是用 bot 来做什么，因此也聊一聊我曾经用聊天机器人做了什么，作为 use case 的一些参考，这部分为了更好的阅读体验放在附录了。

# 多平台桥接

自建 Matrix 服务器的另一个精彩用例是桥接其它聊天软件，这样平时不需要开很多软件就能维持 connection，要是对 GFW 外有接触的话，聊天软件还是蛮多的，比如 Discord、Telegram、Whatsapp 等。比较完整的列表可以在 [官网](https://matrix.org/bridges/) 看到，不再赘述。下面用最简单的语言介绍下各个 bridge 的搭建流程。

## Discord

Discord 有两种 bridge 方式，一个是创建一个 bot 账号，再把 bot 拉到需要同步的 server，这样说话也是通过 bot 来说的；另一个是直接使用用户的 token，以用户的身份进行操作，这样体验更好，不需要拉 bot，不过风险大一些。 下面采用第二种方案， [官方文档](https://github.com/matrix-discord/mx-puppet-discord) 。

首先把 [样例配置](https://github.com/matrix-discord/mx-puppet-discord/blob/master/sample.config.yaml) 拷贝到本地，然后编辑里面必要的信息：

```yaml
bridge:
  port: 8434  # Synapse 访问 bridge 用的端口，任取
  bindAddress: matrixdiscord  # 跟容器名对应，这样在同个 network 下可以访问到
  domain: matrix.qq.com
  homeserverUrl: https://matrix.qq.com

# 权限看情况开，如果是个人服务器可以放的松一些
provisioning:
  whitelist:
    -   ".*"
relay:
  whitelist:
    -   ".*"
selfService:
  whitelist:
    -   ".*"
```

把文件放在 `/home/user/data/bridges/discord/config.yaml` ，然后运行以下命令生成 Synapse 使用的配置文件：

```bash
docker run -it --rm -v /home/user/data/bridges/discord:/data: z sorunome/mx-puppet-discord -r config.yaml
```

之后会在同目录下生成一个 `discord-registration.yaml` 文件，然后修改 `/home/user/data/homeserver.yaml`：

```yaml
app_service_config_files:
  - /data/bridges/discord/discord-registration.yaml
```

然后启动 discord bridge 容器：

```bash
docker run -d \
  --restart unless-stopped \
  -v /home/user/data/bridges/discord:/data: z \
  --network=matrix \
  --name=matrixdiscord \
  sorunome/mx-puppet-discord
```

再重启 Synapse 来激活相关配置：

```bash
docker restart synapse
```

这样就设置完成了，找 bot 账号 `@_discordpuppet_bot:matrix.qq.com` 跟它私聊 help 应该能看到回应。

接下来参考 [这里](https://github.com/Tyrrrz/DiscordChatExporter/wiki/Obtaining-Token-and-Channel-IDs) 获取 discord 的 user token，跟 bot 私聊 `link user <token>` 可以为账户建立 bridge，使用 `list` 查看所有 userid，`listguilds <userid>` 查看用户所有的 server，`bridgeguild <userid> <guildId>` 来 bridge 对应的 server。

## Telegram

Telegram 桥接通过 [mautrix-telegram](https://github.com/tulir/mautrix-telegram) ，相关文档在 [这里](https://docs.mau.fi/bridges/python/setup/docker.html?bridge=telegram) 。

首先生成样例配置文件：

```bash
docker run --rm -v /home/user/data/bridges/telegram:/data: z dock.mau.dev/tulir/mautrix-telegram:latest
```

然后修改 `registration.yaml` 相关配置：

```yaml
homeserver:
    address: https://matrix.qq.com
    domain: matrix.qq.com
appservice:
    address: http://matrixtelegram:29317  # 与容器名 match
    database: sqlite:////data/mautrix-telegram.db
    public:  # 可选修改
        enabled: true
        prefix: /telegram-bridge
        external: https://matrix.qq.com/telegram-bridge
bridge:
    max_initial_member_sync: 20  # 每个群初始化时同步多少用户，太大可能导致服务器僵尸用户爆炸多
    sync_channel_members: false
    permissions:
        '*': relaybot
        matrix.qq.com: full
        '@admin_user: matrix.qq.com': admin
logging:
    handlers:
        file:
            filename: /data/mautrix-telegram.log
```

再在 homeserver.yaml 中引入 telegram 的配置：

```yaml
app_service_config_files:
  - /data/bridges/discord/discord-registration.yaml  # 上一节的配置文件
  - /data/bridges/telegram/registration.yaml
```

最后启动桥接容器并重启服务器：

```bash
docker run -d --restart unless-stopped --network=matrix -v /home/user/data/bridges/telegram:/data: z --name=matrixtelegram dock.mau.dev/tulir/mautrix-telegram:latest
docker restart synapse
```

与账号 `@telegrambot:matrix.qq.com` 私聊，输入 `login`，之后按照提示输入手机号以及验证码，就可以登录了。 登录之后会自动同步所有的聊天。 需要注意的是如果之后登陆了 telegram 网页版的话就会导致 bot 被强制下线，而且目前没有很好的恢复办法，所以以后尽量不要登录网页版了。

## Whatsapp

Whatsapp 跟 Telegram 的 bridge 都是 mautrix 提供的，在同个网站可以找到文档。不过 Whatsapp 的风控也比较严格，有点像微信。 用法和上文类似，先生成 `config.yaml`，配置同样可以参考上文，至少要修改 address, domain 和 appservice.address：

```yaml
homeserver:
    address: https://matrix.qq.com
    domain: matrix.qq.com
appservice:
    address: http://matrixwhatsapp:29318  # 与容器名 match
```

`homeserver.yaml` 新增：

```yaml
app_service_config_files:
  - /data/bridges/discord/discord-registration.yaml  # 上一节的配置文件
  - /data/bridges/telegram/registration.yaml
  - /data/bridges/whatsapp/registration.yaml
```

之后启动 bridge 并重启服务器：

```bash
docker run -d --restart unless-stopped --network=matrix -v /home/user/data/bridges/whatsapp:/data: z --name=matrixwhatsapp dock.mau.dev/tulir/mautrix-whatsapp:latest
docker restart synapse
```

然后找 bot`@whatsappbot:matrix.qq.com` 私聊 login 即可，这里是作为网页版扫码登录，不过 Whatsapp 要求 app 在线的时候网页版才能用，所以比较鸡肋。 想要彻底摆脱手机 APP 的话可以启动一个安卓虚拟机运行 Whatsapp ，不过这要求虚拟机支持摄像头模拟来扫码， Win 和 Linux 下都没有很方便的选择，还不如找一个不用的旧手机常年开机。

## 微信

微信的桥接可以跟随 [项目文档](https://github.com/wechaty/matrix-appservice-wechaty) ，不过项目还处于初期开发阶段，我试了一下发现虽然能正常同步消息，但是用户的名字都是 wxid，不能正常显示用户名和昵称，处于几乎不可用的状态，因此先不写详细教程了。

# 附录

## 我构建聊天机器人的经历

第一次接触到 Chatbot 是在 2018 年大二的时候，参加了微软学生俱乐部的暑期活动，有一个项目是给 Windows 下面的语音助手 Cortana 开发新功能，我提出了一个快捷键小抄表的 proposal，大意就是在工作中想知道某个软件完成某个操作的快捷键是什么的时候，可以直接询问 Cortana，不用再打断工作流去互联网查询。

![cortana](cortana.png)

这个项目主要通过微软的自然语言理解服务 LUIS 来识别用户问题中的软件和操作，之后去预先积累的操作数据中查询并返回，Cortana 侧的交互是通过平台上的开发语言 SCL ( Semantic Composition Language ) 来实现的。 这是我第一次开发聊天机器人，SCL 比较简洁的设计也给了我很多启发。

后来再搞 Chatbot 就是单纯给自己用了，19 年的时候看到一篇关于人生日历的文章，就基于前端写了一个可以记录每天干的事情以及打分的网页，存储是把 xml 文件放在阿里云的 OSS 服务上实现的，但是每天去 OSS 服务上修改 xml 文件就很麻烦，于是就搞了一个 QQ 机器人，每天定时询问当天做了什么，然后把内容追加到 OSS 文件里。 后来到了考完试出分的时候，我觉得经常去教务刷网站看有没有处分很麻烦，就写了一个脚本定期抓取，每当有新课程的成绩出现就发 qq 消息通知。

<center>
<img src="qqbot-2019.png" height=400px />
</center>

<details>
  <summary>NSFW</summary>

  也是 19 年的时候，网上公开了一个 [NSFW 数据集](https://github.com/EBazarov/nsfw_data_source_urls) ，就一时兴起想做一个色图机器人，QQ 微信自然不太合适，于是搞了一个 Telegram 的 bot，每次发送指令，都会从 NSFW 数据集里面随机挑一张图片发出来，可惜感觉图片质量不高。 顺带一提，Telegram 官方提供了 Bot API，比 QQ 微信友好多了。

</details>

后来申请到了国外的学校，大家自发组织了一个刷题打卡微信群，我觉得群里之前的手动打卡方式非常原始，就搞了一个微信的打卡机器人：

<center>
<img src="wechat-bot.jpg" height=400px />
</center>

微信的机器人框架基本都挂掉了，比较靠谱的 Wechaty 并不是免费的，一个 Token 对应一个 session, 200/月，还好成为 contributer 的话可以免费获取，于是我贡献了一个 [Message Awaiter 插件](https://github.com/wechaty/wechaty-plugin-contrib/issues/13) ，成功拿到了免费 Token，不过需要每隔几个月就续期，加上微信对账号的风控很严格，还是挺不爽的。

时间来到 2020 年 8 月，QQ 机器人的框架开始大量被腾讯查封，也包括我使用的酷 Q，这导致我只能另寻出路。 好在这之前的一段时间就接触到了微信的机器人框架 Wechaty，开始用微信的一个小号来做日常的机器人，在迁移的过程中也修改了代码结构，使得逻辑代码可以兼容不同的底层聊天软件。

<center>
<img src="qqbot-baned.jpg" height=400px />
</center>

20 年底的时候，搬到了北京租房工作，开始搞一些身体健康检测和日常记录，需要有个触发事件时提醒的方式，就又开始搞 QQ Bot，发现虽然之前的酷 Q 跑路了，但是还有一个基于 Android QQ 的机器人框架，叫做 Mirai，试了下还是稳定好用的，而且都是基于 HTTP 的 API，没什么学习成本。 于是 QQ Bot 又复活了，负责每天出发事件的时候发消息提醒。

尽管如此，我还是想有个不依赖恶心人的腾讯的聊天机器人，考察了很多聊天软件之后，发现没有符合需求的服务 ( GFW 内外皆可用，不受审查，数据安全，开放 Bot API ) 。 只好采用自建 Matrix 聊天服务器的方式，虽然需要额外付出运维成本，但是前面的需求都能满足。 搭建好之后就把大部分个人相关的聊天机器人的工作迁移了过来，总算能一劳永逸的喘口气了。

最近 [受人启发](https://wechaty.js.org/2021/02/14/ziki-wechaty-helper/) ，搞了一个在特定 room 跟 bot 说话，自动把内容同步到 wiki 的对应条目上的功能，感觉方便了很多，可以作为平时想要记点什么的 inbox。

## Tips

如果长时间积累下来的图片视频占用空间很大，可以参考官方的 [Admin API](https://github.com/matrix-org/synapse/blob/develop/docs/admin_api/media_admin_api.md#delete-local-media-by-date-or-size) 来按照时间和大小筛选媒体文件并删除，需要加 admin 权限用户的 header ，找 header 的方式跟建立 bot 时一样。
