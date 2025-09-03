---
title: Matplotlib科研绘图技巧
date: 2025-09-03 11:01:29 +0800
categories: [编程, 绘图]
tags: [matplotlib]
description: 一些绘制高质量科研图表的小技巧
---

Matplotlib一直是我科研绘图的主力工具，这么多年来积累了一些绘制高质量科研图表的小技巧，记录梳理如下。

## 字体兼容

字体方面的兼容问题主要包括Type 3字体以及字体一致性问题：
- Type 3字体：绝大多数出版商都要求提交最终版手稿时，PDF文件中不使用Type 3字体（包含位图的字体），因此在使用Matplotlib绘图时，要确保生成的PDF格式图表不使用Type 3字体
- 字体一致性：以IEEE的模板为例，手稿的正文字体通常是Times或者Times New Roman，Matplotlib的默认字体DejaVu Sans与其一致性不好，需要重新调整字体以及字号大小。STIX字体与Times系列正文字体相容性较好，且支持大多数符号。一些模板的正文字号大小为10pt，图表中的最大字号应该与此对齐，例如图表中的label字号，其余部分字体可以稍小一号，例如legend、tick等部分字号。

这些设置可以通过全局参数进行调整

```python
import matplotlib

# 避免Type 3 字体
matplotlib.rcParams["pdf.fonttype"] = 42
matplotlib.rcParams["ps.fonttype"] = 42
# 默认字体和字号
matplotlib.rcParams["font.family"] = "STIXGeneral"
matplotlib.rcParams["mathtext.fontset"] = "stix"
matplotlib.rcParams["font.size"] = 10
# 微调图表不同部分字号大小
matplotlib.rcParams["axes.labelsize"] = 10
matplotlib.rcParams["legend.fontsize"] = 9
matplotlib.rcParams["xtick.labelsize"] = 9
matplotlib.rcParams["ytick.labelsize"] = 9
```

### Type 42 字体

Type 1、Type 3、Type 42字体，都是PostScript字体类型。PostScript是由Adobe开发的一种页面描述语言，可以定义字体嵌入到文档中的方式，这些字体类型就对应着不同的嵌入方是。Type 1是很早被广泛使用的矢量字体格式；Type 3则允许使用完整的PostScript语言定义字形，但可能包含位图元素，导致显示或者打印效果不佳；Type 42字体是一个TrueType字体的PostScript包装器，允许将TrueType字体（ttf后缀）嵌入到PDF文件中，这类TrueType字体更加现代，主要由微软和苹果开发，缩放性和显示效果更好。

### LaTeX引擎

曾经尝试过将`text.usetex`设置为True，使用LaTeX引擎渲染字体，但经常遇到问题，而且似乎会导致`fonttype`设置失效。根据[官方文档](https://matplotlib.org/stable/users/explain/text/usetex.html)来看，LaTeX支持的字体种类较为有限。

## 图表大小与格式

绘制Matplotlib图表时，最关键的参数就是图表的尺寸大小，通常是由传入`subplots`或者`figure`函数中的`figsize`参数来指定。作为参考，单栏图表宽度大约在3.0-3.3 in（英寸），双栏宽度大约在6.8-7.0 in（英寸）。高度参数则主要取决于两个限制因素，一是图表内容，首先要确保视觉效果清晰；二是宽度与高度的比例，最佳的比例大约在16:9或者16:10，也就是(3.2, 1.8)或者(3.2, 2.0)左右（以单栏为例）。

此外，为确保满足出版印刷的要求，`dpi`参数通常设置为300。

## 布局自动调整

指定画布大小绘制出来的图片在保存时可能会遇到四周空白较多的问题，这在追求极致的手稿篇幅压缩时，可能会带来一定麻烦。

我之前最常使用的一套去除图片四周的空白的流程是：
- 调用`plt.tight_layout()`，调整子图之间的间距；
- 保存文件时，传入`bbox_inches="tight"`，去除四周的空白；
- 如有必要，再调用`pdfcrop`命令行工具（Tex Live通常包含该工具）去除空白。

这一流程的确能够确保去除图片四周的空白，但在插入LaTeX手稿时，会遇到字号一致性与视觉对齐效果冲突的问题。

具体来讲，在使用`bbox_inches="tight"`或者调用`pdfcrop`工具之后，得到的PDF文件的大小就已经与`figsize`参数指定的大小不同了，因为四周的空白被去除了。因此，即使在绘制不同图表的过程中指定了相同的宽度参数，最终得到的PDF文件的宽度仍不相同。这种情况下，在LaTeX中插入PDF文件时，不推荐使用`width=.85\linewidth`这类缩放命令，而应该使用`scale=0.8`，确保插入的多张图表之间字号的一致性。

使用`bbox_inches`加`scale`参数插入图片能在确保字号大小一致性，但却无法保证视觉上的对齐效果。假设用基本相同的设定，绘制了两张不同的图表，第一张图表A的纵轴刻度是类似1，2，...这样的较短的文本，而第二张图表B的纵轴刻度是1000，2000，...这样较长的文本。经过这一流程后插入到LaTeX手稿中的图表A和B，其纵轴位置是不对齐的（假设A和B均居中显示）。其根本原因在于，图表A和B的四周空白尺寸是不同的。若要追求视觉上的对齐效果，就必须确保二者裁剪后的PDF文件具有相同宽度。粗暴的“裁剪—缩放”会破坏字号大小的一致性。因此，我们陷入了字号大小一致性与视觉对齐效果无法兼得的境地。

最近发现了一个全新的解决方案，可以很好的避免这一问题。在创建`Figure`对象时，可以传入参数`layout="constrained"`来使用[受约束的布局方式](https://matplotlib.org/stable/users/explain/axes/constrainedlayout_guide.html)（与`tight_layout`指定的`tight`布局不同），例如`plt.subplots(layout="constrained")`，或者设定为默认参数`plt.rcParams['figure.constrained_layout.use'] = True`。

在使用constrained布局方式后，Matplotlib的引擎会进行更加有效的布局调整，无需调用`bbox_inches`以及`tight_layout`（如果调用了`tight_layout`就切换回了`tight`布局），即可获得大小尺寸与`figsize`一致且几乎没有四周空白的PDF文件，并且legend图例等元素的位置调整也会更加精准。

总结一句话：使用`plt.subplots(layout="constrained")`来获得更加有效的自动布局，并且不要使用`bbox_inches`和`tight_layout`。

## 配色选择

Matplotlib提供了很多对色盲友好、灰度兼容的配色方案，例如viridis，plasma，Okabe-Ito等。在同一幅图内绘制多条曲线时，可以使用不同的线型（`linestyle`）或者标记点样式（`marker`）加以区分。

绘制完成后，可以用浏览器打开PDF文件，并在CSS样式中插入以下代码查看灰度图
```css
#viewerContainer > #viewer > .page > .canvasWrapper > canvas {
	filter: grayscale(100%);
}
```

## 面向对象的编程接口

绝大多数的demo代码片段以及教程中，都会大量使用`matplotlib.pyplot`的直接接口，例如`plt.plot`等。这些接口能够满足基本的绘图需求，但对于复杂的图表，使用面向对象的编程接口的可读性更好，也更方便实现细粒度的调整。即先创建`Figure`和`Axes`对象，然后对`Axes`对象调用相应的接口。

### 栅格化

当图表中的数据点数量较多，例如达到几十万甚至更多，生成PDF文件的过程会非常缓慢，且文件体积巨大。此时可以在接口内传入`rasterized=True`将复杂元素栅格化（转换为位图），同时保持坐标轴、标签等矢量元素不变。

## 配置持久化

将上述提及的相关全局设置集中在一个Python脚本内可以实现配置的持久化，日后需要绘制图表时，只需要将配置文件导入到新的绘图脚本中即可。

此外，Matplotlib还支持样式表，可以将rcParams设置保存到单独的样式表文件中，例如`manuscript.mplstyle`文件：

```
# Font settings
font.family : STIXGeneral
mathtext.fontset : stix
font.size : 10
axes.labelsize : 10
legend.fontsize : 9
xtick.labelsize : 9
ytick.labelsize : 9
# PDF/PS font settings
pdf.fonttype : 42
ps.fonttype : 42
# Ticks xtick.direction : in
ytick.direction : in
# Layout
figure.constrained_layout.use : True
```
