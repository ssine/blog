---
title: '我的世界基岩版 Mod说明'
date: '2019-02-25'
dateformat: 'Y-m-d H:i'
tags:
    - minecraft
categories:
    - 项目
---

微软收购 Minecraft 之后，用 C++ 重写了这款游戏，为之前代码混乱、 bug 频出、 mod 开发困难的 MC 带来了一片光明，并称这个高性能且稳定的版本为基岩版。

<!-- more -->

# 基岩版的优劣

基岩版相对于 Java 版实在是有太多改进， Java 版的缺点详见 [我的世界这个游戏的代码能排在游戏代码中排第几？
](https://www.zhihu.com/question/302736236) 。 虽然基岩版不开源，但是从使用体验上来说，比 Java 版好很多，微软收购它之后更新速度飞快，从 Wiki 上偏开发的部分也能看出游戏组织比较好，官方还为脚本提供了事件接口（Java版中并没有明确的事件机制）。

更令人开心的一点是，官方不再像以前那样对 mod 暧昧不明，而是很开放的支持大家创作自己的插件(Add-on)。 Java 版的 mod 开发全靠反编译游戏、基于社区改造的API Forge 编码，再编译成 mod ，语言也只能用 Java 。 现在官方提供了脚本引擎，可以直接使用 JavaScript 编写 mod，对于计算需求高的渲染部分有一套高效的 DSL ，语言与性能都堪称完美。 官方的支持也使得插件作者能够方便的盈利，并有统一的平台管理 mod 。

基岩版尚比不过 Java 版的地方是它还缺少使用它进行开发的社区与 mod 积累，相比于各式各样的 Java 版 mod ，基岩版的插件少得可怜，这也跟官方提供的脚本 API 功能有限有关。 在论坛上可以看到很多人都在期待着基岩版脚本引擎的成熟，如果还不断的有人从编程的角度进入 MC 的世界并欣赏它的魅力，我相信基岩版才是未来的主流。

# 项目结构

所有内容基于 Win10 编写。

Minecraft 基岩版 (Minecraft Bedrock Edition 简称 MCBE ) 的目录结构与 Java 版的不太一样，存储游戏与信息的根目录位于

```text
%userprofile%\AppData\Local\Packages\Microsoft.MinecraftUWP_8wekyb3d8bbwe\LocalState\games\com.mojang
```

打开就可以看到行为包、资源包、世界存档等内容，文件夹名字的含义已经很清晰了。

## MCBE 插件管理

MCBE 接管了以前 Java 版中需要启动器来完成的工作： 插件管理。 插件管理的逻辑是，向游戏中下载插件，并在某个世界中使用它。 这一点与启动器的逻辑不同，启动器是直接向某个游戏版本中注入插件，而这会带来很多问题，比如未注入插件的游戏是同一版本，存档却可能不兼容——这也是无奈之举，因为当时只能通过反编译来实现 mod 。

MCBE 所有导入游戏中的插件都位于根目录下的 `behavior_packs` 或 `resource_packs` 中，如果你在创建世界时勾选了要使用插件，那么这些插件会在创建世界时复制一份到这个世界的存档中。 也就是说，插件的版本停留在创建世界时的状态，即使你重新向游戏导入了该插件。 这也为我们带来了方便： 只需要在一个世界存档中修改插件就可以直接看到效果了。

## 脚本引擎与行为包

插件分为两类： 资源包与行为包。 资源包负责像游戏中添加材质、皮肤、音效，行为包负责控制实体的行为。 1.9 版本后新增加的脚本引擎可以执行 Javascript 脚本，这项功能被包括在行为包中。 也就是说，所谓的开发脚本是通过编写行为包的一部分来完成的。

游戏中的行为包位于根目录下 `behavior_packs` 文件夹中，里面的每一个文件夹代表一个行为包，里面的 `manifest.json` 包含了包的信息， `pack_icon.png` 是行为包的图标，其余的文件夹可根据实际内容创建，但有确定的命名规则。

我最关注的脚本位于 `scripts` 文件夹中，里面分为两个文件夹： `client` 与 `server` ，分别表示在客户端与服务端运行的代码。一个最小可运行的行为包目录结构如下：

![minimal](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/mcbe-addin-minimal.png)

`scripts` 文件夹中的服务端脚本总是会运行，要使得客户端脚本也能运行，需要在 `manifest.json` 中加上类型为 `client_data` 的 `module`，如下：

```json
{
    "format_version": 1,
    "header": {
        "description": "Hellow world behavior pack",
        "name": "Hellow World Pack",
        "uuid": "541b2d4d-aab3-4e44-8da9-1c78e75d9e71",
        "version": [0, 0, 1]
    },
    "modules": [
        {
            "description": "Hellow world behavior pack client",
            "type": "client_data",
            "uuid": "8ccc04dd-dba9-4c0f-9a2d-75f4d15a52be",
            "version": [0, 0, 1]
        }
    ]
}
```

# API 与文档

目前官方提供的事件还很少，脚本想要做出行为也基本上只能通过触发事件。 Wiki 上对于各种数据类型有解释，不过仍在开发中，因此读起来并不通顺，这篇文章也是记下一些 Wiki 中没有提到，我自己摸索出来的开发需要的知识。

* [插件简介](https://minecraft.gamepedia.com/Add-on)
* [脚本引擎](https://minecraft.gamepedia.com/Bedrock_Edition_beta_scripting_documentation)
* [游戏命令](https://minecraft.gamepedia.com/Commands)

脚本引擎部分有目前支持的事件、数据结构等信息，脚本想要对世界进行修改（放置、摧毁方块、生成实体等）主要通过发出命令。 在游戏命令部分有相应的介绍。
