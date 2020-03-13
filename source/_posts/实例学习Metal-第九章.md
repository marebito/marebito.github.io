---
title: 实例学习Metal-第九章 压缩纹理
date: 2020-03-01 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

在本章中，我们将考虑几种GPU友好的压缩纹理格式。这些格式允许我们交换一些图像质量，以显着改进磁盘使用和性能。特别是，我们将研究ETC2，PVRTC和ASTC格式。

<!-- more -->

# Chapter9(第九章)

## Compressed Textures(压缩纹理)

在本章中，我们将考虑几种GPU友好的压缩纹理格式。这些格式允许我们交换一些图像质量，以显着改进磁盘使用和性能。特别是，我们将研究ETC2，PVRTC和ASTC格式。

### Why Use Compressed Textures?(为什么使用压缩纹理？)

在前面的章节中，我们通过将图像加载到内存中，将其绘制到位图上下文中，并将这些位复制到Metal纹理中来创建纹理。这些纹理使用未压缩的像素格式，MTLPixelFormatRGBA8Unorm。

Metal还支持多种压缩格式。这些格式允许我们将图像数据直接复制到纹理中而不对其进行解压缩。这些格式设计为在采样纹理时而不是在加载纹理时由GPU按需解压缩。

我们将在下面考虑的格式是有损的，这意味着它们不能完全保留图像内容，像PNG一样的无损格式。相反，他们通过降低图像质量来减少内存使用量。因此，使用压缩纹理数据可以大大减少采样纹理时消耗的内存带宽，并且对缓存更友好。

![](1584066690.png)
<center>图9.1：示例应用程序展示了各种压缩纹理格式</center>

### A Brief Overview of Compressed Texture Formats(压缩纹理格式简介)

#### S3TC

S3TC，也称为DXT，是第一个获得广泛采用的压缩格式，因为它分别在1998年和2001年被包含在DirectX 6.0和OpenGL 1.3中。虽然S3TC在桌面GPU上得到广泛支持，但它在iOS设备上不可用。

#### PVRTC

PVRTC图像格式由Imagination Technologies引入，Imagination Technologies是PowerVR系列GPU的创建者，是每个iOS设备的核心。这是Simon Fenney在2003年的一篇论文中首次描述的。

PVRTC通过将源图像下采样为两个较小的图像来进行操作，这两个较小的图像被放大并混合以重建原始的近似值。它一次考虑4×4或4×8像素的块，它们被打包成一个64位数量。从而，
每个像素分别占据4位或2位。

在iOS上使用PVRTC格式的一个重要限制是纹理必须是方形的，每个维度必须是2的幂。幸运的是，游戏纹理最常以与此限制兼容的尺寸生成。

#### ETC

爱立信纹理压缩（ETC）于2005年首次亮相。与PVRTC的每像素4位模式类似，它将每个4×4像素块压缩为单个64位数量，但缺乏对alpha通道的支持。后续版本ETC2增加了对1位和8位alpha通道的支持。所有与Metal兼容的硬件都支持ETC2，但PVRTC通常在可比较的文件大小下提供更好的质量。

#### ASTC

高级可扩展纹理压缩（ASTC）是Metal支持的最新压缩纹理格式。由AMD开发并在OpenGL ES 3.0级硬件上得到广泛支持，这种格式包含一个可选的块大小(从4×4到12×12)决定图像的压缩比。这种前所未有的灵活性使得ASTC成为一个非常有吸引力的选择，但它最低需要A8处理器，使其无法在iPhone 5s等设备上使用。

### Container Formats(容器格式)

纹理使用的压缩格式只是图片的一半。为了加载来自磁盘的纹理，我们需要知道它的尺寸（宽度和高度）及其像素格式。此元数据通常以头信息形式写入文件本身，该头信息位于图像数据之前。标题的布局由图像的容器格式描述。在本章中，我们将使用两种容器格式：PVR和ASTC。

#### The PVR Container Format(PVR容器格式)

不要与PVRTC压缩格式相混淆，PVR容器格式描述了如何解释纹理文件中的数据。 PVR文件可以包含多种格式的未压缩数据或压缩数据(包括S3TC，ETC和PVRTC)。

PVR格式有一些不同的，互不兼容的版本。这意味着每个版本的PVR都需要不同的代码才能正确解析。例如，“遗留”格式(PVRv2)的头信息结构如下所示：

```
struct PVRv2Header {
    uint32_t headerLength; 
    uint32_t height; 
    uint32_t width; 
    uint32_t mipmapCount; 
    uint32_t flags; 
    uint32_t dataLength; 
    uint32_t bitsPerPixel; 
    uint32_t redBitmask; 
    uint32_t greenBitmask;
    uint32_t blueBitmask; 
    uint32_t alphaBitmask; 
    uint32_t pvrTag; 
    uint32_t surfaceCount;
};
```
最新版本(PVRv3)的头信息结构如下所示：
```
struct PVRv3Header { 
    uint32_t version; 
    uint32_t flags; 
    uint64_t pixelFormat; 
    uint32_t colorSpace; 
    uint32_t channelType; 
    uint32_t height; 
    uint32_t width; 
    uint32_t depth; 
    uint32_t surfaceCount; 
    uint32_t faceCount; 
    uint32_t mipmapCount; 
    uint32_t metadataLength;
};
```

头信息之间存在更多相似之处。各个字段描述了包含的数据，包括纹理的尺寸以及文件是否包含一组mipmap或单个图像。如果文件包含mipmap，则它们只是连接在一起而没有填充。读取文件的程序负责计算每个图像的预期长度。本章的示例代码包括解析旧版和PVRv3容器的代码。

请注意，标题还包含surfaceCount和（在PVRv3的情况下）faceCount字段，可以在将纹理数组或立方体贴图写入单个文件时使用。我们不会在本章中使用这些字段。

#### The KTX Container Format(KTX容器格式)

KTX文件格式是由Khronos Group设计的容器，它负责许多开放标准，包括OpenGL。 KTX旨在尽可能无缝地将纹理数据加载到OpenGL中，但我们可以使用它将纹理数据轻松加载到Metal中。

KTX文件头看起来像这样：
```
struct KTXHeader 
{
    uint8_t identifier[12]; 
    uint32_t endianness;
    uint32_t glType;
    uint32_t glTypeSize;
    uint32_t glFormat;
    uint32_t glInternalFormat; 
    uint32_t glBaseInternalFormat; 
    uint32_t width;
    uint32_t height;
    uint32_t depth;
    uint32_t arrayElementCount; 
    uint32_t faceCount; 
    uint32_t mipmapCount; 
    uint32_t keyValueDataLength;
} MBEKTXHeader;
```

`identifier`字段是一个字节序列，用于将文件标识为KTX容器，并帮助检测编码错误。以gl开头的字段对应于OpenGL语言。我们关心的是`glInternalFormat`字段，它告诉我们使用哪个编码器（PVRTC，ASTC等）来压缩纹理数据。一旦我们知道了这一点，我们就可以将GL内部格式映射为Metal像素格式，并从文件的其余部分中提取数据。

有关此格式的所有详细信息，请参阅KTX规范（Khronos Group 2013）。

### Creating Compressed Textures(创建压缩纹理)

在命令行和GUI上都有各种用于创建压缩纹理的工具。下面，我们将介绍两种工具，它们允许我们在上面讨论的格式中创建纹理。

#### Creating Textures with texturetool(使用texturetool创建纹理)

Apple包含一个名为texturetool的Xcode命令行实用程序，可以将图像转换为压缩纹理格式，如PVRTC和ASTC。在撰写本文时，它不支持PVRv3容器格式。相反，它使用传统（PVRv2）格式写入PVR文件，如上所述。使用`-e`标识，指定所需的容器格式：PVR，KTX或RAW（无标题）。使用-f格式指定压缩类型：PVRTC或ASTC。 `-m`标识表示应生成所有mipmap并将其顺序写入文件。

每种压缩格式都有自己的标志，用于选择压缩程度。例如，要创建带有PVR遗留头的4位/像素PVRTC纹理（包括所有mipmap级别），请使用以下的texturetool调用：

```
texturetool -m -e PVRTC -f PVR --bits-per-pixel-4 -o output.pvr input.png
```

使用“thorough”压缩模式在具有8×8块大小的KTX容器中创建ASTC压缩纹理：

```
texturetool -m -e ASTC -f KTX --block-width-8 --block-height-8 \
--compression-mode-thorough -o output.astc input.png
```

#### Creating Textures with PVRTexToolGUI(使用PVRTexToolGUI创建纹理)

Imagination Technologies在其PowerVR SDK中包含一个工具，允许使用GUI创建压缩纹理。 PVRTexToolGUI允许您导入图像并选择各种压缩格式，包括ASTC，PVRTC和ETC2。与`texturetool`不同，此应用程序默认使用PVRv3容器格式，因此如果使用此工具压缩纹理，则应使用需要适当头信息格式的代码。

PVRTexToolGUI有一个命令行对应物`PVRTexToolCLI`，它在GUI中公开可用的所有参数，并允许您编写脚本以快速批量转换纹理。

![](1584066736.png)
<center>图9.2：Imagination Technologies的PVRTexToolGUI允许用户以各种格式压缩图像</center>

### Loading Compressed Texture Data into Metal(加载压缩纹理数据到Metal中)

加载压缩纹理是一个双重过程。首先，我们读取容器格式的头信息。然后，我们读取头信息描述的图像数据并将其交给Metal。无论使用何种压缩算法，我们都不需要自己解析图像数据，因为Metal能够在硬件中理解和解码压缩纹理。

使用压缩数据创建纹理与使用未压缩数据完全相同。首先，我们使用适当的像素格式和尺寸创建纹理描述符。如果纹理具有mipmap，我们使用mipmapped参数指示。例如，以下是我们如何为每个像素为4位的PVRTC纹理创建纹理描述符，我们有mipmaps：

```objc
 [MTLTextureDescriptor texture2DDescriptorWithPixelFormat:MTLPixelFormatPVRTC_RGBA_4BPP width:width height:height mipmapped:YES];
```

然后我们可以要求设备生成与此描述符匹配的纹理：

```objc
id texture = [device newTextureWithDescriptor：descriptor];
```

对于图像中的每个mipmap级别，我们需要单独调用MTLTexture方法-replaceRegion：mipmapLevel：withBytes：bytesPerRow :,如下所示：

```objc
MTLRegion region = MTLRegionMake2D(0, 0, levelWidth, levelHeight); [texture replaceRegion:region mipmapLevel:level withBytes:levelData
bytesPerRow:levelBytesPerRow];
```

其中levelWidth和levelHeight是当前mip级别的维度，level是当前mip级别的索引（基本级别是索引0），而levelData是指向mip级别的图像数据的指针。 levelBytesPerRow可以计算为图像数据的长度除以级别的高度。加载PVRTC数据时，bytesPerRow可以设置为0。

### The Sample App(示例应用)

本章的示例代码位于09-CompressedTextures目录中。

示例应用程序包括本章中提到的所有纹理格式的演示。点击“切换纹理”按钮会弹出一个菜单，允许用户在各种纹理中进行选择。如果运行应用程序的设备不支持ASTC压缩，则ASTC选项的菜单选项将不可用。

一般来说，ASTC在可用的硬件上提供最佳的质量/尺寸比。在使用A7处理器定位设备时，通常应首选PVRTC格式。

![](1584066772.png)
<center>图9.3：示例应用程序允许您从Metal支持的几种压缩格式中进行选择。</center>

