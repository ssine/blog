---
title: "毕设论文的 Word 排版技巧"
date: 2020-05-22T18:22:38+08:00
---

毕设论文是用 Word 完成的，这里总结一下我是如何用 Word 实现一些相对复杂的排版需求的。

# 自动编号及其引用

## 章节标题

先来看标题要求：

* 一级标题：黑体三号，加粗
* 二级标题：黑体四号，加粗
* 三级标题及以下标题：黑体小四号，加粗

序号和标题之间间隔一格。 一级序号和标题居中；二级序号和标题顶格；三级及以下标题 “首行缩进” 2 字符。 其中一级标题的编号为 第x章，其他标题编号为 x.y.z 。

打开 Home -> Paragraph -> Multilevel List ，这里可以修改上述标题样式。

![1](./multilevel-list.png)

为一级标题的编号样式选择 一、二、三（简） ， 并把上面的格式改为 第 __一__ 章，后面按照要求加一个全角空格。 二级及以下的标题如法炮制，在编号中包括前面的标题编号。 需要注意的是，在更高级标题中包括第一级标题的时候，会变成 "一.1.2" 这样的滑稽样式，这时点击左下角的 More >> ，勾选右侧中间的 Legal style numbering 即可。

![1](./legal-style-numbering.png)

这之后点击 OK ，打出标题后对齐应用 Heading 样式（Home -> styles -> Heading x）， Word 就会自动识别并编号了。

## 图表

在插入图表之后，右键 Insert Caption 就可以插入图表标题了。 Label 新建一个 "图" ，然而由于一级标题编号是中文，只能显示 图 一-1 ，而我们想要的是 图 1-1 ，但是 Word 并没有为图表标题提供 Legal style numbering 选项。 经过查找，这个问题有人给出了解决方案（[链接](https://superuser.com/questions/863715/in-ms-word-how-can-i-make-caption-of-an-image-like-2-1-instead-of-ii-1-if-h
)），其实图表标题的自动编号只是域代码的一个应用，按 Alt + F9 可以在显示域代码/显示代码结果的状态切换。 只要在每一章标题处定义一个变量，之后以这个变量为基准进行编号就可以了。

这是默认的图表标题域代码：

```word
图 { STYLEREF 1 \s }-{ SEQ Figure \* ARABIC \s 1 } 图名
```

我们在每一章的标题处定义 Chap （在 Alt+F9 状态下操作，通过 Insert -> Quick Parts -> Field 插入域代码）：

```word
{ SEQ Chap \h }
```

之后把图表标题改为：

```word
图 { SEQ Chap \c }-{ SEQ Figure \* ARABIC \s 1 } 图名
```

就可以实现正常编号了。 之后新的图表标题可以直接复制上面的这一个，然后在数字编号处右键 -> Update Field 就可以更新这一处的数字。

需要注意的一点是，如果对图表的改动导致很多地方的数字需要更新，不需要每一个都右键更新，直接 Ctrl + A 选中整个文档，按 F9 就可以了。

![1](./field-code.png)

## 引用

这里介绍如何引用有编号的对象，包括章节、图表和参考文献（参考文献就是放在最后的一个列表）。 光标放在要插入的地方，选择 Insert -> Links -> Cross-reference 打开交叉引用框。

要引用章节，在左上角 Reference Type 中选择 Heading ，引用图标也可以选择对应的类型。 参考文献没有专门的类别，选择 Numbered Item 就可以了。

像图标一样，在引用数字出现变化需要更新时，单个变化可以直接右键更新，很多处引用要更新的时候使用 Ctrl + A 全选后按 F9 即可。

# 页码

通过 Insert -> Footer 插入页码。 按照要求，摘要、目录与正文使用不同样式的页脚。 以摘要与目录的分隔为例，通过 Layout -> Breaks -> Next Page 插入一个 section break 之后，在目录所在页双击页脚进入页脚编辑模式，点击 Design -> Navigation -> Link to Previous 来取消页脚与上一节的关联。

# 目录

在需要插入目录的地方选择 References -> Table of Contents -> Custom Table of Contents 打开目录样式面板，点击 Modify 可以修改目录显示的层级以及每一级的样式，还有页码以及标题-页码之间点的样式，最后点 OK 就能插入目录了。

# 善用 style

Word 中已经预定义了很多样式，例如各级标题、正文、图表标题。 在实际编辑时不要手动为每一段话设置“宋体、小四”这类样式，直接应用“正文”样式。 这样，在修改样式的时候只需要在 Home -> Style 里对应的样式上右键 Modify 就可以修改这个样式，而且对所有的内容都生效。

# 数学公式

习惯了 LaTeX 的数学语法，再用 Word 里的 Equation 编辑数学公式的时候就很难受，输入一个 $\alpha$ 都要在符号表里找半天。 好在有一个 Chrome 插件 [LaTeX2Word-Equation](https://chrome.google.com/webstore/detail/latex2word-equation/oicdodhdflfciojjhbhnhpeenbpfipfg) ，把 LaTeX 数学代码粘贴到浏览器随便什么页面的输入框里，选中，右键 LaTeX2Word-Equation 就可以把对应的 Word 格式公式放到剪贴板，在 Word 中直接粘贴就行了，十分的方便。

# 伪代码

Word 没有提供伪代码编辑功能（类比 LaTeX algorithmicx 宏包），找来找去最终决定用 LaTeX 来完成。 直接在 OverLeaf 中编辑好要插入的伪代码，导出 PDF ，在本地用 InkScape 转换成 SVG 插入 Word 。 不用截图是因为矢量图缩放不失真。 在 OverLeaf 中编辑中文伪代码的链接： [🔗](https://www.overleaf.com/read/vdgjkddvzscm) 。

![1](./pseudo-code.png)

# 代码

插入带有行号和高亮的代码也比较麻烦。 我的解决方案是在 Insert -> Text -> Object -> Object 面板中，选择 OpenDocument Text ，然后会弹出一个子 Word 文档的编辑窗口，在这里把四个边距设为 0 ，在 VSCode 中复制代码（建议使用 Quiet Light 颜色主题），并粘贴到文档中。 这样解决了代码高亮的问题，至于编号，将所有代码选中，点击 Home -> Paragraph -> Numbering 来应用编号列表，拿来做行数没有任何问题。

![1](./code.png)

# 其他

最后聊一点小知识。

在 File -> Options -> Display 中勾选 Show all formatting marks 可以显示所有的空白字符与控制字符，可以帮助解决很多疑难杂症，以及帮助了解 Word 是如何排版的。

我在写 markdown 的时候习惯在中英文之间插入空格，因为渲染出来更好看。 Word 已经自动帮助处理了这一点，也就是增大 Asian text 与 Latin text 的间距，看起来更美观， LaTeX 也有这一功能。

毕设论文要求的宋体与 Times New Roman 是两个典型的衬线体，在线条的边角又装饰性形状，看起来更加精致美观。 黑体是一个典型的无衬线体，线条直来直去，末端无装饰，阅读时更加高效。
