---
title: Life Calendar
date: 2018-12-13
---

[Life Calendar](/projects/calendar/)

I have spent some time thinking about life recently. I feel that there are some things in the past. If I forget it, I will never find it again. Many events are not a big deal, and no one remembers for me. If I even forget, what are the meanings of those things? So I want to do something, save and clearly show what happened in the past every day.

<!-- more -->

The idea of ​​using squares to represent time units is inspired by others, and the source can be traced back to [this article](https://waitbutwhy.com/2014/05/life-weeks.html) .

What I want to do is an interactive version of the following image:

![1](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/Weeks-block-LIFE1.png)

Design first. To facilitate the usual recording, assign an entry to an event, which can be represented by the html tag and the tag attribute, as follows:

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

Also consider how to use color to express, feeling that life can be divided into many stages, such as elementary school, junior high school, high school. This stage can also be subdivided, for example, I have studied in two elementary schools. Such a division will lay a "keynote" for a period of time. At the same time, what happens in each day is different to people, and can be expressed by the shade of color. With the above two points, it is more appropriate to use the hsv color space:

![2](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/1280px-HSV_color_solid_cone_chroma_gray.png)

The "keynote" is represented by Hue, and the importance is expressed by chroma (saturate concentration). The importance of defining everything that happened in a day is represented by credit, which ranges from 0 to 100. This will create a degree of importance to color conversion.

The next question is about implementation. A little old-fashioned decision to use bare js to achieve, of course, is also in the least possible to reduce the dependence, I hope this thing can survive as long as possible, then only use js is a better choice. CSS only provides the hsl model, but it is convenient to go from hsv to hsl.

Another point is performance considerations. Because there are about 36,500 days in a lifetime, draw more than 30,000 squares of div. When operating dom, be careful to minimize the number of times the browser is repainted. The method of parsing from the event to the corresponding grid is relatively simple, and there is no optimization because the event is relatively small.

Finally, I wrote a litte more than two hundred lines, and the effect is good. The link is at the head of the article.
