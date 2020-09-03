---
title: 实例学习Metal-第七章 纹理贴图
date: 2020-02-16 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

在本章中，我们将学习mipmapping，这是一种在远处渲染纹理对象的重要技术。我们将找出为什么mipmapping很重要，它如何补充常规纹理过滤，以及如何使用blit命令编码器在GPU上有效地生成mipmap。

<!-- more -->

# Chapter7(第七章)

## Mipmapping(纹理贴图)

在本章中，我们将学习mipmapping，这是一种在远处渲染纹理对象的重要技术。我们将找出为什么mipmapping很重要，它如何补充常规纹理过滤，以及如何使用blit命令编码器在GPU上有效地生成mipmap。

### A Review of Texture Filtering(纹理过滤研究综述)

在上一章中，我们使用纹理过滤来描述当像素的屏幕空间大小与纹素的大小不同时，纹素应如何映射到像素。当每个纹素映射到多个像素时，这称为`放大率`，当每个像素映射到多个纹素时，这称为`缩小`。在Metal中，我们可以选择在这两种方案中应用哪种类型的过滤。

最近的过滤只选择最接近的纹理像素来表示采样点。这导致块状输出图像，但计算上便宜。线性滤波选择四个相邻纹素并生成它们的加权平均值。

在Metal中，我们指定过滤类型使用MTLSamplerDescriptor的minFilter和magFilter属性。

![](1584065223.png)
<center>图7.1：示例应用程序说明了各种mipmapping过滤器的效果。这里使用的mipmap是人工着色的，用于演示目的。</center>

### Mipmap Theory(纹理贴图原理)

名称“mipmap”来自拉丁语短语“multum in parvo”，大致意思是“很少”。这暗示了mipmap中的每个纹素都结合了几个在它上面的水平的纹素。在我们讨论如何构建纹理贴图之前，让我们花一些时间来讨论为什么我们首先需要它们。

#### The Aliasing Problem(锯齿问题)

你可能会认为，因为我们处理了纹理缩小和放大的情况，我们的纹理贴图应该是完美的，没有视觉伪像。不幸的是，当纹理缩小超过某个因素时，会产生一种邪恶的效果。

当虚拟相机跨越场景时，每帧使用不同的纹理像素集来确定包括远处对象的像素的颜色。无论选择何种缩小滤镜，都会发生这种情况。在视觉上，这会产生难看的闪光。问题基本上是对高频信号进行欠采样（在这种情况下信号是纹理）。如果有一种方法可以在采样过程中平滑纹理，我们可以换取少量的模糊度，以显着减少运动过程中的分解闪烁效应。

使用线性缩小滤波器和线性缩小滤波器结合mipmap之间的区别如下所示，以激发进一步的讨论。

![](1584065357.png)
<center>图7.2：基本线性滤波和mipmapped线性滤波的并排比较</center>

#### The Mipmap Solution(Mipmap解决方案)

Mipmapping是一种设计用于解决此混叠问题的技术。不是在运行中对图像进行下采样，而是在离线或加载时生成一系列预滤波图像（级别）。每个级别比其前身小两倍（沿每个维度）。这导致每个纹理的内存使用量增加33％，但可以极大地增强运动中场景的保真度。

在下图中，显示了为棋盘纹理生成的级别。

![](1584065467.png)
<center>图7.3：构成mipmap的各个级别。每个图像的大小是其前一个图像的一半，面积的1/4，低至1个像素。</center>

#### Mipmap Sampling(Mipmap采样)

当对mipmapped纹理进行采样时，片段的投影区域用于确定哪个mipmap级别最接近纹理的纹素大小。相对于纹素大小较小的片段使用已经减少到更大程度的mipmap级别。

在Metal中，mipmap过滤器与缩小过滤器分开指定，在描述符的mipFilter属性中，但min和mip过滤器交互以创建四种可能的场景。下面按照增加计算成本的顺序描述它们。

当minFilter设置为MTLSamplerMinMagFilterNearest并且mipFilter设置为MTLSamplerMipFilterNearest时，将选择最接近匹配的mipmap级别，并使用其中的单个纹素作为样本。

当minFilter设置为MTLSamplerMinMagFilterNearest且mipFilter设置为MTLSamplerMipFilterLinear时，将选择两个最接近匹配的mipmap级别，并从中获取一个样本。然后将这两个样品平均以产生最终样本。

当minFilter设置为MTLSamplerMinMagFilterLinear并且mipFilter设置为MTLSamplerMipFilterNearest时，将选择最接近匹配的mipmap级别，并平均四个纹素以生成样本。

当minFilter设置为MTLSamplerMinMagFilterLinear并且mipFilter设置为MTLSamplerMipFilterLinear时，将选择两个最接近匹配的mipmap级别，并对每个级别的四个样本进行平均以创建该级别的样本。然后将这两个平均样本再次平均以产生最终样本。

下图显示了使用线性最小值滤波器时使用最近和线性mip滤波器之间的差异：

![](1584065494.png)
<center>图7.4：使用线性最小值滤波器和最近的mip滤波器的效果。 Mip水平已经人工着色，以更清楚地显示不同的水平。</center>


![](1584065514.png)
<center>图7.5：使用线性最小值滤波器和线性mip滤波器的效果。 Mip水平已经人工着色，以更清楚地显示不同的水平。</center>

### Mipmapped Textures in Metal(Metal中的Mipmapped纹理)

在Metal中构建一个mipmap纹理是一个由两部分组成的过程：创建纹理对象并将图像数据复制到mipmap级别。 Metal不会自动为我们生成mipmap级别，因此我们将在下面介绍两种自行生成方式。

#### Creating the Texture(创建纹理)

我们可以使用相同的便捷方法来创建2D mipmapped纹理描述符，就像非mipmapped纹理一样，传递YES作为最终参数。

```objc
[MTLTextureDescriptor texture2DDescriptorWithPixelFormat:MTLPixelFormatRGBA8Unorm
width:width height:height mipmapped:YES];
```

当mipmapped参数等于YES时，Metal会计算描述符`mipmapLevelCount`属性。用于查找级别数的公式是floor（log2（max（width，height）））+ 1.例如，512×256纹理有10个级别。

一旦我们有了描述符，我们就从Metal设备请求一个纹理对象：

```objc
id<MTLTexture> texture = [device newTextureWithDescriptor:descriptor];
```

#### Generating Mipmap Levels Manually(手动生成Mipmap级别)

创建mipmap级别涉及创建基本图像的越来越小的版本，直到级别的维度仅为一个像素大小。
iOS和OS X共享一个名为Core Graphics的框架，该框架具有用于绘制形状，文本和图像的低级实用程序。生成mipmap级别包括使用CGBitmapContextCreate创建位图上下文，使用CGContextDrawImage将基本图像绘制到其中，然后将基础数据复制到Metal纹理的相应级别，如下所示：

```objc
MTLRegion region = MTLRegionMake2D(0, 0, mipWidth, mipWidth); [texture replaceRegion:region
mipmapLevel:level withBytes:mipImageData bytesPerRow:mipWidth * bytesPerPixel]
```

对于级别1，mipWidth和mipHeight等于原始图像大小的一半。每次围绕mipmap生成循环，宽度和高度减半，级别被添加，并且过程重复直到生成所有级别。
在示例应用程序中，色调颜色应用于使用Core Graphics生成的每个mipmap级别，以便可以轻松区分它们。下图显示了由Core Graphics生成的包含棋盘纹理的图像。

![](1584065574.png)
<center>图7.6：CPU生成的mipmap级别;在每个级别应用了一种色调颜色，以便在渲染时将其区分开来</center>

### The Blit Command Encoder(Blit命令编码器)

在CPU上生成mipmap的主要缺点是速度。使用Core Graphics缩小图像比使用GPU要**容易十倍**。但是我们如何将工作卸载到GPU？令人高兴的是，Metal包含一种特殊类型的命令编码器，其作用是利用GPU进行图像复制和调整操作大小：blit命令编码器。术语“blit”是短语“block transfer”的衍生词。

#### Capabilities of the Blit Command Encoder(Blit命令编码器的功能)

Blit命令编码器支持GPU资源（缓冲区和纹理）之间的硬件加速传输。 blit命令编码器可用于填充具有特定值的缓冲区，将一个纹理的一部分复制到另一个纹理中，并在缓冲区和纹理之间进行复制。

我们不会在本章中探讨blit命令编码器的所有功能。我们将使用它来生成mipmap的所有级别。

#### Capabilities of the Blit Command Encoder(使用Blit命令编码器生成Mipmap)

使用blit命令编码器生成mipmap是非常简单的，因为有一个名为generateMipmapsForTexture的MTLBlitCommandEncoder协议上的方法：调用此方法后，我们添加一个完成处理程序，以便我们知道何时
命令完成。这个过程非常快，在A8处理器上的1024×1024纹理大约需要1毫秒。
```objc
id<MTLBlitCommandEncoder> commandEncoder = [commandBuffer blitCommandEncoder]; 
[commandEncoder generateMipmapsForTexture:texture];
[commandEncoder endEncoding];
[commandBuffer addCompletedHandler:^(id<MTLCommandBuffer> buffer) {
// texture is now ready for use }];
[commandBuffer commit];
```
调用完成块时，纹理已准备好用于渲染。

### The Sample App(示例应用)

本章的示例代码位于07-Mipmapping目录中。

示例应用程序显示旋转的纹理立方体。您可以使用轻击手势在多种不同模式之间切换：无mipmapping，使用GPU生成的纹理进行mipmapping，使用CPU生成的纹理进行mipmapping和线性mip过滤，以及使用CPU生成的纹理和最近的mip过滤进行mipmapping。 CPU生成的纹理具有应用于每个级别的不同颜色的色调，以明确对哪些级别进行采样。

如果启用了mipmapping，则可以使用捏合手势将立方体缩放得更近和更远，这将导致对不同的mipmap级别进行采样。您还可以观察到当立方体接近边缘时，或者当它离开相机时不使用mipmap导致的降级。