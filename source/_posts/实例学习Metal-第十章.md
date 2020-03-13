---
title: 实例学习Metal-第十章 半透明和透明
date: 2020-03-08 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

到目前为止，我们在探索金属时已经轻易避免的一个主题是渲染不透明的材料。在本章中，我们将探讨一些实现透明度和半透明度的相关技术：alpha测试和alpha混合。

本章的示例场景是一片沙漠，里面有许多棕榈树和几个水池。棕榈树的叶子由几个多边形组成，纹理部分透明纹理，水通过alpha混合呈现为半透明表面，我们将在下面详细讨论。

<!-- more -->

# Chapter 10(第十章)

## Translucency and Transparency(半透明和透明)

到目前为止，我们在探索金属时已经轻易避免的一个主题是渲染不透明的材料。在本章中，我们将探讨一些实现透明度和半透明度的相关技术：alpha测试和alpha混合。

本章的示例场景是一片沙漠，里面有许多棕榈树和几个水池。棕榈树的叶子由几个多边形组成，纹理部分透明纹理，水通过alpha混合呈现为半透明表面，我们将在下面详细讨论。

### What is Alpha, Anyway?(无论如何，Alpha是什么？)

Alpha是颜色值的不透明度(或覆盖率)。 alpha值越高，颜色越不透明。 alpha值为0表示总透明度，而值为1(或100％)表示总不透明度。当根据片段颜色说话时，数据组件指示片段后面的场景显示的程度。请参阅(Glassner 2015)对基础知识的严格讨论。

![](1584066843.png)
<center>图10.1：示例应用程序演示了alpha测试和alpha混合</center>

### Alpha Testing(透明度测试)

我们将用于渲染部分透明表面的第一种技术是`alpha testing`。

透明度测试允许我们通过比较其不透明度与阈值(称为`参考值`)来确定片段是否应该对存储在渲染缓冲区中的颜色做出贡献。 透明度测试用于使表面`选择性`透明。

透明度测试的一个常见应用是渲染树叶，其中相对较少的多边形可用于描述树叶的形状，并且叶子纹理的alpha通道可用于确定绘制哪些像素和不绘制哪些像素。

![](1584066892.png)
<center>图10.2：透明度测试允许我们绘制部分透明的棕榈叶</center>

### Alpha Testing in Metal(Metal中的透明度测试)
Metal中的透明度测试是在片段函数中实现的。着色器文件包含全局参考值（示例应用程序中为0.5），我们的alpha值将与之进行比较。片段函数对棕榈树的漫反射纹理进行采样，对于应该是透明的纹素，其alpha值为0。然后使用采样的颜色来确定片段是否应该是可见的。

#### The ‘discard_fragment’ function(discard_fragment函数)

为了向Metal指示我们不想为片段提供颜色，我们不能简单地将返回的alpha值设置为0.如果我们这样做，片段深度仍会被写入深度缓冲区，从而导致任何几何隐藏在“透明”点之后。

相反，我们需要调用一个特殊的函数来避免完全为片段指定颜色值：`discard_fragment`。调用此函数可防止Metal将片段的计算深度和颜色值写入帧缓冲区，从而允许片段后面的场景显示。

##### The Texture Alpha Channel(纹理Alpha通道)

要执行每片段alpha测试，我们需要一个特殊构造的纹理，其alpha通道中包含合适的覆盖信息。下图显示了示例应用程序中使用的棕榈叶纹理。

![](1584066914.png)
<center>图10.3：用于在示例应用程序中绘制棕榈叶的纹理。不绘制图像的透明部分。</center>

##### Performing the Alpha Test(执行Alpha测试)

片段着色器中alpha测试的实现非常简单。将我们刚刚讨论的技术集合在一起，我们根据阈值测试采样的alpha值，并丢弃未通过alpha测试的任何片段：

```
float4 textureColor = texture.sample(texSampler, vert.texCoords);
if (textureColor.a < kAlphaTestReferenceValue) 
    discard_fragment();
```

#### A Note on Performance(关于绩效的说明)

使用`discard_fragment`会影响性能。特别是，它可以防止硬件执行称为`早期深度测试`（有时称为`早期z`）的优化。

通常，硬件可以在调用片段函数之前确定片段是否有助于渲染缓冲区，因为片段深度是在光栅化器中计算的。这意味着它可以避免着色已知被其他几何体遮挡的碎片块。

另一方面，如果要使用包含条件`discard_fragment`调用的函数对片段进行着色，则无法应用此优化，并且硬件必须为每个可能可见的片段调用着色器。

在本章的示例代码中，我们为几何体提供了单独的片段函数，这些函数使用alpha测试而不是。 alpha测试函数应仅用于实际需要它的几何体，因为过度使用`discard_fragment`会对性能产生很大的负面影响。

早期深度测试的完整解释超出了本书的范围，但Fabien Giesen的博客系列(Giesen 2011)提供了有关深度测试和与现代3D渲染管道相关的许多其他主题的更多详细信息。

#### Alpha Blending(透明度混合)

另一种实现透明度的有用技术是`透明度混合`。

通过在渲染缓冲区（目标）中已有的颜色和当前被遮蔽的片段（源）之间进行插值来实现透明度混合。使用的确切公式取决于所需的效果，但对于我们当前的目的，我们将使用以下等式：

c<sub>f</sub> =α<sub>s</sub> ×c<sub>s</sub> +(1−α<sub>s</sub>)×c<sub>d</sub>

其中c<sub>s</sub>和c<sub>d</sub>分别是源和目标颜色的RGB分量; α<sub>s</sub>是源颜色的不透明度;和c<sub>f</sub>是碎片的最终混合颜色值。

用文字表示，这个公式说：我们将源颜色的不透明度乘以源颜色的RGB分量，产生片段对最终颜色的贡献。这将添加到不透明度的加法倒数乘以渲染缓冲区中已有的颜色。这将“混合”两种颜色以创建写入渲染缓冲区的新颜色。

### Alpha Blending in Metal(Metal中的透明度混合)

在本章的示例项目中，我们使用alpha混合来使水面半透明。这种效果在计算上是便宜的并且不是特别令人信服，但它可以补充立方体图反射或在水面上滑动纹理或物理地移动它的动画。

### Enabling Blending in the Pipeline State(在管道状态中启用混合)

在Metal中混合有两种基本方法：固定功能和可编程。我们将在本节讨论固定功能混合。固定功能混合包括在渲染管道描述符的颜色附件描述符上设置属性。这些属性确定片段函数返回的颜色如何与像素的现有颜色组合以生成最终颜色。

为了使接下来的几个步骤更简洁，我们保存了对颜色附件的引用：

```
MTLRenderPipelineColorAttachmentDescriptor *renderbufferAttachment = pipelineDescriptor.colorAttachments[0];
```

要启用混合，请将blendingEnabled设置为YES：

```objc
renderbufferAttachment.blendingEnabled = YES;
```

接下来，我们选择用于组合加权颜色和透明度分量的操作。在这里，我们选择`Add`：

```objc
renderbufferAttachment.rgbBlendOperation = MTLBlendOperationAdd; renderbufferAttachment.alphaBlendOperation = MTLBlendOperationAdd;
```

现在我们需要指定源颜色和透明度的加权因子。我们选择SourceAlpha因子，以匹配上面给出的公式。

```objc
renderbufferAttachment.sourceRGBBlendFactor = MTLBlendFactorSourceAlpha; renderbufferAttachment.sourceAlphaBlendFactor = MTLBlendFactorSourceAlpha;
```

最后，我们选择目标颜色和透明度的加权因子。这些是源颜色因子的加法逆运算，OneMinusSourceAlpha：

```
renderbufferAttachment.destinationRGBBlendFactor = MTLBlendFactorOneMinusSourceAlpha;
renderbufferAttachment.destinationAlphaBlendFactor = MTLBlendFactorOneMinusSourceAlpha;
```

#### The Alpha Blending Fragment Shader(透明度混合碎片着色器)

片段着色器返回它为其片段计算的颜色。在前面的章节中，此颜色的alpha分量无关紧要，因为未启用混合。现在，我们需要确保着色器返回的alpha分量表示片段所需的不透明度。此值将用作在渲染管道状态上配置的混合方程中的“源”alpha值。

在示例代码中，我们将采样的漫反射纹理颜色与当前顶点颜色相乘，以获得片段的源颜色。由于纹理是不透明的，因此顶点颜色的alpha分量变为水的不透明度。将此值作为alpha的alpha传递片段颜色的返回值根据先前配置颜色附件的方式生成alpha混合。

![](1584067144.png)
<center>图10.4：Alpha混合允许沙漠沙漠通过清澈的水面</center>

#### Order-Dependent Blending versus Order-Independent Blending(依赖于顺序的混合与顺序无关的混合)

在渲染半透明表面时，顺序很重要。为了使透明度混合生成正确的图像，应在所有不透明对象之后渲染半透明对象。另外，相对于相机，它们应该从后端到前端。由于顺序取决于视图位置和方向，因此每当相对顺序发生变化时，通常每当相机移动时，都需要对对象进行重新排序。

我们在示例应用程序中完全避免了这个问题，因为在场景中只有一个凸出的半透明表面：水。其他应用程序不会那么容易。

在过去几年中，已经有很多关于避免这种昂贵的分选步骤的方法的研究。一种技术使用所谓的A缓冲（由Carpenter于1984年引入（Carpenter 1984））来维持对每个像素贡献的片段的一对颜色和深度值。然后将这些片段分类并在第二轮中混合。
最近，研究强调了alpha复合本身的数学，试图推导出一种与订单无关的透明度解决方案。有关此技术的进展，请参阅McGuire和Bavoil的论文(McGuire和Bavoil 2013)。

### 示例应用

本章的示例代码位于10-AlphaBlending目录中。按屏幕可使摄像机沿其当前航向前进，同时向左或向右平移会改变摄像机前进的方向。