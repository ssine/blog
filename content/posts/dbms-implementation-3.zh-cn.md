---
title: '实现一个数据库（三） SQL 解析与执行'
date: '2019-01-30'
dateformat: 'Y-m-d H:i'
tags:
    - 课程
    - 数据库
    - TypeScript
    - JavaScript
    - Electron
categories:
    - 项目
---

确定支持的 SQL 子集，并实现对输入的解析。

<!-- more -->

# 解析器

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

## CREATE TABLE 语句

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

## MiniDB 数据类型

支持哪些数据类型呢？ 标准的 SQL 声明中指定字符串的最大长度和数字的精度是为了方便存储空间的分配，用 js 搞那些纯粹是增加复杂度。

直接使用 js 原声数据类型最省事了，先这么干吧。

这里有三种数据类型： Number, String, Date.

虽然 Number 分为 int 和 float ，但是 js 要进行类型检查比较麻烦（不是 js 的基础数据类型），所以先不支持。 String 天生是不限长度的， Date 属于 js 的 object ，使用的时候需要写 `Date('yyyy/mm/dd')` 。

## INSERT 语句

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

## 表达式

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

## SELECT 语句

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

## DELETE 与 UPDATE

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

# 执行计划

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
