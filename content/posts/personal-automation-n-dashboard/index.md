---
title: "个人自动化系统与信息看板"
date: 2021-04-15T00:51:21+08:00
draft: true
---

有段时间比较关注数字健康和自动化，就折腾了一波相关的东西，搞了数据存储和看板。 回顾一下整个流程感觉像是个微型数据仓库，需要自己去获取数据源，设计存储格式并转移到数据库，然后把数据库里的信息展示出来。

# 自动化与数据收集

首先聊聊我关心的一些信息以及数据收集的过程。

## 安卓自动化工具 Tasker

虽然忘记了一开始为什么研究 Tasker，但是回过头来看 Tasker 是整个系统至关重要的一环，所以优先介绍。

首先是各类自动化工具的选择，不像 IOS 上有几乎一统天下的 ShortCuts 捷径 应用 ( 主要也是 IOS 权限管的严，官方给资源就是太子了 ) ，安卓上面的选择有很多，比较活跃的有 Tasker 和 Macrodroid, Tasker 是很久以前就存在的老牌应用了，社区相关的资源很多，不过 Macrodroid 的 UI 要比 Tasker 强不少。 因为重点还是要做事情，以及看重 Tasker 比较偏程序员的能力 ( js/shell 脚本、HTTP 相关 ) ，还是购买了 Tasker。

软件第一次打开的时候会有教程，下面简单描述一些比较重要的概念：

-   变量： 可以被访问和修改的全局变量，用于存储状态
-   任务： 相当于程序，由一组指令构成，指令可以做各种事情，比如请求 HTTP API
-   触发器： 用来判断何时执行任务的条件，可以分为：
  -   事件： 某件事情发生的时候，比如打开手机屏幕，扫描 NFC Tag
  -   位置： 进入某个位置时触发，需要开启定位，GPS->WLAN->基站，功耗和准确度都依次降低
  -   应用程序： 打开某个应用时触发
  -   时间： 在某个时间点触发
  -   状态： 进入或离开某个状态时触发，比如进入电量>10% 状态时
-   配置文件： 由触发条件和任务组成，在触发条件满足时执行任务的配置
-   项目： 顶级命名空间，可以管理多组上述变量、任务、触发器、配置文件

其他的信息结合后面的具体应用阐述。

## 日常打点

第一件想做的事情是记录 「每天各种事件发生的时间」 ，比如每天几点起床几点睡觉，什么时候上下班，等等。 醒来和入睡的时间要靠手环了，先说说通勤和工作的记录。

每天的日程可以概括成

```text
     通勤时间        工作时间           通勤时间
离家 <------> 到公司 <------> 离开公司 <------> 到家
```

只要记录这四个时间点，相关的时间就能得出了。 那么手机要如何得知这几个事件呢？

1.  通过位置信息，这是最直接的方式，但是高精度的 GPS 定位耗电很多，低精度的基站定位有不够准确，出门好久才反应过来。
2.  通过 Wifi 的连接与断开。 连接上家里的 Wifi=到家了，与家里的 Wifi 断开=离开家，公司也是一样。 不过这有个缺陷，比如在家的时候因故断开 Wifi 一阵子，就会被记为离开家又回来。
3.  Wifi 连接与断开+Wifi 变化。 这是 2 的增强版，如果我上一次断开的 Wifi 是家里的，这一次连接的 Wifi 是公司的，那么就可以确定这次连接 Wifi 等于来到公司了。 不过这仍然有个缺点，就是不好判定 Wifi 断开的时候是否真的是离开了 Wifi 所在地，因为你还不知道下一次连接的 Wifi 是什么 ( 只能等到下一次连接 Wifi 的时候再记录 ) 。

我目前的方式是利用第三种来判断进入地点的事件，并用扫 NFC Tag 的方式来判断离开地点，像是下班打卡。 其实要改成延迟记录的话纯 Wifi 就可以了，不过比较懒。

首先要维护「上一个 Wifi」的信息：

```text
Profile: Anon ( 23 )
  Restore: no
  State: Not Wifi Connected [SSID:* MAC:* IP:* Active: Any ]
Enter: Anon ( 27 )

Profile: Anon ( 25 )
  Restore: no
  State: Wifi Connected [SSID:* MAC:* IP:* Active: Any ]
Enter: Anon ( 29 )

Task: Anon ( 27 )
  A1: Variable Set [Name:%LastWIFI To:%CurrentWIFI Recurse Variables: Off Do Maths: Off Append: Off Max Rounding Digits:3 ]
  A2: Variable Set [Name:%CurrentWIFI To:. Recurse Variables: Off Do Maths: Off Append: Off Max Rounding Digits:3 ]

Task: Anon ( 29 )
  A1: Test Net [Type: Wifi SSID Data: Store Result In:%CurrentWIFI ]
```

就是在断开 Wifi 的时候更新 LastWifi，连接 Wifi 时给 CurrentWifi 赋值。

然后是回家和上班的记录：

```text
Profile: 记录：到家 ( 8 )
  Cooldown: 600 Restore: no
  Variables: [ %action: has value ]
  State: Wifi Connected [SSID:爷斋小二 MAC:* IP:* Active: Any ]
Enter: Anon ( 30 )

Task: Anon ( 30 )
  A1: Wait [MS:0 Seconds:5 Minutes:0 Hours:0 Days:0 ]
  A2: If [ %CurrentWIFI neq %LastWIFI ]
  A3: HTTP Request [Method: GET URL: https://a.b.c/life-tracer/log-action?action=%action]
  A4: End If

Profile: 记录：到公司 ( 14 )
  Cooldown: 600 Restore: no
  Variables: [ %action: has value ]
  State: Wifi Connected [SSID: ByteDance Inc MAC:* IP:* Active: Any ]
Enter: Anon ( 28 )

Task: Anon ( 28 )
  A1: Wait [MS:0 Seconds:5 Minutes:0 Hours:0 Days:0 ]
  A2: If [ %CurrentWIFI neq %LastWIFI ]
  A3: HTTP Request [Method: GET URL: https://a.b.c/life-tracer/log-action?action=%action]
  A4: End If
```

在 Wifi 连接时如果发生变化就去请求搭建好的 API 记录时间，API 的事情后面会说。

至于离开家和离开公司，就靠 NFC 标签触发了：

```text
Profile: 记录：离公司 ( 16 )
  Cooldown: 600 Restore: no
  Variables: [ %action: has value ]
  Event: NFC Tag [ID:048AFF32100289 Content:下班 ]
Enter: Anon ( 9 )

Task: Anon ( 9 )
  A1: HTTP Request [Method: GET URL: https://a.b.c/life-tracer/log-action?action=离公司]
```

这需要 NFC 标签和支持 NFC 的手机 ( 现在基本都支持 ) ，向标签中写入特定的值，或是直接使用 ID 来确定标签，并在标签被扫时触发相应的事件。 NFC 标签使用的是以前 Mock Switch Amiibo 的时候用的 NTAG215 钱币卡， [淘宝链接](https://m.tb.cn/h.4ppvxjN) 。 安卓应用 NFC Tools Pro 是个很好用的 NFC 读取和写入的工具。

虽然 NFC 标签有时候显得比较鸡肋，只是打卡记录时间，点手机屏幕也可以，但是还是比按手机上的按钮更有仪式感，能够放在醒目的位置提醒你去扫一下，效率也比解锁手机打开 App 记录要高。

## 身体指标



小米手环 inotify、体脂秤

## 数字健康

数字健康，手机&电脑屏幕使用时间统计&限制

# 数据存储

自建 API server, Tasker HTTP 收集

Mongodb

# 数据看板

看板结构，Grafana、MongoDB 插件、Plotly.js 插件&修改
