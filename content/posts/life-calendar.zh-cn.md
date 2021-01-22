---
title: 人生日历
date: '2018-12-13'
---

[Life Calendar](/projects/calendar/)

最近花了点时间思考人生，感觉过去的有些事情，自己忘记了，就再也找不回来了。 很多事件都不是什么大事，也没有人替我记着，如果连我都忘了，那些事还有什么意义呢。 因此就想做一个东西，把过去每一天发生的事情，保存并清楚地展现出来。

<!-- more -->

用方格代表时间单位的想法是受到别人的启发产生的，源头大概可以追溯到[这篇文章](https://waitbutwhy.com/2014/05/life-weeks.html)。

我想要做的就是一个互动版的下图：

![1](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/Weeks-block-LIFE1.png)

首先进行设计。 为了方便平时记录，为一个事件分配一个条目，正好可以用 html 标签和标签属性来表示，如下:

```html
<!-- 
Usage:
    Put your events in divs.
    Event metadata are discribed in tag attributes, include:
        date: date for event in a day
        start & end: dates for events that stretch across several days
        credit: how much 'meaning' you give to that event (0-100, floating point)
        icon(one-day event only): use a special icon on that day
        color: use a special color on that day
    Event contents can contain html, but it's recommended to keep it simple.

Palette:
    Big event: #A349A4
    Base color: hue (0-360)
-->
<div class="base" credit="10" start="9/2/2013" end="6/6/2016" hue="192"><i>青岛二中</i></div>
<div class="base" credit="10" start="9/1/2016" end="8/31/2020" hue="160"><i>北邮</i></div>
<div start="6/7/2016" end="6/8/2016" color="#A349A4">高考</div>
<div start="6/9/2016" end="6/18/2016" color="#C8BFE7">一些大学自招</div>

<div date="9/1/2016" color="#A349A4">北邮-新生报到</div>

<div date="4/29/2017" credit="30">第一次错过高铁</div>
<div date="7/7/2017" credit="70">今天有雷暴</div>
<div date="10/15/2017" credit="60">张学友演唱会</div>
<div date="10/26/2017" credit="90">福州CCSP 铜奖</div>
```

还要考虑如何使用颜色进行表示，感觉人生可以分成许多阶段，例如小学、初中、高中。 这样的阶段还可以细分，比如我在两个小学上过学。 这样的划分中，会为一段时间奠定一个“基调”。 同时，每一天里发生的事情对人的重要程度是不同的，可以通过色彩的浓淡来进行表示。 有了以上两点，选用hsv色彩空间比较合适：

![2](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/1280px-HSV_color_solid_cone_chroma_gray.png)

“基调”使用 Hue 来表示，而重要程度使用 chroma(saturate浓度) 来表示。 定义一天里发生过所有事情的重要程度用 credit 来表示，这个数取值范围为 0 - 100 。 这样就可以建立重要程度到颜色的转换了。

接下来就是关于实现的问题。 有点老派地决定用裸 js 实现，当然也是处于尽可能减少依赖的目的，我希望这个东西能够存活的时间尽可能久，那么只使用 js 是比较好的选择。 CSS只提供了 hsl 模型，不过从 hsv 转到 hsl 很方便。

还有一点是性能的考量，由于一生中大约有 36500 天，那么要绘制三万多个 div 方格。 在操作 dom 的时候要注意尽可能减少触发浏览器重新绘制的次数。 从事件解析到对应的格子的方法比较简单，由于事件相对不多也就没有花心思优化。

最后写出来也就两百多行，效果还是不错的。 链接在文章头部。