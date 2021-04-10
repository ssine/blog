---
title: "Self-hosted chat server, chatbot and multi-platform bridging"
date: 2021-03-27T00:36:39+08:00
---

__Note: This article is mainly translated by Google translator, please turn to the Chinese version for more accurate expression if possible.__

# Preface

Use open source solutions to build your own chat server, end-to-end encryption, no annoying EULA, control data yourself, you can form a federation with other servers, and even chat with other chat software's individuals and groups through bridges-it seems It's perfect. Of course, the entire system needs a little time to build. I just wanted to be a stable chat robot at first, but later I found out that self-built chat server brings more convenience.

![img](result.png)

This article is manually built on the Docker level. For students who do not have high requirements for detailed control or scalability, you can directly refer to it.[Ansible Playbook](https://github.com/spantaleev/matrix-docker-ansible-deploy) One-click deployment.

# Matrix 与 Synapse

First explain these two key concepts:

-   [Matrix](https://matrix.org/) It is a real-time chat protocol that defines the way of information transmission between Matrix server and server (federation ), server and client.
-   [Synapse](https://github.com/matrix-org/synapse) It is the official implementation of the server side of the Matrix protocol.

<center>
<img src="matrix-architecture.png" width=500px />
</center>

As shown above ([source](https://arxiv.org/pdf/1910.06295.pdf) ), each Homeserver has its own domain name (xxx.com), and each Matrix user belongs to a Homeserver, so it has a unique ID`@user:xxx.com` , Different Homeservers can communicate information with each other, so that all users can communicate with each other. Matrix does not distinguish between private chats and group chats. All chats are conducted in the chat room. Private chats are chat rooms with only two people. Room also belongs to a Server`#room:xxx.com` Yes, but the Homeserver is responsible for maintaining the rooms where all the users belong to, and also synchronizes Room information from other Homeservers.

For individuals, building a Homeserver (corresponding to a Synapse instance) is enough to meet the needs of daily chats.

The entire directory structure is as follows:

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

The above sh files are all docker startup commands. The data folder contains the data of all containers. Mount is in the synapse container, and data/bridges/xxx is the place where the corresponding bridge container mounts data.

First, prepare a domain name for Homeserver, assuming it is`matrix.qq.com` And the data directory path is`/home/user/data` . Then first generate the configuration file:

```bash
docker run -it --rm \
  -v /home/user/data:/data \
  -e SYNAPSE_SERVER_NAME=matrix.qq.com \
  -e SYNAPSE_REPORT_STATS=yes \
  matrixdotorg/synapse:latest generate
```

Will generate a`homeserver.yaml` Configuration file, the place that needs to be modified (the number of lines is not fixed, please search by yourself):

```yaml
server_name: "matrix.qq.com"
enable_registration: true
```

In order to allow the bridging container and synapse container to communicate with each other, establish a docker network:

```bash
docker network create matrix
```

Then start the server and connect to the network (note the port mapping):

```bash
docker run -d --name synapse \
  -v /home/user/data:/data \
  -p 80:8008 \
  --restart unless-stopped \
  --network matrix \
  matrixdotorg/synapse:latest
```

, If everything is normal, you can directly visit matrix.qq.com to see the page of Synapse successfully launched. You can register an account on the web client https://app.element.io/. Other clients of Element are also easier to use.

# Chatbot

Matrix comes with Bot API, which can be developed more conveniently with SDK. The familiar Typescript is used here. For more information, please refer to[Official tutorial](https://matrix.org/docs/guides/usage-of-matrix-bot-sdk) In short, it is to register an account for the bot through the Web terminal, get the token, and then adjust the SDK. Tip: You can use ts-node to run the typescript file directly for a better experience.

Unlike qq WeChat, everything here is designed and does not require Hack, so you can run easily. Instead, the focus is on what to do with bots, so let's also talk about what I used to do with chatbots. As some reference for use cases, this part is included in the appendix for a better reading experience.

# Multi-platform bridging

Another wonderful use case of the self-built Matrix server is to bridge other chat software, so that you don't need to open a lot of software to maintain the connection. If you have contact with outside GFW, there are still many chat software, such as Discord, Telegram, Whatsapp, etc. A more complete list can be found in[Official website](https://matrix.org/bridges/) See, I won’t repeat it. The following describes the construction process of each bridge in the simplest language.

## Discord

Discord has two bridge methods. One is to create a bot account, and then pull the bot to the server that needs to be synchronized, so that speaking is also through the bot; the other is to directly use the user's token to operate as the user, so The experience is better, there is no need to pull the bot, but the risk is greater. The second scheme is used below,[Official document](https://github.com/matrix-discord/mx-puppet-discord) 。

First put[Sample configuration](https://github.com/matrix-discord/mx-puppet-discord/blob/master/sample.config.yaml) Copy to the local, and then edit the necessary information inside:

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

Put the file in`/home/user/data/bridges/discord/config.yaml` , And then run the following command to generate the configuration file used by Synapse:

```bash
docker run -it --rm -v /home/user/data/bridges/discord:/data: z sorunome/mx-puppet-discord -r config.yaml
```

Then it will generate one in the same directory`discord-registration.yaml` File, then modify`/home/user/data/homeserver.yaml`：

```yaml
app_service_config_files:
  - /data/bridges/discord/discord-registration.yaml
```

Then start the discord bridge container:

```bash
docker run -d \
  --restart unless-stopped \
  -v /home/user/data/bridges/discord:/data: z \
  --network=matrix \
  --name=matrixdiscord \
  sorunome/mx-puppet-discord
```

Restart Synapse to activate the relevant configuration:

```bash
docker restart synapse
```

Now the setup is complete, find the bot account`@_discordpuppet_bot:matrix.qq.com` Talk to it privately, help, and you should be able to see the response.

Next reference[Here](https://github.com/Tyrrrz/DiscordChatExporter/wiki/Obtaining-Token-and-Channel-IDs) Obtain the user token of the discord, and chat privately with the bot`link user <token>` You can create a bridge for the account, use`list` View all userids,`listguilds <userid>` View all servers of the user,`bridgeguild <userid> <guildId>` Come to the server corresponding to the bridge.

## Telegram

Telegram bridge pass[mautrix-telegram](https://github.com/tulir/mautrix-telegram) , The relevant documents are in[Here](https://docs.mau.fi/bridges/python/setup/docker.html?bridge=telegram) 。

First generate a sample configuration file:

```bash
docker run --rm -v /home/user/data/bridges/telegram:/data: z dock.mau.dev/tulir/mautrix-telegram:latest
```

Then modify`registration.yaml` Related configuration:

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

Then introduce the configuration of telegram in homeserver.yaml:

```yaml
app_service_config_files:
  - /data/bridges/discord/discord-registration.yaml  # 上一节的配置文件
  - /data/bridges/telegram/registration.yaml
```

Finally start the bridge container and restart the server:

```bash
docker run -d --restart unless-stopped --network=matrix -v /home/user/data/bridges/telegram:/data: z --name=matrixtelegram dock.mau.dev/tulir/mautrix-telegram:latest
docker restart synapse
```

With account`@telegrambot:matrix.qq.com` Private chat, enter`login`, And then follow the prompts to enter the phone number and verification code, you can log in. After logging in, all chats will be automatically synchronized. It should be noted that if you log in to the telegram web version later, it will cause the bot to be forced offline, and there is currently no good recovery method, so try not to log in to the web version in the future.

## WhatsApp

Whatsapp and Telegram's bridge are both provided by mautrix. You can find the documentation on the same website. However, Whatsapp's risk control is also stricter, a bit like WeChat. The usage is similar to the above, first generate`config.yaml`, The configuration can also refer to the above, at least modify the address, domain and appservice.address:

```yaml
homeserver:
    address: https://matrix.qq.com
    domain: matrix.qq.com
appservice:
    address: http://matrixwhatsapp:29318  # 与容器名 match
```

`homeserver.yaml` Added:

```yaml
app_service_config_files:
  - /data/bridges/discord/discord-registration.yaml  # 上一节的配置文件
  - /data/bridges/telegram/registration.yaml
  - /data/bridges/whatsapp/registration.yaml
```

Then start the bridge and restart the server:

```bash
docker run -d --restart unless-stopped --network=matrix -v /home/user/data/bridges/whatsapp:/data: z --name=matrixwhatsapp dock.mau.dev/tulir/mautrix-whatsapp:latest
docker restart synapse
```

Then find bot`@whatsappbot:matrix.qq.com` You can chat with login privately, here is the web version scan code login, but Whatsapp requires the web version to be used only when the app is online, so it is rather tasteless. If you want to completely get rid of the mobile phone APP, you can start an Android virtual machine to run Whatsapp, but this requires the virtual machine to support camera simulation to scan codes. There is no convenient choice under Win and Linux. It is better to find an old mobile phone that you don't need to start up all year round.

## Wechat

WeChat's bridge can follow[Project documentation](https://github.com/wechaty/matrix-appservice-wechaty) , But the project is still in the early development stage. I tried it and found that although the messages can be synchronized normally, the user's name is wxid, and the user name and nickname cannot be displayed normally, and it is almost unusable, so I won't write a detailed tutorial.

# appendix

## My experience in building chatbots

The first time I came into contact with Chatbot was during my sophomore year in 2018. I participated in the summer activities of the Microsoft Student Club. One project was to develop new functions for the voice assistant Cortana under Windows. I proposed a shortcut key cheat sheet proposal. The general idea is that when you want to know what the shortcut key for a certain software is to complete a certain operation at work, you can directly ask Cortana instead of interrupting the workflow to search on the Internet.

![cortana](cortana.png)

This project mainly uses Microsoft's natural language understanding service LUIS to identify the software and operations in the user’s problem, and then query and return it from the pre-accumulated operating data. The interaction on the Cortana side is through the development language SCL (Semantic Composition Language) on the platform. To achieve. This is the first time I have developed a chatbot, and SCL's relatively simple design also gave me a lot of inspiration.

Later, I started Chatbot just for my own use. When I saw an article about life calendar in 19 years, I wrote a webpage based on the front-end that can record daily tasks and scores. The storage is to store the xml file in Ali. It is implemented on the cloud OSS service, but it is very troublesome to modify the xml file on the OSS service every day, so I set up a QQ bot to ask what was done on the day every day, and then append the content to the OSS file. Later, when the test scores were obtained after the exam, I felt that it was troublesome to go to the educational administration website to see if there were penalties, so I wrote a script to crawl it regularly, and send a QQ message notification whenever there is a new course score.

<center>
<img src="qqbot-2019.png" height=400px />
</center>

<details>
  <summary>NSFW</summary>

  Also in 19 years, a[NSFW data set](https://github.com/EBazarov/nsfw_data_source_urls) , I wanted to be a colormap robot on a whim. QQ WeChat was naturally not suitable, so I created a Telegram bot. Every time I send a command, it will randomly pick a picture from the NSFW data set and send it out. Unfortunately, I feel that the quality of the picture is not good. high. By the way, Telegram officially provides Bot API, which is much more friendly than QQ and WeChat.

</details>

Later, I applied to a foreign school, and everyone spontaneously organized a WeChat group for swiping and punching in. I think the manual punching method in the group was very primitive, so I created a WeChat punching robot:

<center>
<img src="wechat-bot.jpg" height=400px />
</center>

Wechat’s robot framework is basically down. The more reliable Wechaty is not free. One Token corresponds to one session, 200/month. Fortunately, if you become a contributer, you can get it for free, so I contributed one.[Message Awaiter plugin](https://github.com/wechaty/wechaty-plugin-contrib/issues/13) , I successfully got the free Token, but it needs to be renewed every few months. In addition, WeChat has strict risk control on the account, which is still quite uncomfortable.

When the time came to August 2020, the framework of QQ robots began to be blocked by Tencent in large numbers, including the cool Q I used, which caused me to find another way out. Fortunately, I came into contact with Wechaty, the robot framework of Wechat some time ago, and started to use a Wechat account as a daily robot. During the migration process, the code structure was also modified to make the logic code compatible with different underlying chat software. .

<center>
<img src="qqbot-baned.jpg" height=400px />
</center>

At the end of the 20th, I moved to Beijing to rent a house. I started to do some physical health testing and daily records. I needed a way to remind when an event was triggered. I started working on QQ Bot again. I found that although the previous cool Q had gone, it was still There is a robot framework based on Android QQ called Mirai, which is stable and easy to use after trying it out, and it is all based on HTTP API, so there is no learning cost. So QQ Bot is resurrected again, and is responsible for sending reminders when events are set off every day.

Nevertheless, I still want to have a Tencent chat bot that does not rely on disgusting people. After inspecting a lot of chat software, I found that there is no service that meets the requirements (GFW is available inside and outside, not subject to censorship, data security, and open Bot API). I had to adopt the method of self-built Matrix chat server. Although additional operation and maintenance costs are required, the previous requirements can be met. After it was built, most of the work of personal-related chatbots was migrated over, and finally I could catch my breath once and for all.

Recently [Inspired](https://wechaty.js.org/2021/02/14/ziki-wechaty-helper/) , I have made a function to talk to the bot in a specific room and automatically synchronize the content to the corresponding entry on the wiki. It feels a lot more convenient. It can be used as an inbox for memorizing something.

## Tips

If the pictures and videos accumulated over a long period of time take up a lot of space, you can refer to the official[Admin API](https://github.com/matrix-org/synapse/blob/develop/docs/admin_api/media_admin_api.md#delete-local-media-by-date-or-size) To filter and delete media files according to time and size, you need to add the header of the user with admin permission. The way to find the header is the same as when creating a bot.
