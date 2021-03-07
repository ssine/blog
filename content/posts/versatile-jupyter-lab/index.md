---
title: "打造有多个内核的 Jupyter Lab 容器"
date: 2021-03-07T19:08:13+08:00
---

平时经常会有一些小的想法需要测试，比如看一下 TypeScript 能不能支持这个写法，用 Python 做两道题，快速处理一些文件。 这种时候在本地开一个编辑器写脚本再运行就比较繁琐，把各种语言的环境放在本机也比较乱且不便携，因此萌生了部署一个 Jupyter Lab 服务，可以随时随地通过浏览器写各种语言的代码的想法。

目前 Jupyter Lab 深受社区欢迎，已经有各种各样的语言实现了 Jupyter 的 Kernel ( 详见 [文档](https://github.com/jupyter/jupyter/wiki/Jupyter-kernels) ) 。 我挑选了个人比较常用的几个语言做了集成 ( 通常一个语言有好几个 Kernel 实现，挑选了其中比较完善的 ) ：

-   GoLang - [gophernotes](https://github.com/gopherdata/gophernotes)
-   TypeScript & JavaScript - [tslab](https://github.com/yunabe/tslab)
-   C++11/14/17 - [xeus-cling](https://github.com/jupyter-xeus/xeus-cling)
-   Racket - [iracket](https://github.com/rmculpepper/iracket)
-   Scala - [almond](https://github.com/almond-sh/almond)
-   Java - [IJava](https://github.com/SpencerPark/IJava)

目前这个镜像已经在 Docker Hub 上公开，使用方式：

```bash
docker run -d \
  --name=jupyter \
  -e LABUID=1000 \
  -e LABGID=1000 \
  -e LABPASSWD=userpassword \
  -e JUPYTER_TOKEN=token_to_jupyter_page \
  -p 80:8080 \
  -v /home/user/lab:/data \
  --restart unless-stopped \
  sineliu/jupyterlab-all-in-one:latest
```

![screenshot](screenshot.png)

## 一些实现细节

事情很简单，但是做起来有很多要解决的问题，下面简单聊一下。

### 指定 Jupyter Lab 密码

Jupyter Lab 通常会在启动后生成一个 Token，用来在浏览器登录，然而这个方式并不便于用户在一段时间后重新登陆 ( Token 容易丢 ) 。 有两个解决方案，一个是通过用户名密码登录，一个是指定 Token。 用户名密码登录的方式都需要比较麻烦的配置，好在 Jupyter Lab 支持通过环境变量 [JUPYTER_TOKEN](https://jupyter-notebook.readthedocs.io/en/stable/config.html?highlight=JUPYTER_TOKEN#options) 指定 Token ，这个东西很难搜，翻了半天文档才找到 ( 真的很久！ ) 。 这样就可以通过设置环境变量的方式固定一个方便记忆的 Token 了。

### 通过自定义用户避免 Docker Root 权限产生问题

Docker 容器默认以 root 用户运行，这导致在容器下面创建的文件 owner 都是 root。 不清楚 docker 有没有官方的解决方案，我最后是通过在容器第一次启动时创建用户来实现的。 这个过程分为两步，第一个时识别首次启动 ( 否则 restart 时再次创建用户会报错 ) ，第二个是用非交互的方式创建指定密码的用户。

一开始并没有什么首次启动脚本的思路，找了一圈发现可以把一个文件当作 flag，如果文件不存在就是首次，然后创建该文件，之后检测到文件存在，就知道不是首次启动了。

```bash
if test -e /tmp/fr; then \
  regular script \
else \
  touch /tmp/fr \
  initialization script \
fi
```

之后是非交互创建指定密码的用户，一般修改密码是交互式进行的 ( 需要用户手动在命令行输入密码 ) ，好在 `chpasswd` 命令支持流式输入，这样就可以用 `|` 符号设置密码了，镜像会读取三个环境变量：

-   LABUID: uid
-   LABGID: gid
-   LABPASSWD： 用户密码

```bash
groupadd -g ${LABGID} username
useradd -u ${LABUID} -g ${LABGID} -G wheel username
echo "username:${LABPASSWD}" | chpasswd
```

首先创建一个指定 group id 的 group，之后创建用户，并赋予指定的 user id 和 group id，同时添加到 wheel 用户组赋予用户 root 权限。 最后通过 echo + pip 的方式避免 chpasswd 要求交互式输入并设置密码。

### 与网络做斗争

国内的网络环境访问很多外网资源的时候会慢速、卡顿甚至中断。 这个问题在 Docker build 的过程中尤为明显，因为要下载很多资源，比较常用的资源通常有国内源，很多比较罕见的安装包就只能多试几次了。 如何在 `curl` 下载失败的时候不报错而是静默重试呢？ 可以用循环来完成：

```bash
for i in 1 2 3 4 5; do \
  curl -L https://git.io/vQhTU | bash && break \
  || sleep 1; \
done
```

整个循环会重试 5 次，一旦脚本正常执行没有报错，就会 break 退出循环，否则睡眠 1 秒后重试。

后来碰巧租了一个香港的服务器，访问国内国外的资源速度都快的飞起，也是个不错的解决方案 ( 就是要多花点钱 XD ) 。

### 上传 Docker Hub

上传到 Docker Hub 的过程比想象中要简单，首先注册一个账号，然后起个名字，创建仓库。在本地构建镜像的时候使用仓库的名字：

```bash
docker build -t sineliu/jupyterlab-all-in-one:latest .
```

之后

```bash
docker push sineliu/jupyterlab-all-in-one:latest
```

就可以了。
