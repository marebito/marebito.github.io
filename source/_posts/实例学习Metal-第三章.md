---
title: 实例学习Metal-第三章 2D绘制
date: 2020-01-19 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

基于我们在前一章中学习的渲染管道，我们现在将开始三维渲染。

<!-- more -->

# Chapter 3(第三章)

## Drawing in 2D(2D绘制)

在上一章中，我们了解了Metal框架的许多基本移动部分：设备，纹理，命令缓冲区和命令队列。虽然它引入了很多Metal的移动部件，但我们无法同时覆盖所有部件。本章将为渲染几何体时使用的Metal部分的讨论增加更多深度。特别是，我们将通过Metal渲染管道，介绍函数和库，并发出我们的第一个绘制调用。

本章的最终目标是向您展示如何使用Metal开始渲染实际几何体。我们绘制的三角形只能是2D。随后的章节将介绍绘制3D形状所需的数学，并最终为3D模型制作动画。

### Setup(设置)

MBEMetalView类的初始化程序已经过重构，可以调用一系列方法，这些方法将完成为使我们准备渲染所需的所有工作：

```objc
[self makeDevice]; 
[self makeBuffers]; 
[self makePipeline];
```

`-makeDevice`与以前在`-init`方法中直接包含的代码完全相同，正如我们在第2章中看到的：

```objc
 - (void)makeDevice 
 {
    device = MTLCreateSystemDefaultDevice();
    self.metalLayer.device = device;
    self.metalLayer.pixelFormat = MTLPixelFormatBGRA8Unorm; 
}

其他两种方法的定义将在以下部分中给出。

### Using Buffers to Store Data(使用缓冲区存储数据)

Metal提供了一个协议MTLBuffer，用于表示具有固定长度的无类型字节缓冲区。它的接口与NSData非常相似，它具有contents属性（如NSData的bytes属性）和`length`属性。但是，通过从Metal设备请求特定大小的缓冲区来创建Metal缓冲区。

我们将使用一个Metal缓冲区来存储我们想要绘制的顶点的位置和颜色。这称为`交错`缓冲区，因为位置和颜色数据被编织在一起成为一个连续的流，而不是为每个顶点属性指定单独的缓冲区。

我们定义一个结构来保持每个顶点的位置和颜色。这个结构的成员是`vector_float4`类型，它是Apple的SIMD（代表“单指令，多数据”）框架提供的类型。在某些情况下，SIMD类型可以比常规浮动阵列更有效地操作。金属着色器最自然地在SIMD数据上运行，因此我们也在客户端代码中使用它们。

这是我们的顶点类型，表示具有位置和颜色的顶点：

```objc
typedef struct 
{
    vector_float4 position;
    vector_float4 color; 
} MBEVertex;
```

我们的缓冲区将包含三个顶点，足以绘制一个三角形。顶点分别为红色，绿色和蓝色。由于这些值不会改变，我们静态声明它们，然后向Metal询问包含值副本的缓冲区：

```objc
- (void)makeBuffers 
{
    static const MBEVertex vertices[] = 
    {
        {.position={ 0.0, 0.5,0,1},.color={1,0,0,1}}, 
        {.position={-0.5,-0.5,0,1},.color={0,1,0,1}}, 
        {.position={ 0.5,-0.5,0,1},.color={0,0,1,1}}
    };
    self.vertexBuffer = [device newBufferWithBytes:vertices length:sizeof(vertices) options:MTLResourceOptionCPUCacheModeDefault];
}
```

您可能已经注意到每个顶点位置由四个组件组成，即使我们只关心x和y位置（因为我们绘制的是2D三角形）。这是因为Metal在4D齐次坐标中最自然地工作，其中每个点都有x，y，z和w坐标，w固定为1.我们使用齐次坐标，因为它们使得平移和投影等变换更自然。我们将在下一章中介绍这些转换。

为了简化我们需要在着色器中进行的数学运算，这些点在剪辑空间坐标中指定，其中x轴从左到右从-1到1运行，y轴从-1到1运行从下到上。

每种颜色都由熟悉的红色，绿色，蓝色和alpha组成。

现在，让我们看一下绘制时将处理这些数据的顶点和片段函数。

### Functions and Libraries(函数与库)

如第1章所述，现代图形API提供了“可编程管道”，这意味着它们允许GPU执行的许多操作由应用于每个顶点或像素的小程序指定。这些通常被称为“着色”。这是一个糟糕的名称，因为着色器不仅仅是计算像素的阴影（颜色）。核心Metal框架不使用该名称“着色器”，但在本文中我们将与“函数”互换使用。

要将Metal着色器代码合并到我们的项目中，我们需要向项目添加Metal源文件以包含将处理顶点和片段的着色器。

![](1584062747.png)
<center>图3.1：在Xcode中添加Metal Shader源文件</center>

以下是Shaders.metal的序文：
```
using namespace metal;
struct Vertex 
{
    float4 position [[position]];
    float4 color; 
};
```

我们定义了一个名为Vertex的结构，它看起来与我们在Objective-C客户端代码中创建的顶点结构非常相似。一个区别是每个成员都是float4类型而不是vector_float4。这些在概念上是相同的类型（四个32位浮点数的SIMD向量），但由于金属着色语言源自C ++，因此它的编写方式不同。金属着色器代码中的向量和矩阵类型属于simd命名空间，由着色器编译器推断，允许我们简单地将类型写为float4。

另一个区别是位置struct成员上存在[[position]]属性。此属性用于表示Metal应将哪个值视为顶点着色器返回的顶点的剪辑空间位置。从顶点着色器返回自定义结构时，结构的一个成员必须具有此属性。或者，您可以从顶点函数返回一个float4，它隐含地假定为顶点的位置。

Metal着色器函数的定义必须以`函数限定符`为前缀，该函数限定符是`vertex`，`fragment`或`kernel`之一。`vertex`函数限定符表示函数将用作顶点着色器，该几何体在几何体中的每个正在被绘制顶点上运行一次。在这种情况下，我们的顶点着色器名为`vertex_main`，它看起来像这样：

```
vertex Vertex vertex_main(device Vertex *vertices [[buffer(0)]], uint vid [[vertex_id]])
{
    return vertices[vid];
}
```

由于我们没有在顶点着色器中变换顶点的位置或颜色，我们可以简单地返回（通过复制）参数指定的索引处的顶点，其中[[vertex_id]]属性从0到2运行为为每个顶点调用顶点函数。如果我们正在做更复杂的工作，比如投影顶点或顶点照明，我们会在这里做。我们将在后续章节中看到更高级的顶点着色器。

由于我们绘制到屏幕的每个三角形可能覆盖许多像素，因此必须有一个管道阶段，该阶段采用顶点函数返回的值并对其进行插值，以便为当前三角形中的每个可能像素（片段）生成一个值。这个阶段称为光栅化器。光栅化器有效地“切割”三角形到它们的组成片段中，并为可能受当前三角形影响的每个像素的每个属性（在我们的例子中为位置和颜色）产生插值。然后将这些插入的值传递给片段函数，片段函数执行任何必要的每片段工作（例如纹理和每像素照明）。

片段函数的来源非常简短：

```
fragment float4 fragment_main(Vertex inVertex [[stage_in]]) 
{
    return inVertex.color; 
}
```

片段函数`fragment_main`采用Vertex限定的属性标识`[[stage_in]]`，它将其标识为每片段数据，而不是绘制调用中常量的数据。此顶点与顶点函数返回的顶点实例不完全对应。而是，如上所述，它是由光栅化器生成的插值版本。

片段函数负责返回要写入渲染缓冲区的颜色，因此`fragment_main`只提取（插入的）片段颜色并返回它。

#### Libraries(库)

通常，图形库需要单独编译每个着色器的源。然后必须将着色器链接在一起以形成程序。 Metal在着色器周围引入了更好的抽象：库。库只不过是用Metal着色语言编写的逻辑功能组。在本章的示例项目中，我们将顶点和片段函数集中到一个文件（Shaders.metal）中。此文件将与项目的其余部分一起编译，编译的函数包含在应用程序包中。 Metal允许我们在运行时通过名称轻松查找函数，如下一个清单所示。

```
id<MTLLibrary> library = [self.device newDefaultLibrary];

id<MTLFunction> vertexFunc = [library newFunctionWithName:@"vertex_main"]; 
id<MTLFunction> fragmentFunc = [library newFunctionWithName:@"fragment_main"];
```

我们也可以选择在运行时将应用程序的着色器动态编译为库，并从中请求函数对象。由于着色器代码在应用程序编译时是已知的，这是一个不必要的步骤，我们只使用默认库。

一旦引用了顶点和片段函数，就可以配置管道以使用它们，并且Metal会隐式执行必要的链接。这将在以下部分中说明。上面的代码包含在示例代码中的`-makePipeline`方法中。

### The Render Pipeline(渲染管道)

前几代图形库让我们将图形硬件视为状态机：你设置一些状态（比如从哪里拉数据，以及绘制调用是否应该写入深度缓冲区），并且该状态会影响绘制调用你问题之后。存在像OpenGL这样的图形库中的大多数API调用，以允许您设置状态的各个方面。对于第1章中讨论的“固定功能管道”尤其如此。

金属提供了一些不同的模型。不是调用作用于某个全局“上下文”对象的API，而是将大部分Metal的状态构建到预编译对象中，这些对象包含从一端获取顶点数据并在另一端生成光栅化图像的虚拟管道。

为什么这很重要？通过要求在预编译渲染状态中冻结昂贵的状态更改，Metal可以预先执行验证，否则必须在每次绘制调用时执行。像OpenGL这样的复杂状态机会花费大量的CPU时间，只需确保状态配置是自洽的。 Metal要求我们在管道创建过程中修复最昂贵的状态更改，而不是允许它们随时随意更改，从而避免了这种开销。让我们更深入地看一下。

#### Render Pipeline Descriptors(渲染管道描述符)

为了构建描述Metal应该如何渲染几何体的管道状态，我们需要创建一个描述符对象，它将一些东西连接起来：我们想要在绘制时使用的顶点和片段函数，以及我们的帧缓冲区的格式附件。
渲染管道描述符是MTLRenderPipelineDescriptor类型的对象，它包含管道的配置选项。以下是我们如何创建管道描述符对象：

```objc
MTLRenderPipelineDescriptor *pipelineDescriptor = [MTLRenderPipelineDescriptor new];
pipelineDescriptor.vertexFunction = vertexFunc; 
pipelineDescriptor.fragmentFunction = fragmentFunc; 
pipelineDescriptor.colorAttachments[0].pixelFormat = self.metalLayer.pixelFormat;
```

顶点和片段函数属性设置为我们先前从默认库请求的函数对象。

绘图时，我们可以定位许多不同类型的附件。附件描述了绘制结果的纹理。在这种情况下，我们只有一个附件：索引0处的颜色附件。这表示实际将在屏幕上显示的纹理。我们将附件的像素格式设置为`CAMetalLayer`的像素格式，后者支持我们的视图，以便Metal知道要绘制的纹理中像素的颜色深度和组件顺序。

既然我们有了一个管道描述符，我们需要让Metal来完成创建工作渲染管道状态。

#### Render Pipeline State(渲染管道状态)

`渲染管道状态`是符合协议`MTLRenderPipelineState`的对象。我们通过向设备提供新创建的管道描述符来创建管道状态，将结果存储在另一个名为pipeline的属性中。

```objc
 self.pipeline = [self.device newRenderPipelineStateWithDescriptor:pipelineDescriptor
error:NULL];
```

管道状态封装了从我们在描述符上设置的着色器派生的编译和链接着色器程序。因此，它是一个有点昂贵的对象。理想情况下，您只需为每对着色器函数创建一个渲染管道状态，但有时候附件的目标像素格式会有所不同，因此需要额外的渲染管道状态对象。关键是创建渲染管道状态对象是昂贵的，因此您应该避免在每帧的基础上创建它们，并尽可能地缓存它们。

上面的代码是示例代码中-buildPipeline方法的核心部分。由于渲染管道状态对象将在我们的应用程序的生命周期中存活，因此我们将其存储在属性中。我们还创建了一个命令队列并将其存储在一个属性中，如前一章所述。

### Encoding Render Commands(编码渲染命令)

在上一章中，我们没有与render命令编码器进行太多交互，因为我们没有执行任何绘图。这一次，我们需要将实际绘制调用编码到渲染命令缓冲区中。首先，让我们重新审视从图层和渲染过程描述符的配置中检索drawable。这是此示例项目的`-redraw`方法的开头：

```
 id<CAMetalDrawable> drawable = [self.metalLayer nextDrawable]; 
 id<MTLTexture> framebufferTexture = drawable.texture;
 ````
如果由于某种原因我们没有从drawable中获得可绘制或可渲染的纹理，则实际的绘制调用由`if(drawable)`条件保护。尝试使用纹理为nil的传递描述符创建命令缓冲区将导致异常。

renderPass对象的构造与之前完全相同，只是我们选择了浅灰色来清除屏幕，而不是我们上次使用的令人讨厌的红色。

```objc
MTLRenderPassDescriptor *passDescriptor = [MTLRenderPassDescriptor renderPassDescriptor];
passDescriptor.colorAttachments[0].texture = framebufferTexture; 
passDescriptor.colorAttachments[0].clearColor = MTLClearColorMake(0.85, 0.85, 0.85, 1);
passDescriptor.colorAttachments[0].storeAction = MTLStoreActionStore; 
passDescriptor.colorAttachments[0].loadAction = MTLLoadActionClear;
```

现在，我们通过使用渲染传递描述符创建渲染命令编码器并对我们的绘制调用进行编码来启动渲染过程：

```objc
id<MTLRenderCommandEncoder> commandEncoder = [commandBuffer renderCommandEncoderWithDescriptor:passDescriptor];
[commandEncoder setRenderPipelineState:self.pipeline]; 
[commandEncoder setVertexBuffer:self.vertexBuffer offset:0 atIndex:0]; 
[commandEncoder drawPrimitives:MTLPrimitiveTypeTriangle vertexStart:0 vertexCount:3]; 
[commandEncoder endEncoding];
```

-setVertexBuffer：偏移量：atIndex：方法用于从我们之前创建的MTLBuffer对象映射到着色器代码中的顶点函数的参数回想顶点参数是用[[缓冲液（0）]]属性归因的，注意我们现在在准备绘制调用时提供索引0。
我们调用`-drawPrimitives:vertexStart:vertexCount:instanceCount:`。方法来编写绘制三角形的请求我们将0传递给`vertexStart`参数，以便我们从缓冲区的最开始绘制我们为`vertexCount`传递3，因为三角形有三个点。
我们像以前一样通过触发绘制的显示并提交命令缓冲区来完成：

```objc
[commandBuffer presentDrawable:drawable]; 
[commandBuffer commit];
```

这些行完善了`-redraw`方法的新实现。

### Staying In-Sync With CADisplayLink(与CADisplayLink保持同步)

现在我们的-redraw实现已经完成，我们需要弄清楚如何重复调用它。我们可以使用常规的旧NSTimer，但Core Animation提供了更好的方法：CADisplayLink。这是一种特殊的定时器，与设备的显示环路同步，从而实现更一致的定时。每次显示链接
火，我们将调用我们的-redraw方法来更新屏幕。

我们重写-didMoveToSuperview来配置显示链接：
```
- (void)didMoveToSuperview
{
    [super didMoveToSuperview]; 
    if (self.superview)
    {
        self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(displayLinkDidFire:)];
        [self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
    } 
    else 
    {
        [self.displayLink invalidate];
        self.displayLink = nil; 
    }
}
```

这将创建一个显示链接，并使用主运行循环对其进行计划。如果我们从超级视图中删除，我们将使显示链接无效并将其取消。

每秒六十次，显示链接触发并调用其目标方法`-displayLinkDidFire:`，后者又调用`-redraw`:

```objc
- (void)displayLinkDidFire:(CADisplayLink *)displayLink 
{
    [self redraw]; 
}
```

### The Sample Project(示例项目)
本章的示例项目位于03-DrawingIn2D目录中。如果您构建并运行它，您应该会在屏幕上看到一个非常彩色的三角形。

![](1584062830.png)
<center>图3.2：由Metal渲染的多色三角形</center>

请注意，如果旋转显示屏，则三角形会扭曲。这是因为我们在剪辑空间中指定了三角形的坐标，无论纵横比如何，它始终沿着每个轴从-1到1运行。在下一章中，我们将讨论如何在渲染三维图形时补偿纵横比。