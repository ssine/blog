---
title: 'CHIP-8 模拟器'
date: '2018-03-16'
---

来自计算机组成原理课程设计的项目。

<!-- more -->

# 介绍

## 为什么要做它

计算机组成原理的实验课要用高级语言写一个 Virtual Machine ，第一步是先交设计报告，这几天就一直在思前想后要做什么样的 VM 。 软件重制上学期的 Proteus CPU 的话实在太寒碜，想写个 Y86 仿真工程量又太大。 今天上课的时候看到朋友在查 NES 模拟器，受到启发，做一个游戏机模拟器的话多有意思，还能运行游戏成就感满满啊。 原本就对早期的游戏很感兴趣，上世纪中旬电子游戏刚兴起的时候，那些游戏都是鼻祖呢。 之后便开始搜集资料了，根据历史，最早的游戏是在实验室的示波器上玩的，之后是在 PDP-1 机器上，再之后出现了第一台街机。 然而这些机器关注度不高，开发文档也比较少。 之后比较热门的是 NES 游戏机（日本的 FC ）。 还有一个我比较感兴趣的 Gameboy。 深入了解之后发现要实现它们的工作量实在是有点大， 大约是个半年多的项目，拿来做二分之一的 Final Project 有些蠢萌。（Gameboy 系列有 Gameboy Color 和 Gameboy Advanced 两个升级版， GBA直接就 ARM7指令集了，虽然非常非常想做但那还是太不自量力了。） 在我抱着对巨大工作量的恐惧浏览 NES 相关资料的时候，发现了一个神奇的东西 —— Chip-8。

这是一个1970年提出的解释性编程语言（interpreted programming language），运行在专用的 **CHIP-8 Virtual Machine** 上， 为了帮助人们更方便的开发游戏。 Developer: Joseph Weisbecker. 当然，重点是， Chip-8 大约是我目前可以实现的最简单的实际存在的仿真项目了。 So just work on it, let's go.

## 相关资源

英文教程 [How to write an emulator (CHIP-8 interpreter)](http://www.multigesture.net/articles/how-to-write-an-emulator-chip-8-interpreter/)

技术文档 [Cowgod’s Chip-8 Technical Reference v1.0](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM)

游戏ROM合集包 [Chip-8 Games Pack](https://www.zophar.net/pdroms/chip8/chip-8-games-pack.html)

Github 项目 [Chip-8 Emulator](https://github.com/alexanderdickson/Chip-8-Emulator)

中文教程 [手把手教你编写游戏模拟器 - Chip8篇(1)](http://blog.csdn.net/korekara88730/article/details/50987930)

js聪明地绘图 [requestAnimationFrame for Smart Animating](https://www.paulirish.com/2011/requestanimationframe-for-smart-animating/)

一些游戏的说明 [David Winter's CHIP-8 emulation page](http://www.pong-story.com/chip8/)

Chip8 到 Super Chip8 [SUPER-CHIP v1.1](http://devernay.free.fr/hacks/chip8/schip.txt)

# 机器概览

首先要找到关于 Chip-8 VM 的详细信息， 以下内容整理自[维基百科](https://en.wikipedia.org/wiki/CHIP-8#Virtual_machine_description)。

## 内存

CHIP-8 有 4K(`0x1000`) 内存， 8 位字长（chip 8 的由来）。 CHIP-8 解释器占据内存前 512 字节(`0x200`)， 最高的 256 字节(`0xF00-0xFFF`)为显示刷新预留，它下面的 96 字节(`0xEA0-0xEFF`)为堆栈、内部使用和其他变量保留。

在现在的实现中，解释器作为外部代码不需要再放在内存中运行， 但一般会在低 512 字节中存储字体数据。

## 寄存器

CHIP-8 有 16 个 8 位寄存器， 从 `V0` 到 `VF`。 `VF` 寄存器作为一些指令的标志位， 应当避免使用。 （加法中进位标志， 减法中无借位标志，绘图中与像素碰撞有关）

地址寄存器的名字叫做 `I`， 有 16 位字长，与几个内存操作指令有关。

## 堆栈

堆栈只在子程序被调用时存储返回地址。 现在的实现通常有最少 16 层。

## 计时器

CHIP-8 有两个计时器。 它们以 60 Hz 的速度 count down，直到 0 。

* 延迟寄存器： 用于为游戏中的事件计时，它的值可以被设置和读取。
* 声音寄存器： 用于声音效果，如果它的值不是0，发出蜂鸣。

## 输入

输入设备是一个 0 到 F 的 hex keyboard 。 其中 `8462` 通常被用于方向输入。 有三条指令用于检测输入。 一个是当某个特定键被按下时跳过一条指令，另一个在没有按键时跳过一条指令。 第三个等待一个键盘输入， 并把它保存在一个寄存器里。

## 图形与声音

原生的 CHIP-8 显示分辨率为 64×32 像素， 单色。 图像通过向屏幕绘制宽 8 ，高 1 到 15 的 sprites 实现。 Sprite 像素与对应位置屏幕上的像素点异或， 因此是在反转屏幕对应位置的像素点。 在一个 sprit 被画出时，如果屏幕上有像素点从亮变暗， `VF` 寄存器就会被设置为 1 ，否则为 0 。 这被用于碰撞检测。

声音部分在声音寄存器中说过。

## 操作码表

CHIP-8 有 35 个操作码，全部是 2字节长， 以大端法储存。 下面是列表，标志如下：

* NNN: 地址
* NN: 8 位常数
* N: 4 位常数
* X and Y: 4 位寄存器标识符
* PC : 程序计数器
* I : 内存寻址用16位寄存器

懒的翻译了……表格详见[维基百科](https://en.wikipedia.org/wiki/CHIP-8#Opcode_table)
