---
title: "提取微信聊天记录"
date: 2020-04-03T00:17:06+08:00
---

最近需要与人共享一些聊天记录，由于量大截图太麻烦，就想直接拿微信的数据，顺便备份一下。 走的是常见的流程，取 EnMicroMsg.db 文件，找密码，导出。 不过在获取数据库密码时，用于注册微信的手机 imei 已不可考，尝试了当前手机的也失败了，因此只能走暴力破解的路子，详见下文。

整个过程分为三步：

1. 提取聊天消息数据库
2. 获取数据库密码，解密
3. 导出聊天记录

# 提取聊天消息数据库

微信的用户消息用 sqlite3 文件存储，位于 `/data/data/com.tencent.mm/MicroMsg/{hash}/EnMicroMsg.db` 。 只要能把它取出，后面的障碍就不多了。 然而，拥有 `/data/data/com.tencent.mm` 文件夹访问权限的用户只有: 1. 腾讯开发用户 2. root 。 由于我的手机已经 root 过，要导出这个文件很简单，下面为非 root 用户提供几个思路：

## 备份到有 root 权限的系统

既然本机没有 root 权限，就通过微信的聊天记录迁移把当前手机的聊天记录转移到有 root 权限的系统上。 最靠谱的方法是找一台 root 过的手机，如果没有这样的设备，还可以在电脑上安装一个安卓模拟器，模拟器通常是带有 root 权限的。 我曾经尝试过夜神模拟器，不过它只支持安卓 4 ，已经是上古版本了，微信登陆帐号之后一会就闪退了。 除了夜神还有很多种模拟器可以尝试，读者可以自行寻找。

## 借助厂商的备份功能

有一些手机厂商提供了系统级别的备份软件功能，可以借助这个功能获取微信存放数据的目录的备份，然后把备份文件解包，就可以获取这些信息了。 我所知道的有小米，不过没有实际尝试过。 三星手机曾经支持这个功能，不过后来取消了，估计也是出于安全考虑。

## 伪造开发者用户

既然开发者可以访问自己应用的数据目录，那么还可以把自己伪装成开发者，通过 adb shell 把里面的数据拷贝出来。 按照安卓的安全策略，只有开发模式打包的应用才可以用 adb shell 访问数据目录，因此，需要获取一份开发状态的微信安装包。 我曾经尝试反编译微信，更改里面的开发模式标记，再打包回去安装，不过过程极为繁琐，而且因为二次打包时签名与以前不同，无法替代上一个微信应用，就算安装上也无法取得之前的数据了。 不过我没有安卓开发的经验，或许有经验的开发者能够行得通吧。

最后说一句，如果以前没有 root 但是为了获取聊天记录想要 root ，一定要慎重考虑，因为 root 过程有一步是需要删除所有数据的，不删除强行 root 的后果不可预料。

# 获取数据库密码与解密

上一步取出的 `EnMicroMsg.db` 数据库文件是加密过的，加密方式为 SQLCipher V2 。 下面介绍两种获取密码的方式。

## 通过 IMEI 生成密码

网上已经有不少博客提到了微信数据库密码的生成方式，是手机的 IMEI 号码与微信 uin 拼接起来之后的 MD5 值的前 7 位， MD5 取小写字母。

首先是 IMEI ，在手机上拨号 `*#06#` 就可以查看，每个卡槽一个 IMEI （取前 14 位），每部手机一个 MEID ，都拿来试试。 不过蛋疼的一点是 IMEI 未必是当前手机的，我同样看到有说法是注册微信时所用手机的 IMEI 。

剩下的时 uin ，类似于微信用户的 id ，位于 `/data/data/com.tencent.mm/shared_prefs/auth_info_key_prefs.xml` ：

<img src="./auth_info_key_prefs.jpg">

不管 uin 是不是负数，当作字符串和 IMEI 拼接起来就好。 然后把 IMEI + uin 这个字符串拿去计算 MD5 值（百度一下就会有计算的在线工具），取出前 7 位（小写）就是密码。

## 暴力破解

我通过上一节的流程得到的密码并不对，甚至翻出来高中用的手机拿来尝试，同样失败了。不过密码只有 7 位，字符集也有限（MD5 是十六进制字符串），这样总共只有 $16^7$ 种可能性，暴力破解的话计算量也是可以接受的。

网上有现成的工具，我就不造轮子了： [EnMicroMsg.db-Password-Cracker](https://github.com/chg-hou/EnMicroMsg.db-Password-Cracker) ，这个程序使用 CPU ，我的 i7-6700HQ 跑起来大约是 200条/s ，需要 15 天。

这个时间长度还是让人有点难以接受，好在这个仓库下面一个 issue [CUDA version completed but won't release](https://github.com/chg-hou/EnMicroMsg.db-Password-Cracker/issues/10) 里，有个口嫌体正直的用户说着不会放出来自己写的 GPU 版本，还是发布出来了： [SQLCipher-Password-Cracker-OpenCL](https://github.com/whiteblackitty/SQLCipher-Password-Cracker-OpenCL) 。 要跑它的话需要 OpenCL + CUDA 环境，在项目的 README 中有比较详细的配置步骤。 搭环境着实废了一我番功夫，由于 CUDA 的各种问题，花了一天才折腾好。

这个程序的速度就十分可观了，我的 GPU 是 GTX 1060 ，平均速度是 190k条/s ，最坏情况也只要 20 min 。

我跑完之后发现没有找到密码，研究了一下，在 issue [Dose not work on my database in Feb. 2018](https://github.com/chg-hou/EnMicroMsg.db-Password-Cracker/issues/6) 里找到了原因，是微信 7.0 版本更改了数据库的读写头，导致对解密成功的判定出现了问题，这在 CPU 版程序的 [commit 20eb4](https://github.com/chg-hou/EnMicroMsg.db-Password-Cracker/commit/20eb4f9bd8bb440bcc6817b4b7981b8bf58478a4) 中修复了。 不过 GPU 版本并没有更改，因此需要手动更正：

将 `Lib/pbkdf2-sha1_aes-256-cbc.cl` 文件的 752 行 （[link](https://github.com/whiteblackitty/SQLCipher-Password-Cracker-OpenCL/blob/7dc8147c315d625426e79673a5dc5bad7f0177b3/Lib/pbkdf2-sha1_aes-256-cbc.cl#L752)） 修改为

```cpp
if(((uint)(data[5] ^ iv[5])==0x40) && ((uint)(data[6] ^ iv[6])==0x20) && ((uint)(data[7] ^ iv[7])==0x20))
```

。 之后再运行，成功得到了密码。

## 数据库解密

如果是暴力破解方式，得到密码的同时程序会保存一份解密后的数据库。 如果是计算方式，可以参考 [wechat-dump](https://github.com/ppwwyyxx/wechat-dump) 项目，里面提供了使用 IMEI 与 uin 解密的脚本。

# 导出聊天记录

解密后的数据库就是普通的 sqlite3 文件，可以用很多软件打开，这里推荐 SQLiteStudio 。

<img src="./sqlitestudio.png">

message 表里有所有的聊天记录，当然不管用什么软件，可读性都很差，需要有程序把它们按照聊天群组/人分开展示。

[wechat-dump](https://github.com/ppwwyyxx/wechat-dump) 库是一个很好的从数据库中提取消息并分类导出的库，需要在 linux + python 2 环境下运行，具体配置参见项目 README 。

列出所有联系人：

```bash
./list-chats.py decrypted.db
```

导出所有聊天记录为文本文件：

```bash
./count-message.sh output_dir
```

将某个对话渲染为 html 文件：

```bash
./dump-html.py "<contact_display_name>"
```

导出的 html 文件可以用浏览器打开，也可以进一步打印成 pdf 。

<img src="./result.jpg">

---

用户产生的数据不属于用户自己，说来也是可笑，应用的设计并不是以用户体验为中心，而是以用户的钱包为中心。 如此严格的限制聊天数据可以极大的提升微信用户的迁移成本，因为用户无法将聊天记录导入其它的软件。 但是这样做对用户来说就极为不友好，聊天记录漫游的功能需要付费，而且最高级别的漫游也不能保存全部聊天记录，这样用户根本无法在手机之外备份数据，手机出问题数据就全没了。 我一直对 QQ 和微信这样的行为深恶痛绝，奈何上一个手机已经用了很久，想要 root 会失去全部数据，想要通过安卓模拟器暗度陈仓又问题频出。 因此，换手机之后我的第一件事就是 root ，冒着设备安全性的风险，不为了各类插件，只为取回自己的数据。
