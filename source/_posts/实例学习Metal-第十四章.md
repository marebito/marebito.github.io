---
title: 实例学习Metal-第十三章 介绍数据并行编程
date: 2020-04-05 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

在本章中，我们将开始使用Metal着色语言探索图像处理世界。我们将创建一个能够表示图像滤镜链的框架，然后编写一对图像滤镜，以便我们调整图像的饱和度和模糊度。最终结果将是一个交互式应用程序，允许您实时控制图像过滤器参数。

图像处理是数据并行编程的主要应用之一。在许多情况下，图像滤波器仅需要查询源图像中的一个像素或小邻域像素以计算每个输出像素的值。在这种情况下，图像滤波器的工作可以以并行方式完成。这非常适合现代GPU架构，它使用许多小内核同时处理多个数据。

<!-- more -->

# Chapter 14(第十四章)

## Fundamentals of Image Processing(图像处理基础)

在本章中，我们将开始使用Metal着色语言探索图像处理世界。我们将创建一个能够表示图像滤镜链的框架，然后编写一对图像滤镜，以便我们调整图像的饱和度和模糊度。最终结果将是一个交互式应用程序，允许您实时控制图像过滤器参数。

图像处理是数据并行编程的主要应用之一。在许多情况下，图像滤波器仅需要查询源图像中的一个像素或小邻域像素以计算每个输出像素的值。在这种情况下，图像滤波器的工作可以以并行方式完成。这非常适合现代GPU架构，它使用许多小内核同时处理多个数据。

### A Look Ahead(展望未来)

为了进一步激励本章，这里是示例项目的一个片段。它说明了如何简洁地创建并将由可动态调整的用户界面控制的去饱和度和模糊过滤器链接在一起。

```
context = [MBEContext newContext]; 
imageProvider = [MBEMainBundleTextureProvider textureProviderWithImageNamed:@"mandrill" context:context];
desaturateFilter = [MBESaturationAdjustmentFilter filterWithSaturationFactor:0.75
context:context];
desaturateFilter.provider = self.imageProvider; 
blurFilter = [MBEGaussianBlur2DFilter filterWithRadius:0.0 context:context];
blurFilter.provider = desaturateFilter;
imageView.image = [UIImage imageWithMTLTexture:blurFilter.texture];
```

![](1584068924.png)
<center>图14.1：示例应用程序UI，允许实时过滤器调整。</center>

### A Framework for Image Processing(图像处理框架)

现在我们已经了解了我们希望图像处理框架的界面看起来像什么，我们可以讨论构建这样一个框架的实用性。该框架的大部分架构都受到（Jargstorff 2004）的启发。

#### Texture Providers and Consumers(纹理提供者和消费者)

每个过滤器都能够使用其输入和输出纹理配置其计算管道并执行其内核功能。

由于我们将以纹理的形式对图像进行操作，因此我们需要一种抽象的方式来引用生成和使用纹理的对象。例如，过滤器使用和生成纹理，我们还需要一个用于从中生成纹理的类应用程序包。

我们使用名为`MBETextureProvider`的协议来抽象纹理生成的概念：

```objc
@protocol MBETextureProvider <NSObject>
@property (nonatomic, readonly) id<MTLTexture> texture; 
@end
```

这个简单的界面为我们提供了一种(同步)从纹理提供者请求纹理的方法。它可能导致图像过滤器执行其过滤过程，或从磁盘加载图像。重要的是，我们知道我们可以从任何符合`MBETextureProvider`的对象中检索纹理。

另一方面，`MBETextureConsumer`协议允许我们告诉对象它应该从哪个纹理提供者使用纹理：

```objc
@protocol MBETextureConsumer <NSObject>
@property (nonatomic, strong) id<MBETextureProvider> provider; 
@end
```

这个简单的接口为我们提供了一种（同步）从纹理提供者请求纹理的方式。它可能导致图像过滤器执行其过滤过程，或从磁盘加载图像。重要的是，我们知道我们可以从任何符合`MBETextureProvider`的对象中检索纹理。

另一方面，`MBETextureConsumer`协议允许我们告诉对象它应该从哪个纹理提供者使用纹理：

```
@protocol MBETextureConsumer <NSObject>
@property (nonatomic, strong) id<MBETextureProvider> provider; 
@end
```

当纹理使用者想要对纹理进行操作时，它会从其提供者请求纹理并对其进行操作。

#### An Image Filter Base Class(图像过滤器基类)

抽象地，图像滤波器通过对其进行任意操作将一个纹理转换为另一个纹理。我们的MBEImageFilter类完成了调用计算着色器以从另一个生成一个纹理所需的大量工作。

图像过滤器基类符合刚才讨论的纹理提供者和纹理消费者协议。让过滤器表现为纹理提供者和纹理消费者都允许我们将过滤器链接在一起以按顺序执行多个操作。由于图像上下文管理的命令队列的串行特性，因此可以保证每个过滤器在允许其后继执行之前完成其工作。

这是MBEImageFilter类接口的相关部分：

```
@interface MBEImageFilter : NSObject <MBETextureProvider, MBETextureConsumer> @property (nonatomic, strong) MBEContext *context;
@property (nonatomic, strong) id<MTLComputePipelineState> pipeline; @property (nonatomic, strong) id<MTLTexture> internalTexture;
@property (nonatomic, assign, getter=isDirty) BOOL dirty;
- (instancetype)initWithFunctionName:(NSString *)functionName
context:(MBEContext *)context;
- (void)configureArgumentTableWithCommandEncoder: (id<MTLComputeCommandEncoder>)commandEncoder;
```

必须使用函数名称和上下文来实例化过滤器。这些用于创建计算管道状态，然后将其存储在`pipeline`属性中。

图像过滤器保持内部纹理，用作其内核函数的输出纹理。这样它可以存储其计算结果并将其提供给其他过滤器。它也可以绘制到屏幕或转换为图像。

图像过滤器可能具有任何数量的参数，这些参数会影响它们执行计算的方式。当其中一个更改时，内部纹理无效，并且必须重新执行内核函数。脏标志允许过滤器子类指示何时需要。仅当`dirty`标识设置为YES时才会执行过滤器，这在自定义属性设置器中完成。

图像过滤器基类包含`-applyFilter`方法，该方法在被要求提供其输出纹理时被调用，并且当前是脏的。此方法创建一个命令缓冲区和命令编码器，并调度其内核函数，如前一章所述。

既然我们已经拥有了必要的机器，让我们来谈谈将要使用的几个示例过滤器。

### Building a Saturation Adjustment Filter(构建饱和度调整过滤器)

我们将构建的第一个滤镜是饱和度调整滤镜。滤波器将具有可配置的饱和因子，用于确定滤波器应对输入图像进行去饱和的程度。此因子的范围为0到1.当饱和因子为0时，输出图像将是输入的灰度版本。对于介于0和1之间的值，滤镜将通过在灰度图像和输入图像之间进行插值来生成颜色或多或少静音的图像。

![](1584068954.png)
<center>图14.2：一系列图像显示了如何将滤波器的饱和因子从0增加到1会增加图像饱和度</center>

#### Calculating Brightness from RGB(从RGB计算亮度)

每个RGB颜色值都有一个相应的亮度值，我们将用符号Y'表示：
```
Y′ = 0.299R + 0.587G + 0.114B
```

请注意，公式中的因子总和为1.它们的值基于人类对不同原色光强度的感知：人眼对绿光最敏感，其次是红光，最后是蓝光。 （这些值作为ITU-R BT.601-7建议书（国际电信联盟2011）第2.5.1节）的一部分公布。

用相应亮度的灰色像素替换图像中的每种颜色会产生完全去饱和的图像，该图像与原始图像具有相同的感知亮度。这将是我们的去饱和核函数的任务，如下所示。

#### The Desaturation Kernel(去饱和核)

为了支持将饱和因子传递给我们的内核函数，我们创建了一个名为AdjustSaturationUniforms的单元结构：

```
struct AdjustSaturationUniforms 
{
    float saturationFactor; 
};
```

内核函数本身采用输入纹理，输出纹理，对统一结构的引用，以及具有我们在前一章中没有详细描述的属性的整数的2D向量：`thread_position_in_grid`。

回想一下前一章，我们发送了一组二维线程组，其大小是根据源纹理的维度计算的。 `thread_position_in_grid`属性告诉Metal生成一个坐标向量，告诉我们我们在2D网格中的位置，该网格跨越整个调度的工作项集，即源纹理中的当前坐标。

我们为每个纹理参数指定预期的访问模式：`access::read`用于输入纹理，`access::write`用于输出纹理。这限制了我们可以调用这些参数的函数集。

```
kernel void adjust_saturation(texture2d<float, access::read> inTexture [[texture(0)]],
texture2d<float, access::write> outTexture [[texture(1)]], constant AdjustSaturationUniforms &uniforms [[buffer(0)]], uint2 gid [[thread_position_in_grid]])
{
    float4 inColor = inTexture.read(gid);
    float value = dot(inColor.rgb, float3(0.299, 0.587, 0.114));
    float4 grayColor(value, value, value, 1.0);
    float4 outColor = mix(grayColor, inColor, uniforms.saturationFactor); outTexture.write(outColor, gid);
}
```

我们读取源纹素的颜色，并根据前面给出的公式计算其亮度值。点函数允许我们比分别进行三次乘法和两次加法更简洁地完成此操作。然后，我们通过将亮度复制到RGB组件中来生成新颜色，这会产生灰色阴影。

要计算部分去饱和的输出颜色，我们使用Metal标准库中的函数：mix，它采用两种颜色，系数介于0和1之间。如果系数为0，则返回第一种颜色，如果为1，则返回1 ，返回第二种颜色。在它们之间，它使用线性插值将它们混合在一起。

最后，我们将得到的去饱和颜色写入输出纹理。请注意，我们之前假设输入和输出纹理具有相同的大小，并且两者的维度都是我们的线程组大小的倍数。如果不是这种情况，我们需要防止在输出纹理边界之外的输入写入范围之外进行读取。

#### The Saturation Adjustment Class(饱和度调整类)

为了驱动饱和度调整内核，我们需要扩展图像过滤器基类。该子类名为`MBESaturationAdjustmentFilter`：

```objc
@interface MBESaturationAdjustmentFilter : MBEImageFilter 
@property (nonatomic, assign) float saturationFactor;
+ (instancetype)filterWithSaturationFactor:(float)saturation
context:(MBEContext *)context; 
@end
```

该子类使用去饱和内核函数的名称调用初始化程序，并通过对它应该操作的上下文的引用。

设置`saturationFactor`属性会导致过滤器设置其`dirty`属性，这会导致在其纹理属性被请求时延迟重新计算去饱和图像。

子类的`-configureArgumentTableWithCommandEncoder:`的实现：包含样板，用于将饱和因子复制到Metal缓冲区中。这里没有显示。

我们现在有一个完整的过滤器类和内核函数来执行图像去饱和。

#### 模糊

我们将看到的下一类图像滤镜是模糊滤镜。

模糊图像涉及将每个纹素的颜色与相邻文本的颜色混合。在数学上，模糊滤波器是纹素的邻域的加权平均值（这种操作称为`卷积`）。邻域的大小称为过滤器的半径。小半径将平均较少的纹理像素并产生较少模糊的图像。

#### 盒子模糊

最简单的模糊是盒子模糊。框模糊给予所有附近纹素的相同权重，计算它们的平均值。盒子模糊很容易计算，但会产生难看的伪影，因为它们会给噪音带来不适当的重量。

假设我们选择半径为1的盒子模糊。然后输出图像中的每个纹素将是输入纹理元素及其最近邻居的平均值。 9种颜色中的每一种都具有相同的1/9重量。

![](1584068998.png)
<center>图14.3：框模糊过滤器平均每个纹素周围的邻域。</center>

盒子模糊很容易，但它们不会产生非常令人满意的结果。相反，我们将使用更复杂的模糊滤镜，即高斯模糊。

#### Gussian Blur(高斯模糊)

与盒子模糊相反，高斯模糊给相邻纹素提供了不相等的权重，为更接近的纹素赋予了更多的权重，而对于那些更远的纹理则更轻。实际上，权重是根据关于当前纹素的2D正态分布计算的：

$$
G_\sigma(x, y) = {1 \over \sqrt {2\pi\sigma^2}}e^{-{x^2 + y^2\over 2 \sigma^2}}
$$

其中x和y分别是正在处理的纹理元素与其邻居之间沿x和y轴的距离。 σ是分布的标准偏差，默认等于半径的一半。

![](1584069023.png)
<center>图14.4：高斯模糊滤波器。增加滤镜半径可以创建更平滑的图像。此处显示的最大半径为7。</center>

#### The Blur Shaders(模糊着色器)

计算高斯滤波器的模糊权重在核函数中是很昂贵的，特别是对于具有大半径的滤波器。因此，我们将预先计算表格权重并将其作为纹理提供给高斯模糊核函数。此纹理的像素格式为`MTLPixelFormatR32Float`，它是单通道32位浮点格式。每个纹素都包含0到1之间的权重，所有权重总和为1。

在内核函数内部，我们迭代当前纹理元素的邻域，从查找表中读取每个纹素及其相应的权重。然后我们将加权颜色添加到累积值。一旦我们完成了所有加权颜色值的加总，我们就会将最终颜色写入输出纹理。

```
kernel void gaussian_blur_2d(texture2d<float, access::read> inTexture [[texture(0)]],
texture2d<float, access::write> outTexture [[texture(1)]], texture2d<float, access::read> weights [[texture(2)]], uint2 gid [[thread_position_in_grid]])
{
    int size = blurKernel.get_width(); int radius = size / 2;
    float4 accumColor(0, 0, 0, 0); for(intj=0;j<size;++j) 
    {
        for(inti=0;i<size;++i) 
        {
            uint2 kernelIndex(i, j);
            uint2 textureIndex(gid.x + (i - radius), gid.y + (j - radius)); float4 color = inTexture.read(textureIndex).rgba;
            float4 weight = weights.read(kernelIndex).rrrr;
            accumColor += weight * color;
        } 
    }
    outTexture.write(float4(accumColor.rgb, 1), gid); 
}
```

#### The Filter Class(过滤类)

模糊过滤器类`MBEGaussianBlur2DFilter`派生自图像过滤器基类。它的`-configureArgumentTableWithCommandEncoder:`的实现：懒惰地生成模糊权重并将命令编码器上的查找表纹理设置为第三个参数（参数表索引2）。

```objc
- (void)configureArgumentTableWithCommandEncoder: (id<MTLComputeCommandEncoder>)commandEncoder
{
    if (!self.blurWeightTexture) 
    {
        [self generateBlurWeightTexture]; 
    }
    [commandEncoder setTexture:self.blurWeightTexture atIndex:2]; 
}
```

`-generateBlurWeightTexture`方法使用上面的2D标准分布公式计算适当大小的权重矩阵，并将值复制到Metal纹理中。

这完成了我们对高斯模糊滤镜类和着色器的实现。现在我们需要讨论如何将滤镜链接在一起并将最终图像输出到屏幕上。

### Chaining Image Filters(链接图像过滤器)

再次考虑本章开头的代码：

```objc
context = [MBEContext newContext]; imageProvider = [MBEMainBundleTextureProvider
textureProviderWithImageNamed:@"mandrill" context:context];
desaturateFilter = [MBESaturationAdjustmentFilter filterWithSaturationFactor:0.75
context:context];
desaturateFilter.provider = self.imageProvider; 
blurFilter = [MBEGaussianBlur2DFilter filterWithRadius:0.0 context:context];
blurFilter.provider = desaturateFilter;
imageView.image = [UIImage imageWithMTLTexture:blurFilter.texture];
```

主捆绑图像提供程序是一个实用程序，用于将图像加载到Metal纹理中，并充当链的开头。它被设置为去饱和度过滤器的纹理提供者，后者又被设置为模糊过滤器的纹理提供者。

请求模糊滤镜的纹理实际上是将图像滤镜处理设置为运动的原因。这会导致模糊滤镜请求去饱和过滤器的纹理，这反过来会导致去饱和内核同步调度。一旦完成，模糊滤镜将去饱和纹理作为输入并调度其自己的内核函数。

现在我们已经过滤了图像，我们可以使用它来渲染带有Metal的纹理四边形（或其他表面）。假设我们想用UIKit显示它？我们如何从UIImage创建Metal纹理，而不是从Metal纹理创建UIImage？

### Creating a UIImage from a Texture(从纹理创建UIImage)

最有效的方法是使用Core Graphics中的图像实用程序。首先，创建一个临时缓冲区，其中读取Metal纹理的像素数据。可以使用CGDataProviderRef包装此缓冲区，然后使用CGDataProviderRef创建CGImageRef。然后我们可以创建一个包装此CGImageRef的UIImage实例。

示例代码在UIImage上实现了一个类别，该类别在名为`+imageWithMTLTexture:`的方法中执行所有这些操作。为简洁起见，此处不包含此内容，但它有助于阅读。

值得一提的是，这不是在屏幕上获取过滤图像的最有效方法。从纹理创建图像使用额外的内存，占用CPU时间，并要求Core Animation合成器将图像数据复制回纹理以供显示。所有这些都是昂贵的，并且在许多情况下可以避免。幸运的是，在本书的整个过程中，我们已经看到很多方法可以使用不涉及UIKit的纹理。

#### Driving Image Processing Asynchronously(异步驱动图像处理)

上面，我们提到过滤器同步调度它们的内核函数。由于图像处理是计算密集型的，我们需要一种在背景线程上进行工作的方法，以保持用户界面的响应。

将命令缓冲区提交到命令队列本质上是线程安全的，但是控制并发的其他方面是程序员的责任。

幸运的是，Grand Central Dispatch使我们的工作变得轻松。由于我们只会在主线程上响应用户操作而使用图像过滤器，因此我们可以使用一对`dispatch_async`调用将我们的图像处理工作重定位到后台线程上，异步更新主线程上的图像视图过滤器的处理完成。

我们将在视图控制器上以原子64位整数属性的形式使用粗互斥，每次请求更新时都会递增。该计数器的值由入队的块捕获。如果块在后台队列上执行时尚未发生另一个用户事件，则允许执行图像过滤器，并刷新UI。

```objc
- (void)updateImage {
    ++self.jobIndex;
    uint64_t currentJobIndex = self.jobIndex;
    float blurRadius = self.blurRadiusSlider.value; 
    float saturation = self.saturationSlider.value;
    dispatch_async(self.renderingQueue, ^{ 
        if (currentJobIndex != self.jobIndex)
            return;
        self.blurFilter.radius = blurRadius; self.desaturateFilter.saturationFactor = saturation;
        id<MTLTexture> texture = self.blurFilter.texture; UIImage *image = [UIImage imageWithMTLTexture:texture];
        dispatch_async(dispatch_get_main_queue(), ^{ self.imageView.image = image;
        }); 
    });
}
```

### The Sample Project(示例项目)

本章的示例代码位于14-ImageProcessing目录中。

在本章中，我们采用了与Metal并行计算的后续步骤，并看到了几个可以在GPU上高效运行的图像过滤器示例。您现在可以使用此处提供的框架来创建自己的效果，并使用Metal的强大功能在GPU上高效运行它们。