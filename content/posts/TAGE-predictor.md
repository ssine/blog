---
title: '分支预测器 TAGE'
date: '2018-11-24'
math: true
tags:
    - 体系结构
    - 课程
categories:
    - 课程
---

介绍一下强大的分支预测器 TAGE ，没见到有人写相关的文章，大概都去读论文了吧。

这次分享的slides在[这里](https://ssine.cc/slides/TAGE.html)。

<!-- more -->

TAGE predictor 全称 TAgged GEometric history length branch predictor ，自从 2006 年在 CBP-2 上首次出现以来就没有离开过冠军榜单，后来几次比赛的获胜者都是 TAGE 预测器的各种变种。 TAGE 预测器是一个 PPM-like 预测器，最大的不同是改变了更新状态的方式。

TAGE 论文： A. Seznec, P. Michaud,  "[A case for (partially) tagged Geometric History Length Branch Prediction](http://www.irisa.fr/caps/people/seznec/JILP-COTTAGE.pdf)", Journal of Instruction Level Parallelism , Feb. 2006

PPM 论文： P. Michaud, "[A PPM-like tag-based predictor](http://www.jilp.org/vol7/v7paper10.pdf)" , Journal of Instruction Level Parallelism , April 2005

以上两篇论文涵盖了实现一个 TAGE 预测器所需的所有内容，这也是本文接下来所要讲述的。

# 分支预测器

首先解释一下分支预测器的概念。 现代处理器中通常有较深的流水线，在遇到分支指令时不预测会导致断流而产生严重的性能损失，即使是简单的预测总是跳转也会比等待分支结果出现后再填充更有效率。 分支预测器的成功率对处理器的性能有很大影响。

可以将分支预测器看作一个对象，它根据当前的 PC 值判断该指令是否为分支指令以及是否将要跳转 —— 处理器中，在上一条指令的译码阶段就要进行下一条指令的取指操作了，因此分支预测器所能得到的信息只有当前的 PC 值。 根据 PC 地址判断该指令是否为分支指令可以用相对独立的一张表来实现，因此主要考虑判断当前指令是否会跳转。

在分支预测器给出预测结果后，处理器会根据该结果继续执行，在跳转指令的结果被决定时，处理器将结果告知分支预测器，预测器对自己的状态进行更新。

因此，一个分支预测器对象有两个主要的接口：

1. 预测接口： 输入 PC ，输出预测结果。
2. 更新接口： 输入实际跳转结果，更新预测器。

# 部件及其功能

下面是 TAGE 预测器的各个部件及其对应的功能。

## 基础部件

### GHR

GHR: Global History Register 全局历史寄存器。 这是一个移位寄存器，长度按照预测器需求而调整。 每次有一个分支结果被决定， GHR 都会左移一位，并将最新的结果放在最右边一位。

### Saturate Counter

饱和计数器。 饱和计数器的取值范围为 $[0, n]$ ，超出范围的修改没有效果。 每次分支实际跳转 +1 ，分支实际不跳转 -1。 在获取预测时，饱和计数器的值 $>n/2$ 则预测为跳转， $<n/2$ 预测不跳转。

## 基础预测器 (Base Predictor)

基础预测器用于在标记预测器未命中的情况下给出一个默认预测结果。 在论文中基础预测器是一个简单的 2-bits 历史预测器。 预测器以 PC 值的低几位作为索引，每一个表项是一个 __饱和计数器__ ，标记该 PC 位置的指令跳转的情况。

## 标记预测器 (Tagged Predictor)

标记预测器也是一张表。 表项由 饱和计数器、标记、使用标记 组成。 使用 PC 值与 GHR 的哈希作为表的索引，表项中的标记则是 PC 与 GHR 的另一种哈希，这是一种防止碰撞的手段。 使用标记用来标志该表项是否有用，在更新状态时有使用到。

TAGE 预测器包含多个标记预测器，他们的区别是所使用的 GHR 历史长度不同。 假设有 $n$ 个预测器 $T_1, T_2, \cdots, T_n$，那么它们所使用的历史纪录长度组成一个等比数列 $L_n=a_0*q^{n-1}$ 。 举个例子，对于有四个标记预测器的 TAGE ，它们在进行索引与标记的哈希计算时所使用的 GHR 位可以是 $GHR[0:5], GHR[0:10], GHR[0:20], GHR[0:40]$ 。

# 获取预测

在获取预测的时候，查询标记预测器的所有表项，如果有命中，选取历史长度最长的表的结果。 如果所有的标记预测器都没有命中，使用基础预测器给出的结果。

# 更新状态

更新状态的逻辑较为复杂，这也是 TAGE 性能超过原始的 PPM-like 预测器的主要原因。

在获取预测时，可能有不止一张表命中，我们称最后采用的（亦即使用历史最长的）结果为 __预测值__，预测值的提供者为 __提供者__，称提供者未命中时会采用的结果为 __候选值__， 候选值的提供者为 __候选提供者__。

---

更新操作的具体步骤

1. 如果预测值与候选值不同，根据预测结果是否正确增加或减少候选提供者的 使用标记 。
2. 根据实际跳转结果更新提供者的饱和计数器。
3. 如果预测错误，根据下文规则分配标记预测器表项。

首先找到所有的，使用历史比提供者长的标记预测器的对应表项。

如果所有的标记预测器对应项都被占用，那么将它们的使用标记全部减一，操作结束。

如果只有一项为空，那么将它分配给当前的情况。

如果有多项为空，那么按照概率将当前情况分配给其中一个标记预测器。 规则是：假设有 $n$ 个预测器可以被分配，按照历史长度从小到大排序为 $1,2,\cdots,n$，那么第 $i$ 个预测器被分配的概率 $P(i) = 2P(i+1)$。

# 实现细节

使用 C++ 实现 TAGE 预测器时，逻辑部分的难度并不大，但是论文中比较模糊的一点是哈希函数的计算。

将较长的历史纪录与PC值哈希到一个小范围来当作索引/标记，论文中采用的方法是“折叠”操作。定义 $fold$ 函数为

$$
fold(H_h,m)=\underset{j\ge0}{\bigoplus}H_h\times2^{-jm}\,\mod 2^m
$$

这表示将一个大数 $H_h$ 折叠到只有 m 个二进制位的数值中。

## 索引哈希的计算

对于基础预测器来说，它的索引就是简单的 PC 后 n 位。

标记预测器的索引要复杂一些，假设一个索引的二进制长度为 $L_t$，那么索引的哈希值为

$$
(PC >> L_t)\oplus PC \oplus fold(H_h, m)
$$

其中 $H_h$ 为当前标记预测器使用的历史长度， $m$ 为索引长度。

## 标记哈希的计算

标记哈希的计算方式为

$$
PC\oplus fold(H_h,m)\oplus\left(2\times fold(H_h,m-1)\right)
$$

## 环形移位寄存器 CSR

为了计算上面这些哈希，我们需要一个高效的用硬件计算 $fold$ 函数的方法。 PPM 论文中指出这可以用环形移位寄存器实现，下面是我画的图解：

![CSR](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/CSR_shift.png)