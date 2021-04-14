---
title: "HPE MicroServer 的设置与应用"
date: 2021-04-14T01:05:20+08:00
---

几个月前入手了 HPE MicroServer Gen 10 Plus ，从配置好到现在一直都在稳定运行，下面聊聊家庭服务器的相关配置和应用。 买一台主机放在家常年开机专门当做服务器使用，价格会比同配置的云服务器便宜很多，这是以平时的维护成本和家庭环境更低的稳定性为代价的，但是如果有居住条件和技术，这仍然是一个实惠且有趣的选择。 一个比较典型的场景是自建云盘，对硬盘的要求很高，但是硬盘容量大、带宽大、流量足的云主机通常很贵，这时使用家庭服务器就很方便，可以随时添加硬盘，内网也不会出现带宽和流量的瓶颈。

# 装机

慧与官方只出售 MicroServer 本体和 iLO 扩展卡，还有一些东西 ( 主要是硬盘 ) 需要自己购买安装，下面是后续用到的一些物品的淘宝链接。

-   [TOOLFREE 2.5 转 3.5 寸硬盘转接盒](https://m.tb.cn/h.4KYHEL0?sm=ad1309)
-   [西数蓝盘 1T 2.5 英寸 SATA3 SSD ](https://m.tb.cn/h.4ooR65C?sm=fd8d34)
-   [绿联 PCIE 转 NVME 扩展卡](https://m.tb.cn/h.4pgbuV8?sm=3b1128)
-   [西数 蓝盘 SN550 1TB](https://m.tb.cn/h.4LkV8IU?sm=644fe9)
-   [展示盒透明塑料](https://m.tb.cn/h.4oo7xSg?sm=f735fd)

## 硬件配置

MicroServer 本身的配置：

-   处理器 Intel Xeon E-2224 Quad-Core 3.4GHz 8MB CPU, Up To 4.6GHz Turbo
-   内存 16GB ( 2 x 8GB ) DDR4 PC4-21300 2666MHz Unbuffered Memory

其它不细说了，内存是 ECC 内存会很稳定，有四个网口。

我个人对声音比较敏感，在家时电脑机械硬盘转动的声音就足以让我感到烦躁，因此对于要 24 小时运作的服务器来说，还是优先考虑 SSD。 机器本身有四个 3.5 寸 SATA 盘位，但是主板上没有自带的 NVME SSD 槽位，所以想要高速 SSD 只能从 PCIE 转接。 我打算装一个 1T SSD 做虚拟机的主要硬盘，另外四个盘位留给 NAS 系统使用，因此入手了 PCIE 转 NVME 扩展卡和一个 SSD 硬盘。 SATA 盘位也暂时入手了一个 2.5 寸 SATA SSD 和一个 2.5 转 3.5 转接盒，这样就能顺利把 SSD 放入 SATA 盘位了。

对于组件安装的细节可以查阅 MicroServer 用户手册： [中文](https://psnow.ext.hpe.com/doc/a00073430zh_cn)  [英文](https://psnow.ext.hpe.com/doc/a00073430en_us) 。

## 系统选择安装

像这种裸的服务器通常有两种选择，装一个虚拟化管理系统并在里面装虚拟机，或是直接把系统装在主机上。 前者的好处是方便管理很多虚拟机，可以做备份迁移等操作，坏处是虚拟化会有一些性能损失，查了下资料发现计算性能是几乎不受影响的，主要是虚拟硬盘带来的 IO 性能下降，基本在 10% 以内。 后者好处是操作比较熟悉，坏处是稳定性比起专用的服务器管理系统要差很多。

个人使用的话，服务器裸机的管理系统主要有 VMWare 的 ESXi 和开源的 Proxmox VE ，前者靠谱稳定，后者更像是 Linux + 特殊虚拟化支持。 由于 HPE MicroServer 官方有定制的 ESXi 镜像，可以保证驱动兼容性，所以为了方便就使用 ESXi 系统了。 ESXi 可以理解成一个只包含 VMWare Workstation 虚拟机管理软件的操作系统，当然还有一些操作系统的功能，比如文件管理。

至于系统安装还是需要 HDMI 或者 DP 接显示器，把镜像烧录到 U 盘，设置 BIOS 从 USB 启动，然后装机。

-   镜像下载链接： https://my.vmware.com/cn/group/vmware/downloads/details?downloadGroup=OEM-ESXI70-HPE&productId=974
-   官方免费长期有效 Lisense: https://my.vmware.com/en/group/vmware/evalcenter?p=free-esxi7

## iLO

iLO 是一个远程控制用的扩展卡，ESXi 有两个 PCIE 槽位，我是一个转接了 SSD，另一个给 iLO. iLO 卡自带一个网口 ( 加上原本的四个就是五个了 ) ，是为了防止主机的网口被占满导致无法控制机器，如果带宽占用不高的话也可以把网线插在主机的网口上，iLO 可以和其它主机共享一个网口，IP 不同。

iLO 为远程控制主机提供了完整的方案，通过 Web 服务就能操控主机开关机，也可以通过 console 看到主机的显示输出。 iLO 的初始密码位于贴在服务器上的一个标签上，详情可以查阅 iLO 用户手册： [中文](https://psnow.ext.hpe.com/doc/a00048134zh_cn)  [英文](https://support.hpe.com/hpesc/public/docDisplay?docId=a00048134en_us) 。

# 应用

主机设置好之后，就可以安装虚拟机了，把想要的操作系统镜像上传到 ESXi，之后创建虚拟机。 存储空间按需分配，太大的虚拟机备份起来会比较麻烦。

## 虚拟机配置

ESXi 支持 Linux、Windows 等很多类虚拟机，虚拟机内部的配置就和本文无关了。 不过这里有一点要提一下，就是如何动态扩充虚拟机的磁盘空间。 虚拟机的磁盘都是 vmdk 格式的，在 ESXi 扩大 vmdk 大小之后，进入虚拟机 ( 这里是 Centos8，用 LVM 管理磁盘 ) ，运行 `fdisk` 可以看见显示大小不一致，输入 `w q` 可以修复。

之后创建一个新的主分区 ( GPT 支持 128 个而不是 4 个，随便创建 ) ，文件系统类型选择 Linux LVM 。

然后是 LVM 的事情 ( 参照 [这里](https://www.cnblogs.com/gaojun/archive/2012/08/22/2650229.html) ) ，运行 `pvcreate /dev/sdax` 新加一个物理卷，之后 `vgdisplay` 查看所有的卷组，把新加的物理卷放到需要增加的卷组 `vgextend cl /dev/sdax` ，然后就能给卷组里面的逻辑卷扩容了： `lvdisplay` ， `lvextend -l +xxx cl-home` 。 最后把文件系统扩张到卷大小： `xfs_growfs /dev/mapper/cl-home`( 虚拟磁盘设备在 `fdisk -l` 这里找 ) 。

对于 NAS 服务有必要做硬盘直通，可以让虚拟机内部直接控制整块硬盘，包括温度等信息，也能让 HDD 在不用时休眠。 在 ESXi 里面创建直通的虚拟硬盘然后添加到对应的虚拟机就可以了。

## 一些服务

至于要拿家庭服务器来干什么就因人而异了，简单列举下我目前在运行的服务：

-   Jupyter Lab，参见 [打造有多个内核的 Jupyter Lab 容器](https://ssine.ink/posts/versatile-jupyter-lab/)
-   Matrix 聊天服务器 Synapse， 参见 [自建聊天服务器、机器人与多平台桥接](https://ssine.ink/posts/matrix-bot-and-bridges/)
-   RSS 相关：
  -   [RSSHub](https://docs.rsshub.app/) ，用于提供 RSS 信息源
  -   [Tiny Tiny RSS](https://tt-rss.org/) ，RSS Web 客户端，用于阅读
-   Mongodb，存储一些格式化的个人信息
-   Grafana，关注的信息看板
-   NextCloud，云盘，直通了 1T 外部 SSD 硬盘
-   Bitwarden，密码管理
-   Calibre，电子书管理，没怎么用，建议先养成看书习惯再搞
