---
title: 实例学习Metal-第四章 3D绘制
date: 2020-01-26 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

基于我们在前一章中学习的渲染管道，我们现在将开始三维渲染。

<!-- more -->

# Chapter 4(第四章)

## Drawing in 3D(3D绘图)

基于我们在前一章中学习的渲染管道，我们现在将开始三维渲染。

### Specifying Geometry in 3D(在3D中指定几何)

#### Cube Geometry(立方体几何)

我们将在本章中呈现的对象是一个简单的立方体。在代码中编写立方体的顶点很容易，从而避免了现在加载3D模型的复杂性。以下是立方体网格的顶点：

```objc
const MBEVertex vertices[] = {
    {.position={-1, 1, 1,1},.color={0,1,1,1}}, 
    {.position={-1,-1, 1,1},.color={0,0,1,1}}, 
    {.position={ 1,-1, 1,1},.color={1,0,1,1}}, 
    {.position={ 1, 1, 1,1},.color={1,1,1,1}}, 
    {.position={-1, 1,-1,1},.color={0,1,0,1}}, 
    {.position={-1,-1,-1,1},.color={0,0,0,1}}, 
    {.position={ 1,-1,-1,1},.color={1,0,0,1}}, 
    {.position={ 1, 1,-1,1},.color={1,1,0,1}}
};
```

![](1584062898.png)
<center>图4.1：示例应用程序呈现的多维数据集</center>

我们重用前一章中的相同MBEVertex结构，它具有每个顶点的位置和颜色。由于我们尚未引入照明，因此为每个顶点赋予不同的颜色可提供重要的深度提示。和以前一样，我们创建一个缓冲区来保存顶点：
```
vertexBuffer = [device newBufferWithBytes:vertices length:sizeof(vertices) options:MTLResourceOptionCPUCacheModeDefault];
```

### Index Buffers(索引缓冲区)

在上一章中，我们按照绘制它们的顺序存储三角形的顶点，每个顶点只使用一次。在立方体的情况下，每个顶点属于几个三角形。理想情况下，我们将重用这些顶点，而不是在内存中存储每个顶点的额外副本。随着模型的大小增加，顶点重用变得更加重要。

幸运的是，与大多数图形库一样，Metal使我们能够提供索引缓冲区以及顶点缓冲区。索引缓冲区包含一个指向ver-tex缓冲区的索引列表，用于指定每个三角形组成的顶点。

首先，我们定义了几个typedef，它们将简化索引的处理：

```
typedef uint16_t MBEIndex;
const MTLIndexType MBEIndexType = MTLIndexTypeUInt16;
```

首先，我们将使用16位无符号索引。这允许每个网格包含多达65536个不同的顶点，这将在很长一段时间内满足我们的目的。如果我们将来需要容纳更多顶点，我们可以更改这些定义，并且我们的代码将适应更大的索引大小。 Metal允许16位和32位索引。

立方体的每个正方形面分为两个三角形，包括六个指数。我们在数组中指定它们，然后将它们复制到缓冲区中：

```objc
const MBEIndex indices[] = 
{
    3,2,6,6,7,3, 
    4,5,1,1,0,4, 
    4,0,3,3,7,4, 
    1,5,6,6,2,1, 
    0,1,2,2,3,0, 
    7,6,5,5,4,7
};
indexBuffer = [device newBufferWithBytes:indices length:sizeof(indices) options:MTLResourceOptionCPUCacheModeDefault];
```
既然我们已经定义了一些要使用的几何体，让我们来谈谈如何渲染3D场景。

### Dividing Work between the View and the Renderer(在视图和渲染器之间划分工作)

在上一章中，我们给`MBEMetalView`类负责渲染三角形。现在，我们希望通过修复视图的功能，将资源管理和渲染的工作卸载到一个单独的类：渲染器，转向更可持续的模型。

#### Responsibilities of the View Class(在视图和渲染器之间划分工作)

在上一章中，我们给`MBEMetalView`类负责渲染三角形。现在，我们希望通过修复视图的功能，将资源管理和渲染的工作卸载到一个单独的类：渲染器，转向更可持续的模型。

视图类应该只关注将像素放到屏幕上，因此我们从中删除命令队列，渲染管道和缓冲区属性。它保留了监听显示链接和管理纹理的责任渲染过程的附件。

新的`MBEMetalView`提供了名为`currentDrawable`的属性，它为当前帧的`CAMetalDrawable`对象和`currentRenderPassDescriptor`提供了一个属性，它将使用绘制的纹理配置的渲染过程描述符作为其主要颜色附件。

#### The Draw Protocol(绘制协议)

为了绘图，我们需要一种方法让视图与我们沟通，是时候进行绘制调用了。我们通过名为`MBEMetalViewDelegate`的协议将视图的概念与渲染器的概念分离，该协议具有单个必需的方法：`-drawInView：`。

每个显示周期将调用此绘制方法一次，以允许我们刷新视图的内容。在委托的方法实现中，`currentDrawable`和`currentRenderPassDescriptor`属性可用于创建渲染命令编码器（我们将经常调用`渲染过程`）并对其发出绘制调用。

#### Responsibilities of the Renderer Class(渲染者类的责任)

我们的渲染器将保存我们用于使用Metal渲染的长寿命对象，包括管道状态和缓冲区等。它符合`MBEMetalViewDelegate`协议，因此通过创建命令缓冲区和命令编码器来发出绘制调用来响应`-drawInView：`消息。然而，在我们开始讨论之前，我们需要讨论绘制调用将要执行的工作。

### Transforming from 3D to 2D(从3D转换为2D)

为了将3D几何体绘制到2D屏幕，这些点必须经历一系列变换：从对象空间到世界空间，到眼睛空间，到剪辑空间，到规范化设备坐标，最后到屏幕空间。

#### From Object Space to World Space(从物空间到世界空间)

构成3D模型的顶点用局部坐标空间（称为`对象空间`）表示。我们的立方体的顶点是关于原点的，它位于立方体的中心。为了在更大的场景中定位和定位对象，我们需要
指定一个可以缩放，平移和旋转它们到`世界空间`的变换。

您可以从线性代数中回忆一下，矩阵可以相乘（连接）以构建表示线性变换序列的单个矩阵。我们将调用矩阵，该矩阵将将对象移动到`模型矩阵`的世界空间中的变换序列聚集在一起。我们立方体的模型矩阵包括一个尺度变换，然后是两个旋转。这些单独变换中的每一个都随时间变化，以实现脉冲旋转立方体的效果。

以下是创建转换序列并将它们相乘以创建世界转换的代码：
```objc
float scaleFactor = sinf(5 * self.time) * 0.25 + 1;
vector_float3 xAxis = { 1, 0, 0 };
vector_float3 yAxis = { 0, 1, 0 };
matrix_float4x4 xRot = matrix_float4x4_rotation(xAxis, self.rotationX); matrix_float4x4 yRot = matrix_float4x4_rotation(yAxis, self.rotationY); matrix_float4x4 scale = matrix_float4x4_uniform_scale(scaleFactor); matrix_float4x4 modelMatrix = matrix_multiply(matrix_multiply(xRot, yRot),
scale);
```

我们使用矩阵从右到左应用于列向量的约定，因此modelMatrix缩放顶点，然后围绕Y轴旋转它，然后围绕X轴旋转它。每帧都会更新rotationX，rotationY和time属性，以便对此转换进行动画处理。

#### From World Space to View Space(从世界空间到观看空间)

现在我们在世界空间中拥有了场景（缩放，旋转的立方体），我们需要相对于虚拟摄像机的视点定位整个场景。这种变换称为`视图空间`（或等效地，`眼睛空间`或`相机空间`）变换。最终在屏幕上可见的所有内容都包含在称为视锥，如下图所示。虚拟摄像机眼睛的位置是该观察体积的顶点，即观察平截头体近平面中间的点。

![](1584062966.png)
<center>图4.2：视锥体。平截头体的顶点与虚拟相机的眼睛重合。</center>

构建相机的变换需要我们向后思考：将相机向后移动相当于将场景更深地移动到屏幕中，并且绕Y轴逆时针旋转场景相当于顺时针绕轨道旋转场景的相机。

在我们的示例场景中，我们希望将相机从多维数据集放回几个单位。我们使用世界空间“惯用右手”的惯例，Y轴朝上，这意味着Z轴指向屏幕外。因此，正确的转变
是一个平移，沿Z轴移动每个顶点一个`负`距离。等效地，该变换使摄像机沿Z轴移动正距离。这都是相关的。

以下是构建视图矩阵的代码：

```objc
vector_float3 cameraTranslation = { 0, 0, -5 };
matrix_float4x4 viewMatrix = matrix_float4x4_translation(cameraTranslation);
```

#### From View Space to Clip Space(从视图空间到剪辑空间)

投影矩阵将视图空间坐标转换为`剪辑空间`坐标。

剪辑空间是GPU用于确定观看体积内三角形的可见性的3D空间。如果三角形的所有三个顶点都在剪辑体积之外，根本不渲染三角形（它被剔除）。另一方面，如果一个或多个顶点在体积内，则将其剪切到边界，并且使用一个或多个修改的三角形作为顶点着色器的输入。

![](1584062996.png)
<center>图4.3：被剪裁的三角形的图示。它的两个顶点超出了剪裁边界，因此面被剪裁和重新三角化，创建了三个新的三角形。</center>

透视投影矩阵通过一系列缩放操作将视点空间中的点带入剪辑空间。在先前的变换中，`w`分量保持不变并且等于1，但是投影矩阵以这样的方式影响`w`分量：如果`x`，`y`或`z`分量的绝对值大于`w`分量的绝对值，顶点位于查看体积之外并被剪裁。

透视投影变换封装在`matrix_float4x4_perspective`工具函数中。在示例代码中，修复约70度的垂直视野，选择远近平面值，并选择等于我们的Metal视图的当前可绘制宽度和高度之间比例的纵横比。

```objc
float aspect = drawableSize.width / drawableSize.height; floatfov=(2*M_PI)/5;
float near = 1;
float far = 100;
matrix_float4x4 projectionMatrix = matrix_float4x4_perspective(aspect, fov, near, far);
```

我们现在有一系列矩阵将把我们从对象空间一直移动到剪辑空间，这是Metal期望我们的顶点着色器函数返回的顶点的空间。将所有这些矩阵相乘产生一个`模型-视图-投影`（MVP）矩阵，这是我们实际传递给顶点着色器的矩阵，以便于每个顶点都可以在GPU上与它相乘。

#### The Perspective Divide: From Clip Space to NDC(透视划分：从剪辑空间到NDC)

在透视投影的情况下，计算`w`分量以使透视分割产生透视缩小，更远的物体缩小的现象。

在我们从顶点函数将投影顶点移交给Metal之后，它将每个组件除以w组件，从剪辑空间坐标移动到`标准化设备坐标`（NDC），在对查看体积边界进行相关剪切之后。 Metal的NDC空间是一个长方体[-1,1]×[-1,1]×[0,1]，这意味着x和y坐标的范围是-1到1，当我们移动时z坐标的范围是0到1`远离相机`。

#### The Viewport Transform: From NDC to Window Coordinates(视口变换：从NDC到窗口坐标)

为了从NDC的半立方体映射到视图的像素坐标，Metal通过缩放和偏置标准化设备坐标来进行最终的内部转换，使得它们覆盖`视口`的大小。在我们的所有示例代码中，视口都是一个覆盖整个视图的矩形，但是可以调整视口大小，使其仅覆盖视图的一部分。

### 3D Rendering in Metal(Metal中的3D渲染)

#### Uniforms(统一量)

uniform是一个值，它作为参数传递给着色器，该着色器在绘制调用过程中不会发生变化。从着色器的角度来看，它是一个常数。
在接下来的章节中，我们将我们的uniform捆绑在一个自定义结构中。即使我们现在只有一个这样的值（MVP矩阵），我们现在也会养成这样的习惯：
```
typedef struct {
    matrix_float4x4 modelViewProjectionMatrix; 
} MBEUniforms;
```

由于我们正在为我们的立方体制作动画，我们需要每帧重新生成制服，因此我们将用于生成变换的代码放入缓冲区中，并将其写入名为`-updateUniforms`的渲染器类的方法中。

#### The Vertex Shader(顶点着色器)

现在我们在工具箱中有了一些3D绘图的基础数学，让我们讨论如何让Metal实际完成变换顶点的工作。

我们的顶点着色器采用指向Vertex类型顶点数组的指针，这是在着色器源中声明的结构。它还需要一个指向Uniforms类型的统一结构的指针。这些类型的定义是：
```
struct Vertex 
{
    float4 position [[position]];
    float4 color; 
};
struct Uniforms 
{
    float4x4 modelViewProjectionMatrix; 
};
```

请注意，我们在函数的参数列表中使用了两个不同的地址空间限定符：`device`和`constant`。通常，在使用每顶点或每个片段偏移（例如使用`vertex_id`归因的参数）索引到缓冲区时，应使用设备地址空间。当函数的许多调用将访问缓冲区的相同部分时，使用常量地址空间，就像访问每个顶点的统一结构时的情况一样。

#### The Fragment Shader(片段着色器)

片段着色器与前一章中使用的着色器相同：

```
fragment half4 fragment_flatcolor(Vertex vertexIn [[stage_in]]) 
{
    return half4(vertexIn.color);
}
```

#### Preparing the Render Pass and Command Encoder(准备渲染通道和命令编码器)

每个帧，我们需要在发出绘制调用之前配置渲染通道和命令编码器：

```objc
    [commandEncoder setDepthStencilState:self.depthStencilState]; 
    [commandEncoder setFrontFacingWinding:MTLWindingCounterClockwise]; 
    [commandEncoder setCullMode:MTLCullModeBack];
```

`depthStencilState`属性设置为先前配置的模板深度状态对象。

正面`缠绕`顺序确定金属是否将面向顶点顺时针或逆时针考虑为正面。默认情况下，Metal将顺时针面视为正面。样品数据和样品代码更喜欢逆时针方向，因为这在右手坐标系中更有意义，因此我们通过在此设置缠绕顺序来覆盖默认值。

`剔除模式`确定是应该丢弃（剔除）正面或背面三角形（或两者都不）。这是一种优化，可以防止绘制不可见的三角形。

#### Issuing the Draw Call(发出绘制调用)

渲染器对象在渲染过程'参数表上设置必要的缓冲区属性，然后调用适当的方法来渲染顶点。由于我们不需要发出额外的绘制调用，因此我们可以在此渲染过程中结束编码。

```objc
[renderPass setVertexBuffer:self.vertexBuffer offset:0 atIndex:0];

NSUInteger uniformBufferOffset = sizeof(MBEUniforms) * self.bufferIndex; [renderPass setVertexBuffer:self.uniformBuffer offset:uniformBufferOffset
atIndex:1];

[renderPass drawIndexedPrimitives:MTLPrimitiveTypeTriangle indexCount:[self.indexBuffer length] / sizeof(MBEIndex) indexType:MBEIndexType
indexBuffer:self.indexBuffer
indexBufferOffset:0]; 

[renderPass endEncoding];
```

正如我们在前一章中看到的，第一个参数告诉Metal我们想要绘制什么类型的原始图形，无论是点，线还是三角形。其余参数告诉Metal索引缓冲区的计数，大小，地址和偏移量，用于索引到先前设置的顶点缓冲区。

### The Sample Project(示例项目)

本章的示例代码位于04-DrawingIn3D目录中。将上面学到的所有内容整合在一起，就可以呈现出光彩夺目的旋转立方体。