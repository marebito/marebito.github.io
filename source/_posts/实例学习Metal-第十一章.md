---
title: 实例学习Metal-第十一章 实例渲染
date: 2020-03-15 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

在本章中，我们将讨论一种通过单个绘制调用有效绘制许多对象的重要技术：实例渲染。此技术可帮助您充分利用GPU，同时将内存和CPU使用率降至最低。

本章的示例应用程序呈现了数十个动画奶牛在顶部移动随机生成的地形补丁。每头奶牛都有自己的位置，方向和运动方向，所有这些都在每一帧都更新。我们只用两次绘制调用来完成所有这些绘图。该应用程序仅消耗A8处理器上CPU的百分之几，但最大化GPU，每帧绘制超过240,000个三角形。即使有这么大的负载，这样的设备也能够以理想的每秒60帧的速度渲染。

<!-- more -->

# Chapter11(第11章)

## Instanced Rendering(实例渲染)

在本章中，我们将讨论一种通过单个绘制调用有效绘制许多对象的重要技术：实例渲染。此技术可帮助您充分利用GPU，同时将内存和CPU使用率降至最低。

本章的示例应用程序呈现了数十个动画奶牛在顶部移动随机生成的地形补丁。每头奶牛都有自己的位置，方向和运动方向，所有这些都在每一帧都更新。我们只用两次绘制调用来完成所有这些绘图。该应用程序仅消耗A8处理器上CPU的百分之几，但最大化GPU，每帧绘制超过240,000个三角形。即使有这么大的负载，这样的设备也能够以理想的每秒60帧的速度渲染。

### What is Instanced Rendering?(什么是实例渲染？)

虚拟世界经常在场景中拥有许多某些元素的副本：粒子，树叶，敌人等等。这些元素在内存中由单个几何体（网格）和一组特定于应用程序的属性表示。实例化渲染多次绘制相同的几何体，每个实例的属性用于控制它出现的位置和方式。

实例化渲染也称为“几何实例化”，“实例绘制”，或者有时只是“实例化”。

#### Setting the Scene(设置场景)

这一章的虚拟场景是一个田园山坡，有许多流浪的牛。每次应用程序运行时，地形都是唯一生成的，每头牛都有自己的随机运动路径。

![](1584067202.png)
<center>图11.1：使用实例化渲染可以有效地绘制数十个动画角色。 （草纹理由Simon Murray提供的goodtextures.com）</center>

#### Generating the Terrain(生成地形)

地形的特征由称为中点位移的算法创建，也称为“菱形方形”算法。这是一种递归细分地形边缘的技术，并随机向上或向下轻推它们以产生看起来自然的山丘。由于本章的重点是实例绘图而非地形生成，如果您对此技术感到好奇，请参阅示例源代码（请参阅MBETerrainMesh类）。可以在线找到该技术的交互式演示（Boxley 2011）。

#### Loading the Model(加载模型)

我们使用前面章节中的OBJ模型加载器来加载牛模型。一旦我们在内存中有一个OBJ模型，我们就会从OBJ文件中的相应组创建一个MBEOBJMesh实例。

#### Instanced Rendering(实例渲染)

通过发出绘制调用来执行实例渲染，该调用指定应该渲染几何的次数。为了使每个实例都有自己的属性，我们将包含每个实例数据的缓冲区设置为顶点着色器参数表中的缓冲区参数之一。我们还需要传入一个共享的统一缓冲区，它存储所有实例共享的制服。以下是配置用于渲染牛网格的完整参数表：

```objc
[commandEncoder setVertexBuffer:cowMesh.vertexBuffer offset:0 atIndex:0]; 
[commandEncoder setVertexBuffer:sharedUniformBuffer offset:0 atIndex:1]; 
[commandEncoder setVertexBuffer:cowUniformBuffer offset:0 atIndex:2];
```

现在让我们来看看我们如何在校园里布置制服。

#### Storing Per-Instance Uniforms(存储每个实例统一量)

对于每个实例，我们需要一个唯一的模型矩阵和一个相应的正常矩阵。回想一下，常规矩阵用于将网格的法线转换为世界空间。我们还希望将投影矩阵本身存储在共享的统一缓冲区中。我们将Uniforms结构拆分为两个结构：

```objc
typedef struct 
{
    matrix_float4x4 viewProjectionMatrix; 
} Uniforms;

typedef struct 
{
    matrix_float4x4 modelMatrix;
    matrix_float3x3 normalMatrix; 
} PerInstanceUniforms;
```

共享统一量存储在Metal缓冲区中，该缓冲区可容纳Uniforms类型的单个实例。每个实例的制服缓冲区可以为我们想要呈现的每头牛的一个PerInstanceUniforms实例提供空间：

```objc
cowUniformBuffer = [device newBufferWithLength:sizeof(PerInstanceUniforms) * MBECowCount
options:MTLResourceOptionCPUCacheModeDefault];
```

#### Updating Per-Instance Uniforms(更新每个实例统一量)

因为我们希望奶牛移动，所以我们在名为MBECow的数据模型中存储了一些简单的属性。每一帧，我们更新这些值以将奶牛移动到新位置并旋转它以使其与其行进方向对齐。

一旦奶牛对象是最新的，我们可以为每头奶牛生成适当的矩阵并将它们写入每个实例缓冲区，以便与下一个绘图调用一起使用：

```objc
PerInstanceUniforms uniforms;
uniforms.modelMatrix = matrix_multiply(translation, rotation); 
uniforms.normalMatrix = matrix_upper_left3x3(uniforms.modelMatrix);
NSInteger instanceUniformOffset = sizeof(PerInstanceUniforms) * instanceIndex; 
memcpy([self.cowUniformBuffer contents] + instanceUniformOffset, &uniforms, sizeof(PerInstanceUniforms));
```

#### Issuing the Draw Call(发给绘制调用)

要发给实例化绘图调用，我们在具有`instanceCount:`参数的渲染命令编码器上使用`-drawIndexedPrimitive:`方法。在这里，我们传递实例总数：

```objc
[commandEncoder drawIndexedPrimitives:MTLPrimitiveTypeTriangle                                    indexCount:indexCount
                            indexType:MTLIndexTypeUInt16
                          indexBuffer:cowMesh.indexBuffer
                    indexBufferOffset:0 
                        instanceCount:MBECowCount];
```

要执行此绘制调用，GPU将多次绘制网格，每次重复使用几何体。但是，我们需要一种在顶点着色器中获取适当矩阵集的方法，这样我们就可以将每头牛转换到它在世界中的位置。为此，我们来看看如何从顶点着色器中获取实例ID。

#### Accessing Per-Instance Data in Shaders(访问着色器中的每实例数据)

要索引每个实例的统一缓冲区，我们使用instance_id属性添加顶点着色器参数。这告诉Metal我们希望它向我们传递一个参数，该参数表示当前正在绘制的实例的索引。然后我们可以在正确的偏移处访问每个实例的uniforms数组并提取适当的矩阵：

```
vertex ProjectedVertex vertex_project(device InVertex *vertices [[buffer(0)]], constant Uniforms &uniforms [[buffer(1)]], constant PerInstanceUniforms *perInstanceUniforms [[buffer(2)]], ushort vid [[vertex_id]], ushort iid [[instance_id]]) 
{
    float4x4 instanceModelMatrix = perInstanceUniforms[iid].modelMatrix; 
    float3x3 instanceNormalMatrix = perInstanceUniforms[iid].normalMatrix;
```

顶点着色器的其余部分很简单。它投影顶点，变换法线，并穿过纹理坐标。

#### Going Further(走得更远)

您可以在每个实例的统一结构中存储所需的任何类型的数据。例如，您可以为每个实例传递颜色，并使用它来唯一地着色每个对象。您可以包含纹理索引，并将索引编入纹理数组，以便为​​某些实例提供完全不同的视觉外观。您还可以将缩放矩阵乘以模型转换，以便为每个实例提供不同的物理大小。基本上任何特征（网格拓扑本身除外）都可以改变，以便为每个实例创建一个独特的外观。

### The Sample App(示例应用)

本章的示例代码位于11-InstancedDrawing目录中。

您可以通过将手指放在屏幕上向前移动来移动示例应用程序。通过向左或向右平移来转动相机。