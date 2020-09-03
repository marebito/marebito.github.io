---
title: 实例学习Metal-第六章 纹理
date: 2020-02-09 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

纹理是渲染的核心主题。虽然它们有许多用途，但它们的主要目的之一是为表面提供比单独使用顶点颜色更高的细节水平。使用纹理贴图允许我们基本上按像素指定颜色，大大提高了渲染图像的视觉保真度。

<!-- more -->

Chapter 6(第六章)

## Textures(纹理)

纹理是渲染的核心主题。虽然它们有许多用途，但它们的主要目的之一是为表面提供比单独使用顶点颜色更高的细节水平。使用纹理贴图允许我们基本上按像素指定颜色，大大提高了渲染图像的视觉保真度。

在接下来的几章中，我们将讨论与Metal中纹理的使用相关的几个主题。

在本章中，我们将讨论纹理映射，它可以帮助我们将虚拟角色带入生活。我们还将介绍采样器，它们可以强大地控制在绘制时如何解释纹理数据。一路上，我们将得到名为Spot的卡通牛的协助。

![](1584064666.png)
<center>图6.1：Meet Spot。 （型号由Keenan Crane提供）</center>

###  Textures(纹理)

纹理是格式化的图像数据。这使它们与缓冲区不同，缓冲区是非结构化的内存块。我们将在本章中使用的纹理类型是2D图像。尽管Metal支持其他几种纹理，但我们可以通过遍历纹理映射的示例来介绍几乎所有重要的概念。

### Texture Mapping(纹理映射)

纹理映射是将网格中的每个顶点与纹理中的点相关联的过程。这与包装礼物类似，因为2D纸张包装纸（纹理）符合3D存在（网格）。

纹理映射通常使用专门的编辑工具完成。将网格展开为平面图形（2D图形，其尽可能多地保持3D模型的连通性），然后将其覆盖在纹理图像上。
下图显示了奶牛的三维模型（其组成面的边缘突出显示）及其对应的纹理贴图。注意到的各个部分
牛（头部，躯干，腿部，角部）已经在地图上分离，即使它们连接在3D模型上。

![](1584064893.png)
<center>图6.2：如何打开一头牛。网格被展平为2D表面，以便更容易将纹理坐标指定给顶点</center>

### Coordinate Systems(坐标系统)

在Metal中，纹理的像素坐标系的原点与其左上角重合。这与UIKit中的坐标系相同。但是，它与OpenGL中的默认纹理坐标系不同，其中原点是左下角。
纹理坐标系的轴通常标记为s和t（或u和v），以区别于世界坐标系的x和y轴。

![](1584064942.png)
<center>图6.3：Metal的纹理坐标系的原点位于左上角。</center>

#### Pixel Coordinates Versus Normalized Coordinates(像素坐标与标准化坐标)

Metal足够灵活，允许我们指定像素坐标或标准化坐标中的纹理坐标。像素坐标范围从（0,0）到（width-1，height-1）。因此，它们取决于纹理图像的尺寸。归一化坐标范围从（0,0）到（1,1），这使得它们与图像大小无关。

我们选择在本章和本书的其余部分使用标准化坐标。

### Filtering(过滤)

纹理是由有限数量的像素（称为`纹素`）组成的离散图像。但是，在绘制时，可以以高于或低于其原始大小的分辨率绘制纹理。因此，重要的是能够确定纹理在其纹理像素之间应该是什么颜色，或者当许多纹理像素被碾压到相同的空间中时。当以大于其原始大小的大小绘制纹理时，这称为`放大`。在其原始分辨率下绘制纹理的逆过程称为`缩小`。

从纹理重建图像数据的过程称为过滤。Metal提供了两种不同的过滤模式：最近和线性。

最近（也称为“最近邻居”）过滤只是选择与请求的纹理坐标最接近的纹素。这具有非常快的优点，但是当纹理被放大时（即，当每个纹理像素覆盖多个像素时），它可能导致渲染图像看起来是块状的。

线性滤波选择四个最接近的纹素，并根据从采样坐标到纹素的距离产生加权平均值。线性滤波比最近邻滤波产生更加视觉上令人愉悦的结果，并且足够快速地实时执行。

### Mipmaps(纹理贴图)

当纹理缩小时，多个纹素可以与单个像素重合。即使是场景中的轻微运动也会出现闪烁现象。线性过滤对情况没有帮助，因为覆盖每个像素的纹理像素集在帧与帧之间变化。

减轻此问题的一种方法是将纹理图像预滤波为一系列图像，称为mipmap。序列中的每个mipmap都是前一个图像的一半，大小为1×1。当缩小纹理时，最接近分辨率的mipmap将替换原始纹理图像。

下一章将深入介绍Mipmapping。

### Addressing(寻址)

通常，当将纹理坐标与网格的顶点相关联时，这些值沿两个轴被约束为[0,1]。然而，这并非总是如此。也可以使用负纹理坐标或大于1的坐标。当使用[0,1]范围之外的坐标时，采样器的寻址模式生效。可以选择各种不同的行为。

#### Clamp-to-Edge Addressing(钳位到边缘寻址)

在Clamp-to-Edge寻址中，对于越界值重复沿着纹理边缘的值。

![](1584064983.png)
<center>图6.4：Clamp-to-edge过滤重复纹理图像的外边缘以获得越界坐标</center>

#### Clamp-to-Zero Addressing(钳位到零寻址)

在Clamp-to-Zero寻址中，越界坐标的采样值是黑色或清晰的，具体取决于纹理是否具有alpha颜色分量。

![](1584065000.png)
<center>Figure 6.5: Clamp-to-Zero过滤在纹理图像的边界之外产生黑色</center>

#### Repeat Addressing(重复寻址)

在重复寻址中，越界坐标围绕纹理的相应边缘并从零开始重复。换句话说，采样坐标是输入坐标的分数部分，忽略整数部分。

![](1584065021.png)
<center>图6.6：重复寻址通过重复对纹理进行平铺</center>

#### Mirrored Repeat Addressing(镜像重复寻址)

在镜像重复寻址中，采样坐标首先从0增加到1，然后减少回0，依此类推。这会导致纹理被翻转并在每个其他整数边界上重复。

![](1584065040.png)
<center>图6.7：镜像重复寻址首先重复围绕轴镜像的纹理，然后正常重复，依此类推</center>

### Creating Textures in Metal(在Metal中创建纹理)

在我们实际要求Metal创建纹理对象之前，我们需要更多地了解像素格式，以及如何将纹理数据添加到内存中。

#### Pixel Formats(像素格式)

`像素`格式描述了颜色信息在存储器中的布局方式。此信息有几个不同的方面：颜色`组件`，颜色组件`排序`，颜色组件`大小`以及是否存在`压缩`。
常见的颜色成分是红色，绿色，蓝色和alpha（透明度）。这些都可以存在（如RGBA格式），或者可以不存在一个或多个。在完全不透明的图像的情况下，省略了alpha信息。
颜色分量排序是指哪些颜色分量出现在存储器中：BGRA或RGBA。
颜色可以用任何精度表示，但是两个流行的选择是每个组件8位和每个组件32位。最常见的是，当使用8位时，每个组件都是无符号的8位整数，介于0和255之间。当使用32位时，每个组件通常是一个32位浮点，范围从0.0到1.0。显然，32位比8位提供更高的精度，但8位通常足以捕获颜色之间可感知的差异，并且从存储器的角度来看要好得多。
Metal支持的像素格式列在MTLPixelFormat枚举中。支持的格式因操作系统版本和硬件而异，请参阅“金属编程指南”以获取更多详细信息。

#### Loading Image Data(加载图像数据)

我们将使用UIKit提供的强大实用程序从应用程序包中加载图像。要从包中的图像创建UIImage实例，我们只需要一行代码：

```
UIImage *image = [UIImage imageNamed:@"my-texture-name"];
```

不幸的是，UIKit没有提供访问UIImage底层位的方法。相反，我们必须将图像绘制到Core Graphics位图上下文中，该上下文具有与所需纹理相同的格式。作为此过程的一部分，我们转换上下文（使用平移后跟刻度），以便生成的图像位垂直翻转。这会使我们图像的坐标空间与Metal的纹理坐标空间一致。

```objc
CGImageRef imageRef = [image CGImage];
// Create a suitable bitmap context for extracting the bits of the image NSUInteger width = CGImageGetWidth(imageRef);
NSUInteger height = CGImageGetHeight(imageRef);
CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
uint8_t *rawData = (uint8_t *)calloc(height * width * 4, sizeof(uint8_t)); NSUInteger bytesPerPixel = 4;
NSUInteger bytesPerRow = bytesPerPixel * width;
NSUInteger bitsPerComponent = 8;
CGContextRef context = CGBitmapContextCreate(rawData, width, height,
bitsPerComponent, bytesPerRow, colorSpace,
kCGImageAlphaPremultipliedLast | kCGBitmapByteOrder32Big); CGColorSpaceRelease(colorSpace);
// Flip the context so the positive Y axis points down CGContextTranslateCTM(context, 0, height); CGContextScaleCTM(context, 1, -1);
CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef); CGContextRelease(context);
```

此代码包含在MTKTextureLoader实用程序类中。

#### Texture Descriptors(纹理描述符)

纹理描述符是一个轻量级对象，它指定纹理的尺寸和格式。在创建纹理时，您需要提供纹理描述符并接收符合MTLTexture协议的对象，该协议是MTLResource的子协议。纹理描述符（纹理类型，尺寸和格式）上指定的属性在创建纹理后是不可变的，但只要新数据的像素格式与像素格式匹配，您仍然可以更新纹理的内容。接收纹理。
MTLTextureDescriptor类提供了几种用于构建常见纹理类型的工厂方法。要描述2D纹理，您必须指定像素格式，纹理尺寸（以像素为单位），以及Metal是否应分配空间来存储纹理的相应mipmap级别。

```
[MTLTextureDescriptor texture2DDescriptorWithPixelFormat:MTLPixelFormatRGBA8Unorm
width:width height:height mipmapped:YES];
```

#### Creating a Texture Object(创建一个纹理对象)

现在创建纹理对象非常简单。我们只需通过提供有效的纹理描述符来从设备请求纹理：

```objc
id<MTLTexture> texture = [self.device newTextureWithDescriptor:textureDescriptor];
 ```
 
#### Updating Texture Contents(更新纹理内容)

现在创建纹理对象非常简单。我们只需通过提供有效的纹理描述符来从设备请求纹理：

```objc
 MTLRegion region = MTLRegionMake2D(0, 0, width, height); [texture replaceRegion:region mipmapLevel:0 withBytes:rawData
bytesPerRow:bytesPerRow];
 ```

#### Passing Textures to Shader Functions(传递纹理到着色器函数)

现在可以在着色器中使用此纹理。要将纹理传递给着色器函数，我们在发出绘制调用之前将它设置在命令编码器上。纹理在参数表中有自己的插槽与缓冲区分开，因此我们使用的索引从0开始。

```objc
[commandEncoder setFragmentTexture:texture atIndex:0];
```

现在，此纹理可以绑定到着色器函数的参数列表中具有[[texture（0）]]属性的参数。

### Samplers(采样器)

在Metal中，采样器是一个对象，它封装了与读取纹理相关的各种渲染状态：坐标系，寻址模式和过滤。可以在着色器功能或应用程序代码中创建采样器。我们将在以下部分中依次讨论每个问题。

#### Creating Samplers in Shaders(在Shader中创建采样器)

我们将在片段函数中使用采样器，因为我们希望为渲染图像中的每个像素生成不同的颜色。因此，有时在片段函数中直接创建采样器是有意义的。
以下代码创建了一个采样器，它将使用重复寻址模式在标准化坐标空间中进行采样，并使用线性过滤：

```
 constexpr sampler s(coord::normalized, address::repeat, filter::linear);
```

着色函数本地的采样器必须使用`constexpr`进行限定。这个关键字是C++ 11中的新关键字，表示可以在编译时而不是运行时计算表达式。这意味着将只创建一个采样器结构，以便在函数的所有调用中使用。

坐标值可以是`normalized`或`pixel`。

地址的可能值是`clamp_to_zero`，`clamp_to_edge`，`repeat`，
`mirrored_repeat`。

过滤器的可能值最接近和线性'，以便在最近的位置之间进行选择邻居和线性过滤。

所有这些值都属于强类型枚举。强类型枚举是C ++ 11中的一项新功能，它允许对枚举值进行更严格的类型检查。省略值的类型名称是一个错误（即，你必须说`filter::linear`而不是简单的`linear`）。

可以按任何顺序指定参数，因为采样器的构造函数实现为可变参数模板函数（C++ 11的另一个新特性）。

#### Using a Sampler(使用一个采样器)

从采样器获取颜色很简单。纹理有一个示例函数，它采用一个采样器和一组纹理坐标，返回一种颜色。在着色器函数中，我们调用此函数，传入一个采样器和当前顶点的（插值）纹理坐标。

```
float4 sampledColor = texture.sample(sampler, vertex.textureCoords);
```

然后，采样的颜色可以用于您想要的任何进一步计算，例如照明。

#### Creating Samplers in Application Code(在应用程序代码中创建采样器)

要在应用程序代码中创建一个采样器，我们填写MTLSamplerDescriptor对象并要求设备为我们提供一个采样器（符合MTLSamplerState的对象）。

```objc
MTLSamplerDescriptor *samplerDescriptor = [MTLSamplerDescriptor new]; 
samplerDescriptor.minFilter = MTLSamplerMinMagFilterNearest; 
samplerDescriptor.magFilter = MTLSamplerMinMagFilterLinear; 
samplerDescriptor.sAddressMode = MTLSamplerAddressModeClampToEdge; 
samplerDescriptor.tAddressMode = MTLSamplerAddressModeClampToEdge;

samplerState = [device newSamplerStateWithDescriptor:samplerDescriptor];
```

请注意，我们必须单独指定放大和缩小过滤器。在着色器代码中创建采样器时，我们使用filter参数同时指定两者，但我们也可以分别使用mag_filter和min_filter来获得与上面相同的行为。

类似地，必须为每个纹理轴单独设置地址模式，而我们使用着色器代码中的唯一地址参数来完成此操作。

#### Passing Samplers as Shader Arguments(将采样器作为着色器参数传递)

传递采样器看起来与传递纹理非常相似，但采样器驻留在另一组参数表槽中，因此我们使用不同的方法将它们绑定到从0开始的索引：

```objc
[commandEncoder setFragmentSamplerState:samplerState atIndex:0];
```

我们现在可以通过在着色器代码中使用[[sampler(0)]]来引用此采样器。

### The Sample Project(示例项目)

示例项目位于06-Texturing目录中。它从前面的章节中大量借鉴，因此本章不再重申共享的概念。
主要的变化是使用纹理/采样器对来确定每个像素处的网格的漫反射颜色，而不是在整个表面上插入单个颜色。这使我们可以绘制我们的纹理牛模型，在场景中产生更大的真实感和细节感。

