---
title: "HPE MicroServer 的设置与应用"
date: 2021-04-14T01:05:20+08:00
draft: true
---

装机、虚拟化： ESXI vs PVE

iLO 卡，单根网线复用

虚拟机直连裸盘，LVM 扩展分区

lisense: https://my.vmware.com/en/group/vmware/evalcenter?p=free-esxi7

download: https://my.vmware.com/cn/group/vmware/downloads/details?downloadGroup=OEM-ESXI70-HPE&productId=974

文档： https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=emr_na-a00073430en_us

买一台主机放在家常年开机专门当做服务器使用，价格会比同配置的云服务器便宜很多，这是以平时的维护成本和家庭环境更低的稳定性为代价的，但是如果有居住条件和技术，这仍然是一个实惠且有趣的选择。 一个比较典型的场景是自建云盘，对硬盘的要求很高，但是硬盘容量大、带宽大、流量足的云主机通常很贵，这时使用家庭服务器就很方便，可以随时添加硬盘，内网也不会出现带宽和流量的瓶颈。 几个月前入手了 HPE MicroServer Gen 10 Plus ，从配置好到现在一直都在稳定运行，下面聊聊家庭服务器的相关配置和应用。

# 装机

慧与官方只出售 MicroServer 本体和 iLO 扩展卡，还有一些东西 ( 主要是硬盘 ) 需要自己购买安装，下面是后续用到的一些物品的淘宝链接。

-   [TOOLFREE 2.5 转 3.5 寸硬盘转接盒](https://m.tb.cn/h.4KYHEL0?sm=ad1309)
-   [WD 西数 Blue 3D 版 1T 2.5 英寸 SATA3 SSD](https://m.tb.cn/h.4ooR65C?sm=fd8d34)
-   [展示盒透明塑料](https://m.tb.cn/h.4oo7xSg?sm=f735fd)
-   [绿联 pcie 转 nvme 扩展卡](https://m.tb.cn/h.4pgbuV8?sm=3b1128)
-   [WD/西部数据 蓝盘 SN550 1TB](https://m.tb.cn/h.4LkV8IU?sm=644fe9)

## 硬件配置

我个人对声音比较敏感，在家时电脑机械硬盘转动的声音就足以让我感到烦躁，因此对于要 24 小时运作的服务器来说，还是优先考虑 SSD。 机器本身有四个 3.5 寸盘位，

## 系统选择

## iLO

# 应用

## 虚拟机配置

## 一些服务
