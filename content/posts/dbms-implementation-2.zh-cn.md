---
title: '实现一个数据库（二） 项目结构'
date: '2019-01-28'
tags:
    - 课程
    - 数据库
    - TypeScript
    - JavaScript
    - Electron
categories:
    - 项目
---

项目结构与开发坏境搭建。 网上对于 Typescript + Electron 项目搭建的内容并不多，因此有必要单独描述一下。

<!-- more -->

# 项目结构

项目采用 Client-Server 结构， Client 提供用户界面，并将用户输入通过 IPC 方式发往 Server ， Server 负责余下所有工作，并将结果作为字符串返回给 Client 。 Client 端实现了一个简陋的终端，使用了 xterm 库。

Server 端收到输入后，先通过判断第一个字符是否为 `.` 判断系统指令，如果是系统指令就调用 manager 模块中的代码修改/查询系统信息。 如果是 SQL 语句，就把 SQL 交由 Jison 生成的 parser 解析，拿到解析树。 之后根据语句类型调用 query 模块中的 semantic check 函数，它会检查解析树的合法性并补全语句中省略的列名。 语义检查结束后 query 模块负责将 SQL 语句转换为执行计划，所有的执行计划都在 plan 模块中。 最后，根据执行计划执行并将输出转换为字符串，返回给 Client 。

# 环境搭建

## Typescript + Electron

用这个技术栈的话想去搜索手把手的博客就比较困难了，需要对 Electron 、 Typescript 和 NPM 的工作方式有一定的了解。 应用可以打包发布（没有生产环境），所以下面只考虑开发环境。 在命令行使用 electron 的方式是 `electron [filename]` ，文件是 electron 的入口文件。 NPM 配置文件 `config.json` 里面的 `scripts` 脚本是在 NPM 自己的环境下运行的， `node_modules` 文件夹下的依赖会被加入环境。 Typescript 相当于在所有正常行为之前加了编译的一步，在设置里面指定 ts 文件所在目录和输出目录就好了。

```bash
npm init
npm i typescript -S
npm i electron --save-exact
```

electron的版本变化较快，因此存储特定版本比较好。 由于是单打独斗就不用TSLint了。

创建 `tsconfig.json` ，使得编译器 `tsc` 从 src 文件夹读取输入，并把输出设置到 js 文件夹。

这样， NPM 的运行脚本就是 `tsc && electron ./js/main.js`

## Terminal

找到一个更好用的命令行终端 hterm ，受[这篇文章](https://tbodt.com/hterm-xterm.html)推荐的。

DBMS的主要界面还是命令行，这里使用xterm。

```bash
npm i xterm -S
```

简单使用如下：

```js
import { Terminal } from 'xterm'

let term = new Terminal()
term.open(document.getElementById('terminal'))

term.on('key', (key, ev) => {
  term.write(key);
})
```

后面封装了一个好用的命令行，不再赘述了。

## Parser

parser generator 使用 Jison：

```bash
npm i jison -S
```

使用 Jison 生成 parser 的指令为：

```bash
jison ./ts/parser/sql.y -o ./ts/parser/parser.js
```

由于用ts进行编译，这个 js 文件并不会被放到输出文件夹（改后缀会导致编译不通过），因此要手动复制过去：

```bash
copy .\\ts\\parser\\parser.js .\\js\\parser\\parser.js /Y
```

这是 windows 下的命令。

## Webpack

项目使用 npm 开发，但是发布的时候免不了要打包，就会用到 Webpack 。 不过这些无聊的事情等项目写完了再说吧。
