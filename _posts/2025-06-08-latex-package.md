---
title: 一些科研写作LaTeX宏包
date: 2025-06-08 21:59:32 +0800
categories: [编程, LaTeX]
tags: [LaTeX]
description: ACM出版系统接受的LaTeX宏包介绍和反复跳入的坑
---

ACM的出版系统TAPS有一个[可接受的LaTeX宏包列表](https://authors.acm.org/proceedings/production-information/accepted-latex-packages)，在提交最终版时，手稿必须只使用其中的宏包。
最近研究了一下其中一部分宏包，感觉其中有一些宏包还是挺有用的，在此记录一下。

## AMS数学宏包

我不止一次地打开过一个[关于每个AMS宏包作用的讨论](https://tex.stackexchange.com/questions/32100/what-does-each-ams-package-do)，因为每次看完就忘记，大致关系如下：

- `amsmath`：提供各类数学公式的环境和命令；
- `amsthm`：主要用于定义类似定理、定义等环境；
- `amssymb`：提供了额外的数学符号，例如常用的`\mathbb`字体；
- `amsfonts`：提供了额外的数学字体，会被`amssymb`加载。

### 奇怪的指示函数\mathbb{1}

在使用`acmart`模板时，经常发现用`\mathbb{1}`来表示指示函数的时候会显示出另一个奇怪的符号，是`amssymb`宏包中的`\nVdash`符号。
在看其他人的手稿时我也发现过这个问题。
Stack Overflow上的[相关讨论](https://tex.stackexchange.com/questions/706273/why-mathbb1-looking-weird)中提到，`\mathbb`命令只提供了针对大写字母的黑板粗体字体，而数字不会被视为大写字母。
讨论中的一些相关链接提到了一些解决方法，例如使用`bbm`和`doublestroke`宏包来打印指示函数，但可惜这两个宏包都不在TAPS的可接受宏包列表中。
提到的另一种可以兼容于TAPS系统的解决方法是加载`stix2`字体中的指示函数符号，直接在preamble中添加:

```latex
\DeclareFontFamily{U}{stix2bb}{}
\DeclareFontShape{U}{stix2bb}{m}{n} {<-> stix2-mathbb}{}
\NewDocumentCommand{\indicator}{}{\text{\usefont{U}{stix2bb}{m}{n}1}}
```

## 伪代码宏包

LaTeX中有很多个名字中带有algorithm的宏包，都是用于展示伪代码和算法的，[相关的讨论](https://tex.stackexchange.com/questions/229355/algorithm-algorithmic-algorithmicx-algorithm2e-algpseudocode-confused)我也打开过不止一次，大致关系如下：

- `algorithm`：提供算法浮动体环境；
- `algorithmic`、`algorithmicx`：两种不同的伪代码排版系统，提供不同的命令来展示算法伪代码，其中`algorithmicx`是对`algorithmic`的增强，提供了更多的命令和功能，例如自定义命令，二者都需要与`algorithm`宏包配合使用；
- `algorithm2e`：提供了另一种算法伪代码排版方式，功能强大，不需要配合`algorithm`宏包使用；
- `algpseudocode`：自动加载`algorithmicx`宏包，为其定义了一种类似`algorithmic`的排版方式，与之类似的提供不同排版风格的宏包还有`algcompatible`等。

Overleaf的一篇[教程](https://www.overleaf.com/learn/latex/Algorithms)对这些宏包的使用有详细的介绍和例子，其中最有价值的部分直接指出了你应该如何使用这些包：“选择以下两种方式之一来编写算法伪代码，两种方式**不兼容**，不能同时使用”

- 选择`algpseudocode`、`algcompatible`、`algorithmic`中的一个来排版算法主体，并使用`algorithm`宏包来创建浮动体；
- 使用`algorithm2e`宏包。

ACM的TAPS系统只接受这些宏包中的`algorithm`、`algorithmic`和`algorithm2e`，所以我现在基本只使用`algorithm2e`宏包来展示伪代码，简单不用思考，不用去想我是不是还有个宏包没加载（很懒）。

## `acmart`中加载的一些宏包

`acmart`模板加载的一部分宏包提供了有别于其他模板的排版风格，我在使用IEEE等其他模板时也经常会使用它们：

- `balance`：用于在双栏文档中平衡两栏的内容，确保页面布局美观；
- `booktabs`：一种不同的表格排版风格方式，提供`\toprule`、`\midrule`和`\bottomrule`等命令来创建更美观的表格线条；
- `caption`：自定义图表标题的样式和格式，控制标题的字体；
- `cmap`：用于处理PDF文档中的字符映射，可以使得pdfLaTeX生成的PDF文档中的字符可以被搜索和复制；
- `float`：提供了更多的浮动体选项，例如`H`选项可以强制图表在当前位置显示；
- `framed`：创建带边框的文本框，一般用来写Research Question的Findings；
- `geometry`：用于设置页面的边距和布局，可以自定义页面大小、边距等，一般会议和期刊都会提供模板并且不允许修改；
- `graphicx`：插入图像，基本是必备的宏包；
- `hyperref`：用于创建PDF文档中的创建超链接和书签；
- `hyperxmp`：在PDF文件中嵌入作者、关键词、版权等元数据；
- `microtype`：提供微调排版的功能，可以改善字符间距调整和连字效果；
- `natbib`：对于`\cite`命令的重新实现，提供了更多引用样式，与大多数引用管理宏包兼容，会导致参考文献列表样式改变；
- `newtxmath`：提供与默认的Computer Modern不同字体的数学字体，这也是ACM模板和其他模板数学字体不同的原因；
- `pbalance`：基于`balance`宏包的增强版本，目的是加载宏包开箱即用，不需要额外TeX命令；
- `xcolor`：提供各种颜色支持。

## TAPS支持的一些其他宏包

- `appendix`：用于处理附录的格式和样式，可以创建附录章节并自动编号；
- `bm`：提供粗体数学符号的支持，可以使用`\bm{}`命令来加粗数学符号；
- `breakurl`：用于处理长URL的换行问题，确保在PDF中长链接可以正确显示；
- `cleveref`：提供智能引用功能，可以自动识别引用的类型（如图、表、章节等），并在引用时添加适当的前缀；
- `enumitem`:用于自定义列表的样式和格式，可以控制列表的缩进、标签等，经常在压缩篇幅时使用；
- `listings`：代码高亮，并且可以实现在代码中插入LaTeX命令改变字体颜色等功能；
- `makecell`：可以创建表格中的单元格，实现单元格内换行和对齐的功能；
- `mathtools`：提供了更多的数学符号，例如定义符号`\coloneqq`，以及一些数学公式的增强功能；
- `multirow`：在表格中创建跨行单元格，非常常用；
- `siunitx`：用于处理物理量和单位的宏包，提供了统一的格式和样式，我的使用场景一般是`\SI{64}{GB}`这种有点蠢的写法，还提供了一种新的表格对齐方式S，用于对齐表格中的数字；
- `soul`：提供删除线、下划线等功能，用于强调文本；
- `textcomp`：提供额外的文本符号和字体，例如bullet，copyright符号等。