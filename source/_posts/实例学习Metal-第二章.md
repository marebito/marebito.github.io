---
title: 实例学习Metal-第二章 设置舞台并清除屏幕
date: 2020-01-12 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

本章介绍了在Metal中将屏幕清除为纯色所需的最低限度。即使这种简单的操作也需要金属框架所暴露的许多概念。以下章节将以此材料为基础，演示如何进行3D渲染等。

<!-- more -->

# Chapter2 (第二章)

## Setting the Stage and Clearing the Screen(设置舞台和清除屏幕)

本章介绍了在Metal中将屏幕清除为纯色所需的最低限度。即使这种简单的操作也需要金属框架所暴露的许多概念。以下章节将以此材料为基础，演示如何进行3D渲染等。

### Creating a New Project(创建一个新项目)

让我们在Xcode中创建一个新项目。我们更喜欢从单视图模板开始，因为它创建了一个视图控制器并将它连接到我们的窗口。

![](1584062478.png)
<center>图2.1：在Xcode中创建一个新的单视图项目</center>

### Interfacing with UIKit(与UIKit接口)

iOS上的每个UIView都使用Core Animation层作为其后备存储。换句话说，视图的图层包含在屏幕上绘制的实际内容。我们说这样的视图由CALayer实例支持。

您可以通过覆盖UIView子类上的`+layerClass`方法，告诉UIView更改它实例化为其后备层的图层类型。

您可以使用Xcode中的“File•New•File ...”菜单在您自己的项目中进行操作，并生成一个新的Cocoa Touch类，它是UIView的子类。我们称之为`MBEMetalView`。

![](1584062555.png)
<center>图2.2：在Xcode中添加一个新的UIView子类</center>

CAMetalLayer不是由Metal框架提供，而是由Core Animation提供。 CAMetalLayer是将UIKit和Metal绑定在一起的粘合剂，它提供了一些我们很快就会看到的非常好的功能。

让我们在UIView子类中实现+ layerClass，这样它就知道我们需要一个Metal层而不是一个CALayer。以下是完整的MBEMetalView实现：

```
@implementation MBEMetalView

+ (id)layerClass 
{
    return [CAMetalLayer class]; 
}

@end
```

将主故事板文件中的视图的自定义类更改为`MBEMetalView`。这将导致在加载故事板时实例化子类。这反过来为我们提供了一个适当的Metal层支持视图。

![](1584062589.png)
<center>图2.3：为视图控制器的根视图设置自定义类</center>

为方便起见，您可以添加一个属性到您的视图类，它是一个CAMetalLayer类型的。

```objc
@property (readonly) CAMetalLayer *metalLayer;
```

这可以防止您不得不重复从图层属性（即`CALayer`）的类型转换为实际的子类（`CAMetalLayer`），因为`CAMetalLayer`提供了一些在`CALayer`上找不到的方法。您可以按如下方式实现此属性：

```objc
- (CAMetalLayer *)metalLayer 
{
    return (CAMetalLayer *)self.layer;
}
```

如果您在设备上构建并运行此项目，您将只看到纯白色屏幕。要实际进行任何绘图，我们需要了解Metal设备以及它们帮助我们创建的所有其他对象。首先，关于在Metal中使用协议的一个词。

### Protocols(协议)

Metal API中的一个共同主题是使用协议而不是具体类来公开Metal功能。许多Metal API返回符合特定协议的对象，具体类型是次要的。这样做的好处是您无需关心实现该功能的确切类。

声明符合协议MTLDevice的对象的语法如下所示：

```objc
id <MTLDevice> device;
```

现在，我们来看看如何检索和使用设备。

### Devices(设备)

设备是GPU的抽象。它提供了创建命令队列，渲染状态和库等对象的方法。我们将很快查看其中的每一个。

Metal提供了一个C函数`MTLCreateSystemDefaultDevice`，它可以创建并返回一个满足我们需求的设备。此函数不带参数，因为没有可指定的设备属性。

我们的Metal层需要知道哪个设备将被渲染到其中。我们还需要在图层上配置像素格式，以便每个人都对其颜色组件的大小和顺序达成一致。 `MTLPixelFormatBGRA8Unorm`是一个不错的选择。使用此像素格式，每个像素将由蓝色，绿色，红色和alpha分量组成，每个分量将是一个8位无符号整数（介于0和255之间）。

在视图子类上创建设备属性很有帮助，因为我们需要设备在代码的其余部分为我们创建各种资源：

```objc
@interface MBEMetalView ()
@property (readonly) id<MTLDevice> device; 
@end
```

在我们使用Metal的函数之前，我们需要使用以下行导入Metal模块：

```objc
@import Metal;
```

以下是`-init`的完整实现，显示了所有必要的配置：

```
- (instancetype)initWithCoder:(NSCoder *)aDecoder 
{
    if ((self = [super initWithCoder:aDecoder])) 
    {
        _metalLayer = (CAMetalLayer *)[self layer];
        _device = MTLCreateSystemDefaultDevice(); _metalLayer.device = _device; 
        _metalLayer.pixelFormat = MTLPixelFormatBGRA8Unorm;
    }
    return self; 
}
```

请注意，我们覆盖`initWithCoder:`因为我们知道我们正在从故事板加载我们的视图。为了完整起见，我们还应该包含一个`init`的重写，以便我们可以以编程方式实例化该类。

### The redraw method(重绘方法)

在接下来的章节中，我们将绘图的责任委托给一个单独的类。目前，我们的视图类中的`-redraw`方法是我们发出绘图命令的地方。此外，我们不会重复重绘屏幕，只需将其清除一次为纯色。因此，只需调用`-redraw`就足够了。我们可以通过覆盖`-didMoveToWindow`方法来调用它，因为这个方法将在应用程序启动时调用一次。

```objc
- (void)didMoveToWindow 
{
    [self redraw]; 
}
```

重绘方法本身将完成清除屏幕所需的所有工作。本章其余部分的所有代码都包含在此方法中。

### Textures and Drawables(纹理和绘图)

Metal中的纹理是图像的容器。您可能习惯将纹理视为单个图像，但Metal中的纹理更抽象。 Metal还允许单个纹理对象表示图像`阵列`，每个图像称为`切片`。纹理中的每个图像都具有特定的大小和像素格式。纹理可以是1D，2D或3D。

我们现在不需要任何这些奇特的纹理。相反，我们将使用单个2D纹理作为我们的渲染缓冲区（即，实际像素被写入的位置）。此文本将与我们的应用程序运行设备的屏幕具有相同的分辨率。我们通过使用Core Animation提供的功能之一获得对此纹理的引用：`CAMetalDrawable`协议。

可绘制物(drawable)是由Metal层提供的物体，可以为我们提供可渲染的纹理。每次绘制时，我们都会向Metal图层请求一个可绘制对象，我们可以从中提取一个充当我们帧缓冲区的纹理。代码非常简单：

```objc
id<CAMetalDrawable> drawable = [self.metalLayer nextDrawable]; 
id<MTLTexture> texture = drawable.texture;
```

当我们完成渲染到纹理时，我们还将使用drawable向核心动画发信号，因此它可以在屏幕上显示。要实际清除帧缓冲区的纹理，我们需要设置一个渲染通道描述符来描述每帧采取的操作。

### Render Passes(渲染通道)

`渲染过程描述符`告诉Metal在渲染图像时要执行的操作。在渲染过程开始时，`loadAction`确定是否清除或保留纹理的先前内容。 `storeAction`确定了什么
渲染对纹理的影响：结果可以存储或丢弃。由于我们希望我们的像素在屏幕上结束，因此我们选择我们的商店操作为`MTLStoreActionStore`。

传递描述符也是我们选择在绘制任何几何图形之前屏幕将被清除的颜色的位置。在下面的情况中，我们选择不透明的红色（red= 1，green= 0，blue= 0，alpha = 1）。

```objc
MTLRenderPassDescriptor *passDescriptor = [MTLRenderPassDescriptor renderPassDescriptor];
passDescriptor.colorAttachments[0].texture = texture; 
passDescriptor.colorAttachments[0].loadAction = MTLLoadActionClear; 
passDescriptor.colorAttachments[0].storeAction = MTLStoreActionStore; 
passDescriptor.colorAttachments[0].clearColor = MTLClearColorMake(1, 0, 0, 1);
```

下面将使用渲染过程描述符来创建用于执行渲染命令的命令编码器。我们有时会将命令编码器本身称为`渲染过程`。

### Queues, Buffers, and Encoders(队列，缓冲区和编码器)

`命令队列`是一个对象，用于保存要执行的渲染命令缓冲区列表。我们只需询问设备即可获得一个。通常，命令队列是一个长期存在的对象，因此在更高级的场景中，我们将保留为多个帧创建的队列。

```objc
id<MTLCommandQueue> commandQueue = [self.device newCommandQueue];
```

`命令缓冲区`表示要作为一个单元执行的渲染命令的集合。每个命令缓冲区都与一个队列相关联：

```objc
id<MTLCommandBuffer> commandBuffer = [commandQueue commandBuffer];
```

命令编码器是一个对象，用于告诉Metal我们实际想要做什么的绘图。它负责将这些高级命令（设置这些着色器参数，绘制这些三角形等）转换为低级指令，然后将这些指令写入相应的命令缓冲区。一旦我们发出了所有的绘制调用（我们将在下一章讨论），我们将endEncoding消息发送到命令编码器，以便它有机会完成其编码。

```objc
id <MTLRenderCommandEncoder> commandEncoder = [commandBuffer renderCommandEncoderWithDescriptor:passDescriptor];
[commandEncoder endEncoding];
```

作为最后一个操作，命令缓冲区将发出信号，一旦所有前面的命令完成，其drawable将准备好显示在屏幕上。然后，我们调用commit来指示此命令缓冲区已完成并准备好放入命令队列以便在GPU上执行。反过来，这将导致我们的帧缓冲区填充我们选择的清晰颜色，红色。

```objc
[commandBuffer presentDrawable:drawable]; 
[commandBuffer commit];
```

### The Sample App(示例项目)

本章的示例代码位于02-ClearScreen目录中。

![](1584062640.png)
<center>图2.4：将屏幕清除为纯红色的结果</center>

### Conclusion(总结)

本章为更多令人兴奋的主题奠定了基础，例如3D渲染。希望你现在对使用Metal时会看到的一些对象有所了解。在下一章中，我们将介绍2D中的绘图几何。