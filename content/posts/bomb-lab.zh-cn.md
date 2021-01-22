---
title: 'Bomb Lab'
date: '2018-07-20'
---

这是汇编语言的第四个实验，没想到会用CSAPP上知名的bomb lab，做起来也是十分带劲呢。

<!-- more -->

# 将炸弹变成“哑弹”

作为一名玻璃心的完美主义者，自然是不希望自己的爆炸记录被登记在册，因此要想办法能够在本地调试炸弹。好在两边都是x86，至少二进制文件是能运行的。在本地linux环境下运行，报错：

```assembly
Initialization error: Running on an illegal host [2]
```

经过观察反汇编，以及给的c文件中有这么一行：

```c
/* Do all sorts of secret stuff that makes the bomb harder to defuse. */
initialize_bomb();
```

发现是这个函数在检查运行环境，看它的注释就不是好东西。用二进制编辑器把函数调用语句替换为空指令：

![replace-init](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/PAS20180720234228.png)

之后就可以在本地运行了，但是炸弹爆炸的信息还是会被发出去（上课的时候就不小心炸了），继续hack——

炸弹爆炸会调用explode_bomb函数，炸弹被拆除会调用phase_defused函数（编译的时候保留了符号表真的很良心了，大大降低了难度）；而这两个函数都会调用一个叫做send_msg的函数来传递信息。因此只需要将explode_bomb与phase_defused中的函数调用语句替换为空指令即可（函数有返回值，但是并没有没用到过，因此不详细追究了）：

![replace-call-msg](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/PAS20180720234058.png)

![replace-call-msg](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/PAS20180720234154.png)

大功告成，至此这个炸弹就变成了任人摆布的哑弹了。

# 工具简介

工欲善其事，必先利其器。本次实验中主要使用到了以下几个工具：

|工具|作用|
|:--:|:--:|
|gdb|动态调试|
|IDA|辅助阅读汇编代码，反编译|
|objdump|反汇编|
|Hex Editor Neo|编辑二进制文件|

# Phase 1

第一题很简单，题目实现了strings_not_equal函数，功能如其名，只需要知道传入参数在rdi、rsi，返回值为eax即可。根据地址找到参与比较的另外一个字符串`Verbosity leads to unclear, inarticulate things.`。

# Phase 2

第二题，有一个函数read_six_numbers，功能是从第一个参数指向的字符串中读出六个数字，保存在第二个int指针指向的数组中。注意函数调用sscanf时参数超过七个，使用堆栈传递了参数。这一点反编译工具IDA并没有意识到。

读完六个数之后，程序确认第一项是0，第二项是1，之后遵循斐波那契数的规律，每项是前面两项的和，因此答案为：

```number
0 1 1 2 3 5
```

PS 这里展示一下IDA的界面，能够分块显示跳转指令分开的程序块，大大提高了代码可读性：

![ida-tool](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/PAS20180721000135.png)

# Phase 3

第三题，跳转表，相当于一个switch语句。输入字符串是`"%d %c %d"`，以第一个数字作为索引，有0到7七种选择，每一种都有对应的字符与数字，这里为了方便直接取第一个数字为0，对应79H的字符为y，数字210H为528。因此一个可能的答案：`0 y 528`。

# Phase 4

这一关的输入是两个数字a、b，拆弹的条件为`a == func4(7, b)`且`2 <= b <= 4`。

通过研究func4的函数体得知这是一个递归函数，与其逻辑等价的python函数为：

```python
def func4(x, y):
    if x <= 0:
        return 0
    if x == 1:
        return y
    return func4(x-1, y) + func4(x-2, y) + y
```

取 b = 3，可以算出func4(7, 3) = 99，因此答案为`99 3`。

# Phase 5

本题要求读入一个字符串，长度为6。假设其中一个字符为c，后面的程序从1到6循环，将`array_3154[c & 0xF]`处的字符加到一个空字符串s上。那个数组的长度刚好为16，查看内存得知其内容为`maduiersnfotvbyl`。最后用s与目标串`"flyers"`比较。因此，构建一个字符串使得每个字符后四位的值为flyers对应位置的索引即可。见下图：

![ph5](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/bomb-phase-5.png)

一个可行的答案为：

`yonuvw`

# Phase 6

第六题实在是太绕了。。。

首先读入六个数（还是read_six_numbers函数），之后二重循环判定六个数不相同。（六个数的全排列是720，想办法暴力搜索也是可以的）

后面还有两个循环看得有点懵了，只能借助gdb动态调试。

![node](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/PAS20180721135950.png)

上图是循环中频繁涉及到的一个数组，经过观察，程序是新开了一个数组，按照原有的1到6将之对应为0x603490到0x6034e0。之后比较数组中如上图第一列所示的位置，必须是降序排列。因此将上图第一列所示数字排列，之后将每个数字对应的顺序（同上图第二列）作为答案即可。

```number
2 1 6 4 3 5
```

# Secret Phase

从各种地方都得知隐藏关的存在，比如c文件中的注释，以及已经破解了七关的排行榜。

直接查看函数列表便可发现有一个名叫secret_phase的函数。使用gdb在`return 0;`语句前加上断点，在前六关破解之后使用`call secret_phase`指令即可进入隐藏关。如果直接调用隐藏关的话排行榜会显示phase 1 invalid。

最后一关反而比较简单，读入一个十进制数y，我们希望fun7函数的返回值为0。

![fun7](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/bomb-fun7.png)

虽然函数的定义比较奇怪，但可以看出，想要函数返回值为0的话，只需要让y等于第一个参数的指针指向的数字就可以了，通过查看内存得知这个数字为36。