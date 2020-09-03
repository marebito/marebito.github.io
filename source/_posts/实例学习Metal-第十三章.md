---
title: 实例学习Metal-第十三章 介绍数据并行编程
date: 2020-03-29 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

本章介绍了数据并行编程的主题，在GPU上完成时也称为计算编程（与图形编程相反，这是我们通常使用GPU执行的操作）。我们将介绍设置计算管道和并行处理大型数据集上的内核函数的基础知识。本章将作为下一章的设置，演示如何使用计算内核在GPU上进行图像处理。

<!-- more -->

# Chapter 13(第十三章)

## Introduction to Data-Parallel Programming(介绍数据并行编程)

本章介绍了数据并行编程的主题，在GPU上完成时也称为计算编程（与图形编程相反，这是我们通常使用GPU执行的操作）。我们将介绍设置计算管道和并行处理大型数据集上的内核函数的基础知识。本章将作为下一章的设置，演示如何使用计算内核在GPU上进行图像处理。

### Kenel Functions(内核函数)

在本书中，我们广泛使用了顶点和片段函数。在本章中，我们介绍一种新的着色器函数：内核函数。内核函数允许我们构建大规模并行程序，这些程序可以同时处理多个数据。我们将交替使用术语“内核函数”，“计算内核”和“计算着色器”。

在着色器源中通过为内核限定符添加前缀来标识内核函数，就像我们使用`顶点`或`片段`为其他类型的函数添加前缀一样。这些类型的函数之间的一个区别是内核函数`必须`返回void。这是因为内核函数对缓冲区和纹理进行操作，而不是像顶点和片段函数那样将数据提供给其他管道阶段。

以下代码段是内核函数签名的示例。稍后将详细讨论此函数签名中引入的新属性。

```
kernel void kernel_function(
texture2d<float, access::read> inTexture [[texture(0)]], texture2d<float, access::write> outTexture [[texture(1)]], uint2 gid [[thread_position_in_grid]]);
```

### The Compute Pipeline(计算管道)

构建计算管道类似于为我们的3D渲染工作构建渲染管道。我们创建一个上下文对象来保存各种Metal对象，而不是渲染器类。

#### The Context Class(上下文类)

上下文包装了设备，库和命令队列，因为它们是长期存在的对象，将由我们创建的各种内核类引用。

上下文类有一个非常简单的接口。调用`+newContext`工厂方法将返回具有系统默认设备的上下文。

```
@interface MBEContext : NSObject
@property (strong) id<MTLDevice> device;
@property (strong) id<MTLLibrary> library; @property (strong) id<MTLCommandQueue> commandQueue;
+ (instancetype)newContext; @end
```

每个上下文有一个串行命令队列。这允许我们序列化工作项，例如可能在它们之间具有数据依赖性的图像过滤器。

#### Creating a Pipeline State(创建管道状态)

构建用于执行内核函数的管道状态比创建渲染管道要简单一些。没有与计算管道的MTLRenderPipelineDescriptor等效，因为计算管道的唯一可配置部分是其关联的内核函数。

与顶点和片段函数一样，内核函数通过库中的名称检索：

```
[library newFunctionWithName:@"kernel_function"];
```

然后通过从设备请求计算管道状态对象来创建计算管道。如果在为目标硬件编译内核函数时发生错误，则会将错误分配给error参数，并返回nil。

```
 id<MTLComputePipelineState> pipeline = [device newComputePipelineStateWithFunction:kernelFunction error:&error];
```

#### Creating a Command Buffer and Command Encoder(创建命令缓冲区和命令编码器)

正如我们使用渲染命令编码器将绘制调用编码到命令缓冲区中一样，我们使用新类型的命令编码器来执行内核函数：MTLComputeCommandEncoder。
```
id<MTLCommandBuffer> commandBuffer = [context.commandQueue commandBuffer]; id<MTLComputeCommandEncoder> commandEncoder =
[commandBuffer computeCommandEncoder];
```
但是，在将工作发送到GPU之前，我们需要了解如何配置计算命令编码器的参数表以及如何描述我们希望GPU执行的工作。

#### The Argument Table(参数表)

我们之前使用参数表来指定哪些缓冲区，纹理和采样器状态绑定到顶点和片段函数的参数。

例如，我们通过在渲染命令编码器上调用`-setFragmentTexture:atIndex:`方法来配置片段着色器的参数。回想一下，索引参数将参数表中的条目与片段函数的签名中具有相应属性（例如，`[[texture（0）]]`）的参数匹配。
设置计算编码器的参数表有类似的方法：`-setTexture:atIndex:`。在准备执行内核函数时，我们将使用此方法设置参数表。

#### Threadgroups(线程组)

线程组是Metal内核函数编程的核心概念。为了并行执行，必须将每个工作负载分解为称为线程组的块，这些块可以进一步划分并分配给GPU上的线程池。

为了有效运行，GPU不会调度单个线程。相反，它们被安排成套（有时称为“warp”或“wavefronts”，尽管Metal文档不使用这些术语）。线程执行宽度表示此执行单元的大小。这是实际计划在GPU上并发运行的线程数。您可以使用其`threadExecutionWidth`属性从命令编码器查询此值。它可能是2的小功率，例如32或64。

我们还可以通过查询`maxTotalThreadsPerThreadgroup`来确定线程组大小的上限。此数字始终是线程执行宽度的倍数。例如，在iPhone 6上它是512。

为了最有效地使用GPU，线程组中的项目总数应该是线程执行宽度的倍数，并且必须低于每个线程组的最大总线程数。这通知我们如何选择细分输入数据以实现最快和最方便的执行。

线程组不必是一维的。通常，线程组的尺寸与正在操作的数据的尺寸相匹配是很方便的。例如，当在2D图像上操作时，每个块通常将是源纹理的矩形区域。下图显示了如何选择将纹理细分为线程组。

![](1584068791.png)
<center>图13.1：要处理的图像被划分为多个线程组，此处用白色方块表示。</center>

为了告诉Metal每个线程组的维度以及在给定的计算调用中应该执行多少个线程组，我们创建了一对MTLSize结构：

```objc
MTLSize threadgroupCounts = MTLSizeMake(8, 8, 1);
MTLSize threadgroups = MTLSizeMake([texture width] / threadgroupCounts.width,
[texture height] / threadgroupCounts.height, 1);
```

在这里，我们有点随意选择8行8列的线程组大小，或者每个线程组总共64个项目。我们假设这个代码处理的纹理的维度是8的倍数，这通常是一个安全的选择。总线程组大小是目标硬件线程执行宽度的偶数倍，并且安全地低于最大总线程数。如果纹理尺寸不能被我们的线程组大小整除，我们需要在内核函数中采取措施，不要在纹理边界之外读取或写入。

既然在我们已经确定了线程组的大小以及我们需要执行多少线程组，现在我们已经准备好让GPU工作了。

#### Dispatching Threadgroups for Execution(调度线程组以执行)

编码命令来对一组数据执行内核函数称为`调度`。一旦我们引用了一个计算命令编码器，我们就可以调用它的`-dispatchThreadgroups:threadsPerThreadgroup:`方法对执行请求进行编码，传递我们之前计算过的MTLSize结构。

```
 [commandEncoder dispatchThreadgroups:threadgroups threadsPerThreadgroup:threadgroupCounts];
```
一旦完成调度，我们告诉命令编码器endEncoding，然后提交相应的命令缓冲区。然后我们可以在命令缓冲区上使用`-waitUntilCompleted`方法来阻止，直到着色器在GPU上运行完毕。每个数据项将执行一次内核函数（例如，源纹理中每个纹素一次）。

### Conclusion(结论)

在这个简短的章节中，我们为讨论Metal中的高性能图像滤波以及数据并行编程的其他应用奠定了基础。在下一章中，我们将在Metal中应用数据并行编程来解决图像处理中的一些有趣问题。