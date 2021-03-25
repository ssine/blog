---
title: "自建聊天服务器、机器人与多平台桥接"
date: 2021-03-08T00:36:39+08:00
draft: true
---

# 前言

利用开源解决方案自建聊天服务器，端到端加密，没有恼人的 EULA，自己控制数据，可以和其他的服务器组成联邦，甚至还能通过桥接与其它聊天软件的个人和群组聊天——看起来十分的完美。 当然，整个系统需要一点时间来搭建，我一开始只是想做一个稳定的聊天机器人，后来才发现自建聊天服务器带来的的更多便利。

## 我构建聊天机器人的经历

在进入正题之前想先聊聊我对于构建聊天机器人的一些经历，也是展示一些聊天机器人的 Use Case，不感兴趣的读者请自行跳过。

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

# Matrix 与 Synapse

首先解释一下这两个关键概念：

-   [Matrix](https://matrix.org/) 是一个及时聊天的协议，定义了 Matrix 服务器与服务器 ( federation ) 、服务器与客户端之间的信息传递方式。
-   [Synapse](https://github.com/matrix-org/synapse) 是对 Matrix 协议中服务器端的官方实现。

<center>
<img src="matrix-architecture.png" width=500px />
</center>

如上图 ( [来源](https://arxiv.org/pdf/1910.06295.pdf) ) 所示，每个 Homeserver 有自己的域名 ( xxx.com ) ，每个 Matrix 用户属于一个 Homeserver，因此有一个唯一的 ID `@user: xxx.com` ，不同的 Homeserver 之间可以互相沟通信息，使得所有用户之间可以互相交流。 Matrix 没有区分私聊和群聊，所有聊天都是在聊天室 Room 中进行的，私聊就是只有两个人的聊天室。 Room 也是属于某个 Server `#room: xxx.com` 的，不过 Homeserver 负责维护所有所属 User 所在的 Room，也要从别的 Homeserver 同步 Room 的信息过来。

对于个人来说，搭建一个 Homeserver ( 对应一个 Synapse 实例 ) 就可以满足日常的聊天等需求了。

# 聊天机器人

# 多平台桥接

## Discord

## Telegram

## Whatsapp

## 微信
