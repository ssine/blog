---
title: 'Minecraft Bedrock Edition Mod Description'
date: '2019-02-25'
dateformat: 'Y-m-d H:i'
tags:
    - minecraft
categories:
    - 项目
---

After Microsoft acquired Minecraft, it rewrote the game in C++, which brought a bright future to MCs with confusing code, frequent bugs, and difficult mod development, and called this high-performance and stable version a bedrock version.

<!-- more -->

# The pros and cons of the bedrock version

1Bedrock version with respect to the Java version of it is there are too many improvements, Java version of the shortcomings detailed in [Minecraft game code can be ranked in the game code in the first few rows?](https://www.zhihu.com/question/302736236) . Although the bedrock version is not open source, it is much better than the Java version in terms of experience. After Microsoft acquired it, the update speed is very fast. From the part of the wiki development, the game organization is better. The official also provides the script. Event interface (there is no explicit event mechanism in the Java version).

Even more gratifying is that the official is no longer ambiguous about mods as before, but is very open to support everyone to create their own add-ons. The Java version of mod development relies on decompilation games, community-based API Forge coding, and then compiled into mods, and languages ​​can only be used in Java. The official scripting engine is now available, and mods can be written directly in JavaScript. There is a set of efficient DSL for the computationally demanding rendering part, and the language and performance are perfect. Official support also makes plug-in authors profitable and has a unified platform management mod.

The bedrock version is still better than the Java version. It is still lacking the community and mods that use it for development. Compared to the various Java versions of the mod, the bedrock version of the plugin is very poor, which is also provided by the official. The scripting API has limited functionality. In the forum, we can see that many people are looking forward to the maturity of the bedrock version of the scripting engine. If someone continues to enter the MC world from the perspective of programming and appreciate its charm, I believe that the bedrock version is the mainstream of the future.

# Project structure

All content is written in Win10.

The Minecraft Bedrock Edition (MCBE) has a different directory structure than the Java version. The root directory for storing games and information is located at:

```text
%userprofile%\AppData\Local\Packages\Microsoft.MinecraftUWP_8wekyb3d8bbwe\LocalState\games\com.mojang
```

When you open it, you can see the behavior package, resource package, world archive, etc. The meaning of the folder name is already clear.

## MCBE plugin management

MCBE took over the work that was required in the previous Java version to require an initiator: plugin management. The logic of plugin management is to download the plugin into the game and use it in a certain world. This is different from the logic of the launcher. The launcher directly injects the plugin into a certain game version, and this brings a lot of problems. For example, the game without the plugin is the same version, but the archive may not be compatible - this is helpless. This is because mod can only be implemented by decompilation.

MCBE plug all imported games are located in the root directory `behavior_packs` or `resource_packs`. If you want to check the use of plug-in when creating the world, these plug-ins will copy to the archive in the world in the creation of the world. That is to say, the version of the plugin stays in the state when the world was created, even if you re-import the plugin into the game. This also brings us convenience: you only need to modify the plugin in a world archive to see the effect directly.

## Scripting engine and behavior package

There are two types of plugins: resource bundles and behavior packs. The resource bundle is responsible for adding materials, skins, and sound effects to the game. The behavior package is responsible for controlling the behavior of the entity. The newly added scripting engine after version 1.9 can execute Javascript scripts, which are included in the behavior package. In other words, the so-called development script is done by writing a part of the behavior package.

The behavior of the package the game in the root directory `behavior_packs` folder inside each folder represents a behavior package, which is `manifest.json` contained packets, `pack_icon.png` is an icon behavior package, the rest of the folders can be created based on the actual content, but there are Determined naming rules.

I am most concerned about the script is in `scripts` the folder, which is divided into two folders:  `client` and `server` , respectively, the code on the client and the server is running. A minimally runable behavior package directory structure is as follows:

![minimal](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/mcbe-addin-minimal.png)

`scripts` Server-side script in the folder will always be run, to make the client-side script can run, you need `manifest.json` the plus type `client_data` of `module`, as follows:

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

# API and documentation

There are currently very few official events, and scripts that want to behave are basically only able to trigger events. There are explanations for various data types on the wiki, but they are still under development, so it is not fluent to read. This article also writes down the knowledge that is not mentioned in the wiki and I have explored myself.

* [Introduction to the plugin](https://minecraft.gamepedia.com/Add-on)
* [Scripting engine](https://minecraft.gamepedia.com/Bedrock_Edition_beta_scripting_documentation)
* [Game command](https://minecraft.gamepedia.com/Commands)

The script engine part has information such as the currently supported events, data structures, etc. The script wants to modify the world (place, destroy blocks, generate entities, etc.) mainly by issuing commands. There is a corresponding introduction in the game command section.
