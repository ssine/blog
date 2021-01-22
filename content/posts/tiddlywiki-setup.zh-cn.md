---
title: '搭建个人非线性笔记本 TiddlyWiki Server'
date: '2019-08-13'
---

# TiddlyWiki

[TOC]

[TiddlyWiki](https://tiddlywiki.com/) 是一个适合管理复杂信息的笔记应用，它有两点特性我非常喜欢：

## Philosophy of Tiddlers

我认为人们做笔记的目的有两种，一种是借助记笔记的输出过程加深对知识的理解，另一种是方便以后重用这些知识。 前者是一时的作用，只是出于这个目的的笔记也无需保存下来，而后者则是很重要的，需要有比较完备的形式将知识组织起来，方便以后的查阅。 TiddlyWiki 就是通过 Tiddler 来记录有意义的小到不可再分的信息，并通过它们之间的联系来表达更深层次的信息，以此达到对任意层次的知识的记录与检索。 正如其官网所言：

> The purpose of recording and organising information is so that it can be used again. The value of recorded information is directly proportional to the ease with which it can be re-used.
>
> The philosophy of tiddlers is that we maximise the possibilities for re-use by slicing information up into the smallest semantically meaningful units with rich modelling of relationships between them. Then we use aggregation and composition to weave the fragments together to present narrative stories.
>
> TiddlyWiki aspires to provide an algebra for tiddlers, a concise way of expressing and exploring the relationships between items of information.

## Self-contained

TiddlyWiki 是一个完整的应用程序，主要有两种表现形式，一个是独立的单个 html 文件，一个是包含有许多 Tiddler 文件和元信息的目录。 单个 html 文件包括了所有的程序逻辑和数据，目录模式需要 node 程序 `tiddlywiki` 作为服务端。 程序本身是开源的，它又将所有数据的管理权交给了用户，你只需要一个浏览器就可以访问这些数据。 不像其他的在线服务，就算几十年之后 --- 只要 Web 技术还在，你就能继续使用整个应用。

# 服务端部署

TiddlyWiki 有很多种使用方式，可以作为单个 html 使用，结合 TiddlyDesktop 做保存，并使用 pages 服务放到公网，或是使用服务端程序管理一个目录。 在实际使用过程中，单个 html 体验并不好，因为保存数据时 html 没有权限覆盖自己，只能通过下载新文件或是用其他应用程序的方式保存，在网络上访问时也会遇到要下载的文件过大的问题。 下面主要介绍通过服务器 serve 一个目录的部署方式。

## NodeJS 环境

TiddlyWiki 是 js 写的，需要的运行时环境是 node ，在 ubuntu 下安装 node ：

```bash
# install package manager
apt install npm
# instal node version manager
npm install -g n
# update nodejs to stable version
n stable
# change the registry
npm config set registry http://registry.npm.taobao.org/
```

## TiddlyWiki 服务端

安装好 node 之后使用以下指令安装 TiddlyWiki 服务端程序：

```bash
sudo npm install -g tiddlywiki
```

### 新建 TiddlyWiki Folder

```bash
tiddlywiki mywiki --init server
```

会在当前目录下新建一个名叫 mywiki 的目录，存放所有 wiki 的数据。 使用

```bash
tiddlywiki mywiki --listen
```

会启动 TiddlyWiki 的 Server 端，在浏览器就可以访问了。

关于指令的详细解释参考[官网](https://tiddlywiki.com/#WebServer)。 在官方的 [Github 仓库](https://github.com/Jermolene/TiddlyWiki5#installing-tiddlywiki-on-nodejs) 也有更详细的解释。

### 认证

使用服务端并放在公网时会有一个问题，就是任何人都可以修改你的 Wiki 。 我们希望能够通过用户名密码登陆的用户可以修改，但是未登录的用户只能查看，[这个功能]((https://tiddlywiki.com/#WebServer%20Authorization))在 [5.1.18 版本](https://tiddlywiki.com/#Release%205.1.18) 新增。 新建文件 users.csv ，包含所有用户的用户名和密码，形如：

```csv
username,password
jane,do3
andy,sm1th
roger,m00re
```

在启动服务器时，增加 `--listen credentials="/path/to/users.csv" "readers=(anon)" writers=jane,andy,roger` 即可实现匿名查看，登录修改的功能。

### Nginx, HTTPS

如果希望将 TiddlyWiki 部署在某个子目录（即不由 TiddlyWiki 监听 80 端口），就需要使用 Nginx 做反向代理，将请求转发到 TiddlyWiki 服务端。 下面以我的服务为例，我的主页是 ssine.cc ，希望 TiddlyWiki 运行在 wiki.ssine.cc 域名，就需要由 Nginx 将 wiki.ssine.cc 的请求转发到 TiddlyWiki 所在的端口。

修改 `/etc/nginx/sites-available/wiki.ssine.cc.conf` ：

```conf
server {
    listen 80;
    listen [::]:80;

    server_name wiki.ssine.cc;
    root /var/www/ssine.cc;

    index index.html;
    location / {
            try_files $uri $uri/ =404;
    }
}
```

之后重启 Nginx ，激活这个路由：

```bash
sudo ln -s /etc/nginx/sites-available/wiki.ssine.cc.conf /etc/nginx/sites-enabled/wiki.ssine.cc.conf
sudo nginx -t
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

获取 HTTPS 证书：

```bash
sudo certbot --rsa-key-size 4096 --nginx
```

之后修改 Nginx 设置，启用反向代理：

```conf
server {
    server_name wiki.ssine.cc;
    root /var/www/ssine.cc;

    index index.html;

    location / {
        proxy_pass http://localhost:1004/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
        client_max_body_size 0;

        access_log /var/log/nginx/wiki.access.log;
        error_log /var/log/nginx/wiki.error.log;
    }

    listen [::]:443 ssl http2; # managed by Certbot
    listen 443 ssl http2; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/nextcloud.dennisnotes.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/nextcloud.dennisnotes.com/privkey.pem; # managed by Certbot
    ssl_trusted_certificate /etc/letsencrypt/live/nextcloud.dennisnotes.com/chain.pem;
    include /etc/nginx/snippets/ssl.conf;
}
```

之后重复上面的步骤，重启 Nginx ，就可以将 wiki.ssine.cc 的流量转发至本地 1004 端口。

最后，在 1004 端口启动 TiddlyWiki Server，目录是 tw ：

```bash
tiddlywiki tw --listen host=127.0.0.1 port=1004 credentials="/path/to/users.csv" writers=sine,jill
```

还可以使用 nohup 指令让服务在后台运行：

```bash
nohup tiddlywiki tw --listen host=127.0.0.1 port=1004 credentials="/path/to/users.csv" writers=sine,jill > log.txt 2>&1 &
```

# 设置

下面介绍一些日常使用中对 TiddlyWiki 的调整。

## 添加 CSS

添加自己的 CSS 的方式很简单，创建一个新的 Tiddler ， Type 设为 text/css ，并加上 $:/tags/Stylesheet 的 Tag ，就会生效了。

### 故事河居中

显示所有 Tiddlers 的一列叫做故事河 Story River ，显示标题的一列叫做侧边栏 Side Bar 。官方有两种布局模式：固定故事河和固定侧边栏，但是固定故事河的话宽屏显示会很偏，固定侧边栏的话故事河又太宽，一口气读不完一行。 因此手动修改 CSS ，让故事河宽度固定并居中，侧边栏在故事河右边：

```css
@media (min-width: {{$:/themes/tiddlywiki/vanilla/metrics/sidebarbreakpoint}}) {

    html .tc-page-container {
        text-align: center;
    }

    html .tc-story-river {
        position: relative;
        width: 712px;
        /*width:1300px;*/
        padding-right: 0px;
                padding-left: 0px;
        margin: 0 auto;
        text-align: left;
    }

    html .tc-sidebar-scrollable {
        text-align: left;
        left: 50%;
        right: 0;
        margin-left: 350px;
    }

        html .tc-tiddler-frame {
                width: 100%;
        }
}
```

### 选中文本背景色

把选中的文本的背景改成黑色，没别的，就是觉得比较酷😎。

```css
::selection {
    background:#333; 
    color:#fff;
}

::-moz-selection {
    background:#333; 
    color:#fff;
}

::-webkit-selection {
    background:#333; 
    color:#fff;
}
```

### Tiddler 内容

修改列表的缩进，默认的缩进太多了，看着难受：

```css
ul{
  padding-left:0px;
  margin-left:25px;
}

ol{
  padding-left:0px;
  margin-left:25px;
}
```

文本间距微调，标题样式，水平线样式：

```css
.tc-story-river {
  word-spacing: 0.1em;
}

.tc-story-river h1 {
  text-align: center;
}

.tc-story-river h2 {
  text-align: center;
}

.tc-story-river h3 {
  text-decoration: underline;
}

hr {
  margin:0 auto;
  border: 0;
  height: 1px;
  background: #333;
  background-image: linear-gradient(to right, #333, #A349A4, #333);
}
```

### 新增字体

很多好看的字体并不 safe ，也就是说客户端上可能没有，为了保证展示效果，使用 font-face 添加字体，在客户端没有字体的时候会去 font-face 声明的地址下载。

```css
@font-face
{
font-family: Constantia;
src: url('https://source-bed.oss-cn-beijing.aliyuncs.com/fonts/constan.ttf');
}

@font-face
{
font-family: Constantia;
src: url('https://source-bed.oss-cn-beijing.aliyuncs.com/fonts/constanb.ttf');
font-weight:bold;
}

@font-face
{
font-family: Constantia;
src: url('https://source-bed.oss-cn-beijing.aliyuncs.com/fonts/constani.ttf');
font-style:italic;
}
```

这样就可以在设置里使用 Constantia 字体了。

## 插件

插件的主要作用是处理 Tiddler 文本的渲染，例如将 latex 数学公式渲染出来，或是把文本描述的流程图转换成 svg 。 安装插件的方式异常简单，把它的链接拖到 TiddlyWiki 的界面里然后导入就可以了。

### Latex

虽然 Web 端常用的 latex 解析器是 Mathjax ，但是 TiddlyWiki 上比较完善的插件只有 KaTeX ，我为了将只支持单页模式的 Mathjax 插件移植到服务端也花了不少功夫，后来觉得还是用 KaTeX 吧，自己维护一个插件太费心了。

### Mermaid

想在这边画流程图，已经有的插件叫做 Mermaid.js 但是布局非常丑。 也用了一段时间，后来记形式语言自动机笔记的时候希望能够在节点和边上显示 Tiddler 的链接，做了很多 Ugly Hack 。 现在已经忘了那时的语法了，自己写的代码也有点懒得看。 也是一种取舍，功能很多但是要多记些语法维护些东西，或者功能少表现形式简朴但是人比较轻松。 打算以后搞一个 viz.js 的移植，因为 Graphviz 排版很棒，不过 js 的文件有点大，要慎重考虑。 不过是否使用这种生成工具的主要因素是以后是否需要修改，否则在本地生成图片再上传就好了。

## 宏

TiddlyWiki 还支持宏定义，可以把一个宏（类似于函数）调用展开成一段文字。 创建宏的方式是新建一个 Tiddler ，把 Type 设置为 application/javascript ，并在最下方 Add a new field 增加 filed:  module-type， value: macro 。

在其它 Tiddler 中调用时使用 `<<macro-name param1 param2 ...>>` 语法。

### 嵌入 SVG

把 svg 嵌入到 html 中式很炫酷的，因为图片里的文字甚至还能被选中，因为经常使用就把这块包装成了宏：

```javascript
(function(){

/*jslint node: true, browser: true */
/*global $tw: false */
"use strict";

/*
Information about this macro
*/

exports.name = "svg";

exports.params = [
	{name: "src"},
    {name: "width"}
];

/*
Run the macro
*/
exports.run = function(src, width) {
	if(width[width.length-1] == "%") return '<center><embed src="' + src + '" style="width:' + width + ';" type="image/svg+xml" /></center>';
    else
	return '<center><embed src="' + src + '" style="width:' + width + 'px;" type="image/svg+xml" /></center>';
};

})();
```

参数是 svg 的 url 和宽度，使用时就像下面这样：

```tid
<<svg "https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/8086general-regs.svg" 450>>
```

### 防止图片缓存

有些外链图片会被缓存很久，如果希望每次打开页面时都会重新请求图片的话可以使用下面的宏：

```javascript
(function(){

/*jslint node: true, browser: true */
/*global $tw: false */
"use strict";

/*
Information about this macro
*/

exports.name = "nc-img";

exports.params = [
	{name: "url"}
];

/*
Run the macro
*/
exports.run = function(url) {
	return url + "?" + new Date().getTime();
};

})();
```

原理是每次在图片地址后增加 `?+随机数` ，这样浏览器就会认为是另一张图片重新请求。

使用方式（唯一用到的地方是更新欧拉计划的徽章 XD ）：

```tid
<img src=<<nc-img url:"https://projecteuler.net/profile/sineLiu.png">>/>
```
