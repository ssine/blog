---
title: '实现一个数据库'
date: '2019-01-24'
---

# 概览

下学期的小学期要做一个数据库管理系统 DBMS ，因为觉得很有意思所以打算提前写一写。

## 一些想法

首先是想试试 Typescript + Electron 的开发模式。 Javascript 几乎是我目前最喜欢的编程语言，语法类似于 C/C++ 写起来很舒服，没有 Python 那么让人不自在的缩进；还是一个真正的跨平台语言，有浏览器的地方就能跑，而且借助 Electron 可以轻松实现跨平台客户端，打起包来也比 Python / C++ 轻松很多；面向对象和函数式都有一定的支持，尤其是 Typescript 为它带来了静态类型检查；语言自带的单线程/事件机制使得它天生就是异步的，用来做用户界面很方便；新技术也使得 JS 的运行效率越来越高。 不过在这里选择用 JS 也有一定的缺点， JS 的生态主要是在前端，用它来做这种偏底层的系统可能会缺少一些包。 最终选择 JS 也是因为这只是一个课程设计，不会有很高的性能与可靠性需求。

其次是关于可视化的一些东西，我特别希望能把 DBMS 的工作过程直观的展示出来，尤其是 SQL 解析那一块，可以画抽象语法树之类的，还能把表结构、元信息等画出来，想到这些就有点激动，这是驱使我做它的一大动力。

或许项目完成之后可以不仅仅是一个课程设计，如果逻辑与可视化做得比较好完全可以拿给老师教学用，直观的展示 SQL 的解析、生成关系代数、优化的内容，还有很多东西可以做动态展示，比如多粒度锁、基于日志的恢复，当然这些是后话了。

最后，完成这个项目是一个渐进的过程（比如先支持一小部分表达式，后面发现需要再增加），但文章也这么写的话结构就比较乱，因此在项目过程中所有文章都会随时更新。

## 设计与评估

DBMS 还是一个相当大的系统，老师说到时候只让我们实现一小部分功能，不过我现在提前做就要自己规划要实现哪些部分了。 这样要列出所有的模块/功能，划分一下依赖关系，评估一下实现的方案与难度，最后决定要实现的部分。

下面是一个数据库系统结构的图：

![dbms](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/dbms-structure.svg)

### 数据结构

关系型数据库的表使用二维数组存储。 一个 database 包含很多张表，而一个 dbms 的实例可以包含许多个 database 。 这里的每一级都会有一些 metadata 需要存储，初步想法是 metadata 同样使用表结构存储，这样更方便为用户提供编辑界面。

### 文件系统

比较难受的一点是，我要实现的是一个玩具模型，必然不会有大量数据的需求，最多也就是 MB 级别的数据量，这时文件系统只是作为一个持久化方案，而不是以之为中心。 对于这么少的数据完全可以全部放入内存，文件管理与缓冲区管理就显得没有什么意义。 那么实现 buffer 机制实属费力不讨好了，所以先实现一个基于内存的数据库管理系统，数据的存取直接通过序列化与反序列化实现。

还有一点是，之所以索引会采用b+树实现，是因为b+树对磁盘操作更亲和，需要的读写比较小，既然所有内容都在内存里，用其它平衡查找树代替b+树没准反而更快。

### SQL 的解析与执行

DBMS 提供给用户的界面是 SQL (DDL + DML)，对一个小的 DBMS 来说，这也是最复杂的部分。 [SQL 文法属于 LALR(1) 文法](https://stackoverflow.com/questions/40448716/stuck-parsing-sql-select-statement-at-the-join-clause-bnf-grammar-included)，[这里](https://github.com/ronsavage/SQL)有几个标准 SQL 的 BNF 声明和 HTML 版本可以看一下，十分巨大。 要从文法生成 parser 的话，需要一个 parser generator ， Lex 与 Yacc 只能用于 C/C++ ，查了一下发现 JS 有个移植版，叫做 Jison (Bison in Javascript)，用起来没有什么问题。 完整的 SQL 太大了，它的语法制导定义也不是我一己之力能够完成的，所以要实现它的一个子集。 很幸运，我在一个烂尾的项目中找到一份[能用于 jison 的语法制导定义](https://github.com/agershun/WebSQLShim/blob/master/src/sqliteparser.jison)，要做的是在它的基础上进行精简或重构。

## 参考

在系统的结构设计上，参考了 redbase 数据库，它包含了文件系统、系统管理、语言执行、索引四个主要模块，在执行 SQL 时生成了物理计划，并没有逻辑计划， minidb 没有文件系统部分，索引还没有实现，其它的结构都与 redbase 相似。

物理计划的划分主要参考了 [SQL Server 的文档](https://docs.microsoft.com/en-us/sql/relational-databases/showplan-logical-and-physical-operators-reference?view=sql-server-2017)。

## 下一步

到目前为止，实现了SQL的解析、语义检查、执行功能，能够进行增删改查操作，可以创建、删除、切换数据库以及持久化数据。 不过只是实现了主要流程上的操作，如果有违规操作还是会漏洞百出。 下一步要做的是物理计划可视化，加入类型检查、数据完整性约束，再往后是索引。

目前的成果：

![cur](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/dbms_cur_fig2.png)

# 项目结构

项目结构与开发坏境搭建。 网上对于 Typescript + Electron 项目搭建的内容并不多，因此有必要单独描述一下。

项目采用 Client-Server 结构， Client 提供用户界面，并将用户输入通过 IPC 方式发往 Server ， Server 负责余下所有工作，并将结果作为字符串返回给 Client 。 Client 端实现了一个简陋的终端，使用了 xterm 库。

Server 端收到输入后，先通过判断第一个字符是否为 `.` 判断系统指令，如果是系统指令就调用 manager 模块中的代码修改/查询系统信息。 如果是 SQL 语句，就把 SQL 交由 Jison 生成的 parser 解析，拿到解析树。 之后根据语句类型调用 query 模块中的 semantic check 函数，它会检查解析树的合法性并补全语句中省略的列名。 语义检查结束后 query 模块负责将 SQL 语句转换为执行计划，所有的执行计划都在 plan 模块中。 最后，根据执行计划执行并将输出转换为字符串，返回给 Client 。

## 环境搭建

### Typescript + Electron

用这个技术栈的话想去搜索手把手的博客就比较困难了，需要对 Electron 、 Typescript 和 NPM 的工作方式有一定的了解。 应用可以打包发布（没有生产环境），所以下面只考虑开发环境。 在命令行使用 electron 的方式是 `electron [filename]` ，文件是 electron 的入口文件。 NPM 配置文件 `config.json` 里面的 `scripts` 脚本是在 NPM 自己的环境下运行的， `node_modules` 文件夹下的依赖会被加入环境。 Typescript 相当于在所有正常行为之前加了编译的一步，在设置里面指定 ts 文件所在目录和输出目录就好了。

```bash
npm init
npm i typescript -S
npm i electron --save-exact
```

electron的版本变化较快，因此存储特定版本比较好。 由于是单打独斗就不用TSLint了。

创建 `tsconfig.json` ，使得编译器 `tsc` 从 src 文件夹读取输入，并把输出设置到 js 文件夹。

这样， NPM 的运行脚本就是 `tsc && electron ./js/main.js`

### Terminal

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

### Parser

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

### Webpack

项目使用 npm 开发，但是发布的时候免不了要打包，就会用到 Webpack 。 不过这些无聊的事情等项目写完了再说吧。

# SQL 解析与执行

确定支持的 SQL 子集，并实现对输入的解析。

## 解析器

直接手写 SQL Parser 也不会很难，第一个单词就能确定语句的类型，后面可以直接套用语句模板，但是表达式部分想要解析还是有些麻烦。 为了方便和清晰，使用 Jison (js 实现的 Yacc) 来生成 js 的 SQL Parser 。

在第一篇文章提到了[用于 jison 的语法制导定义](https://github.com/agershun/WebSQLShim/blob/master/src/sqliteparser.jison)，这份定义有一千五百多行，想要读懂都很麻烦，所以我写了 python 脚本，通过 Graphviz 绘制 SQL 文法的树状图（由于文法中有递归定义，并不是一棵树，叫做依赖图好一点）。

![sql_all](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/sql_graph_all.svg)

上面是个矢量图，新标签页打开就可以尽情放大了。 直接看这一整张图还是很累，不过多亏了 Graphviz 强大的布局，我们可以一目了然的看出有哪些子语句并把它们拆分出来。 下面就是依次检查这些语句并对它们进行精简的过程了。

目前为止，已经实现了表的创建、删除以及数据的增删改查语句：

```yacc
sql_stmt
    : create_table_stmt
    | drop_table_stmt
    | insert_stmt
    | select_stmt
    | delete_stmt
    | update_stmt
    ;
```

除了 SQL 语句，还有一些系统命令来进行数据库的创建删除、系统管理等操作：

```text
.exit                        | 退出
.file [filename]             | 将文件内容作为输入
.createdb [database name]    | 创建数据库
.dropdb [database name]      | 删除数据库
.usedb [database name]       | 使用数据库
.showdb                      | 显示所有数据库
.showtb                      | 显示当前数据库所有表
```

为了方便与 SQL 区分，所有的系统指令都以 `.` 开始。

### CREATE TABLE 语句

先实现一个最基础的 CREATE TABLE 语句，完整性约束先省了。

```yacc
create_table_stmt
    : CREATE TABLE database_table_name LPAR column_defs RPAR
    ;

database_table_name
    : name DOT name
    | name
    ;

name
    : LITERAL
    ;

column_defs
    : column_defs COMMA column_def
    | column_def
    ;

column_def
    : name type_name
    ;

type_name
    : names
    ;
```

语义动作在代码仓库中可以看到，先试试解析的效果：

![1](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/dbms_im_fig1.png)

看起来没有问题。

### MiniDB 数据类型

支持哪些数据类型呢？ 标准的 SQL 声明中指定字符串的最大长度和数字的精度是为了方便存储空间的分配，用 js 搞那些纯粹是增加复杂度。

直接使用 js 原声数据类型最省事了，先这么干吧。

这里有三种数据类型： Number, String, Date.

虽然 Number 分为 int 和 float ，但是 js 要进行类型检查比较麻烦（不是 js 的基础数据类型），所以先不支持。 String 天生是不限长度的， Date 属于 js 的 object ，使用的时候需要写 `Date('yyyy/mm/dd')` 。

### INSERT 语句

简单的向表中插入一组数据，目前还没有做类型检查。

```yacc
insert_stmt
    : INSERT INTO database_table_name columns_par insert_values
    ;

columns_par
    :
    | LPAR columns RPAR
    ;

columns
    : columns COMMA name
    | name
    ;

insert_values
    : VALUES LPAR subvalues RPAR
    ;

subvalues
    : subvalues COMMA expr
    | expr
    ;

expr
    : literal_value
    | NULL
    ;

literal_value
    : NUMBER
    | STRING
    ;
```

### 表达式

表达式的基本单位有三个：字面量、 NULL 与名字，名字可以是 列名 或 表名.列名 或 数据库名.表名.列名 。以后名字代表这三种。

表达式依照返回值类型非为两种： 数据型与布尔型。

数据型表达式通过比较运算符连接称为布尔型运算符，布尔型运算符通过逻辑连接符连接还是布尔型。

```yacc
expr
    : literal_value
    | name
    | name DOT name
    | name DOT name DOT name

    | expr EQ expr
    | expr NE expr
    | expr GT expr
    | expr GE expr
    | expr LT expr
    | expr LE expr

    | expr AND expr
    | expr OR expr
    ;
```

对表达式的解释使用函数递归的方式，对于数值型直接求值，布尔型则递归求出左右子树的值后再求值。

### SELECT 语句

SELECT 语句是整个 SQL 中最复杂的部分了，一个典型的 SELECT 语句是这样的：

```sql
SELECT result_columns
FROM table1
(INNER) JOIN table2 ON join_condition
WHERE expression
```

创建执行计划时，先执行 FROM 从句里面的表连接操作，再将大表放到 WHERE 从句的表达式中过滤，最后用取出存在于 result_columns 里面的列。 目前还没有考虑优化。

```yacc
select
    : SELECT result_columns from where
    ;

result_columns
    : column_list
    | STAR
    ;

column_list
    : column_list COMMA name
    | name
    ;

alias
    :
    | name
    | AS name
    ;

from
    : FROM join_clause
    ;

join_clause
    : table_or_subquery
    | join_clause join_operator table_or_subquery join_constraint
    ;

table_or_subquery
    : database_table_name alias
    ;

join_operator
    : COMMA
    | join_type JOIN
    ;

join_type
    :
    | INNER
    | CROSS
    ;

join_constraint
    :
    | ON expr
    ;

where
    : WHERE expr
    |
    ;
```

### DELETE 与 UPDATE

DELETE 与 UPDATE 语句结构较为简单，里面的限定语句 where 已经在前面的声明中定义过了。

```yacc
delete_stmt
    : DELETE FROM database_table_name where
    ;

update_stmt
    : UPDATE database_table_name SET column_expr_list where
    ;

column_expr_list
    : column_expr_list COMMA column_expr
    | column_expr
    ;

column_expr
    : name EQ expr
    ;
```

## 执行计划

在 SQL Parser 的输出是语法树，之后程序将语法树转换为执行计划，并执行它，获得表项输出。 执行计划是一级很好的接口，每个计划代表一次操作，主要提供 get_next() 接口，用于获取下一个表项。 执行计划为程序提供了更好的模块化以及优化的余地，比如，在扫描一张表时，如果没有建立索引就只能用循环扫描计划，如果有索引就可以使用索引扫描计划来提升效率；在表连接时，可以通过二重循环、哈希、归并三种方式实现，程序可以根据实际情况选择更好的执行计划。

计划有两种，逻辑计划和物理计划，逻辑计划差不多就是关系代数表达式，主要用于优化，随后转换为物理计划实际执行。 物理计划主要参考了 [SQL Server](https://docs.microsoft.com/en-us/sql/relational-databases/showplan-logical-and-physical-operators-reference?view=sql-server-2017) ，挑选出了一下一些物理计划准备实现：

* compute_scalar
* table_filter

* hash_join
* merge_join
* nested_loop_join

* aggregate

* table_project

* table_insert
* index_insert
* clustered_index_insert

* table_delete
* index_delete
* clustered_index_delete

* table_update
* index_update
* clustered_index_update

* table_scan
* index_scan
* clustered_index_scan

目前已经实现了无索引的最基本的几个物理计划，足够支持当前的一些 SQL 语句了，等以后实现了索引再扩充。

# 持久化

持久化指将内存中的数据保存在磁盘中，以便下一次程序启动时恢复之前的状态。

## 实现方式

持久化必然要借助操作系统提供的文件系统，也就是把数据保存在文件中。 打开 SQLite 的单文件看了一下，除了表里面的数据都是乱码，说明它使用自己规定的格式进行保存，也就是在二进制层面上进行的。 NodeJS 虽然也提供了这么做的接口，但是没有必要设计精巧的存储格式，因为毕竟只是玩玩而已。 最简单的方式是直接将数据库对象序列化，以字符串的形式存到文件中。 读取的时候将文件以 JSON 形式解析就好了。

## 对象的持久化

在把一个对象序列化之后，对象持有的方法会丢失，再恢复时也就不是原来的对象了。 为了解决这个问题，只好手动为每个对象编写了从 json 读取数据的方法。 将对象与数据一起存储会对持久化造成不便，如果这两者分开就没有什么问题了，对象只存储数据，其他操作用独立的函数来进行，这可能是之后要重构的方向。
