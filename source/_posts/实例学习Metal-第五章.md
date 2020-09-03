---
title: 实例学习Metal-第五章 光照
date: 2020-02-02 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

在本章中, 我们将开始增加虚拟场景中的现实主义, 方法是从文件中加载3D 模型, 并用方向灯照亮它们。

直到现在, 我们一直进行硬编码我们的几何数据直接进入程序。这对于微小的几何形状来说是很好的, 但是它很快就变得不可持续了。通过将模型数据以标准格式存储在文件系统中, 我们可以开始使用更大的数据集。

<!-- more -->

## Lighting(光照)

在本章中, 我们将开始增加虚拟场景中的现实主义, 方法是从文件中加载3D 模型, 并用方向灯照亮它们。

直到现在, 我们一直进行硬编码我们的几何数据直接进入程序。这对于微小的几何形状来说是很好的, 但是它很快就变得不可持续了。通过将模型数据以标准格式存储在文件系统中, 我们可以开始使用更大的数据集。

### Loading OBJ Models(加载OBJ模型)

存储基本3D 模型数据最常用的格式之一是 OBJ 格式 (Wavefront技术 1991)。OBJ文件以可读的格式存储顶点位置、法线、纹理坐标和面孔的列表。示例代码包括一个未成熟的obj分析器, 它可以用合适的格式将OBJ模型加载到内存中以使用Metal进行渲染。我们不会在这里详细描述加载器。 

我们在本章中使用的模型是一个茶壶。茶壶形状是计算机图形学早期最知名的可识别图形之一。它最初是以1975年犹他大学的Martin Newell的Melitta茶壶为蓝本的。由于其有趣的拓扑和不对称性, 它经常在图形教程中露面。

![](1584063082.png)
<center>图 5.1：计算机渲染的Utah茶壶模型</center>

您应该随时重新编译示例项目, 并用您自己的OBJ文件替换包含的模型。标准化为适合原点周围的1×1×1立方体的模型将是最容易使用的模型。


#### The Structure of OBJ Models(OBJ模型的结构)

OBJ格式的模型在逻辑上被分成`groups`，这些组基本上是多边形的命名列表。加载模型就像在应用程序包中查找OBJ文件，初始化MBEOBJModel对象以及按索引查询组一样简单。

```
NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"teapot" withExtension:@"obj"];
MBEOBJModel *teapot = [[MBEOBJModel alloc] initWithContentsOfURL:modelURL]; MBEOBJGroup *group = [teapot groupAtIndex:1];
```

索引1对应于文件中的第一个组;不属于任何组的多边形将添加到索引0的隐式、未命名组中。
MBEOBJGroup 是一个具有两个指针成员的结构: `vertices(顶点)`和`indices(索引)`。这些顶点和指数列表对我们自己没有任何好处;我们需要把它们放到缓冲里区。由于顶点和索引的列表经常一起使用, 所以我们需要一个对象来保存这对缓冲区。这就是我们将要说的`mesh(网格)`。

#### The Mesh Class(网格类)

网格类的接口非常简单，只包含两个缓冲区属性。

```
@interface MBEMesh : NSObject
@property (readonly) id<MTLBuffer> vertexBuffer; @property (readonly) id<MTLBuffer> indexBuffer; @end
```

我们将网格类子类化以生成特定于应用程序的网格类型。现在，我们需要一个表示OBJ组的网格，因此我们使用以下初始化程序创建`MBEOBJMesh`类：

```
 - (instancetype)initWithGroup:(MBEOBJGroup *)group device:(id<MTLDevice>)device;
 ```

此方法采用OBJ组和Metal设备，并生成一个缓冲区对，其中包含组顶点和索引的存储空间。我们可以通过将其顶点缓冲区绑定到参数表并编码索引绘制调用来渲染这样的网格，就像我们在上一章中所做的那样：

```
[renderPass setVertexBuffer:mesh.vertexBuffer offset:0 atIndex:0]; [renderPass drawIndexedPrimitives:MTLPrimitiveTypeTriangle
indexCount:[mesh.indexBuffer length] / sizeof(MBEIndex) indexType:MBEIndexType indexBuffer:self.mesh.indexBuffer indexBufferOffset:0];
```

#### Normals(法线)

我们的光照计算将取决于对象的`法线向量`。法线向量是垂直于表面的`单位向量`（长度为1的向量）。如果您想象一个平面仅在一个点（我们称之为切平面）接触凸面，则法向量是垂直于平面的向量。它代表了
表面的“向外”方向。

我们的OBJ模型加载器完成了为模型中的每个定点生成法线向量所需的工作。我们还在顶点结构中添加了一个成员来持有法线，因此它在着色器源中的声明现在看起来像这样：

```C++
struct Vertex 
{
    float4 position;
    float4 normal; 
};
```

### Lighting(光照)

为了开始构建逼真的场景，我们需要一种方法来模拟光线与曲面的相互作用。幸运的是，许多计算机图形本身都在寻找巧妙的方法来近似光的行为。我们将使用由三个项组成的近似值来模拟场景中出现的不同种类的光：环境光，漫反射光和镜面光。每个像素的颜色将是这三个术语的总和。以下是表示最高级别关系的等式：

I<sub>total</sub> = I<sub>ambient</sub> + I<sub>diffuse</sub> + I<sub>specular</sub>

在上面，`I`代表intensity(强度)，特别是外向辐射强度。我们将使用`L`来表示光源的属性，使用`M`来表示材质的属性。这些值都没有精确的物理基础。我们第一次尝试近似照明涉及很多捷径。因此，我们为这些数量选择的值将基于美学而非物理精确度。

我们将要处理的光照类型是`directional(方向性的)`;他们在太空中没有明确的位置。相反，我们想象它们足够远，只能通过它们发光的方向来表征。太阳是这种定向光的一个例子。

##### Ambient Light(环境光)

环境光是与特定光源无关的非定向种类的光。相反，它是用于对间接光（从场景中的其他表面反弹的光）进行建模的近似值，其不被其他术语捕获。环境照明可防止未直接照明的几何体部分完全变黑。环境光的贡献通常是微妙的。

环境光计算为光的环境强度和光的强度的乘积
表面的环境响应：

I<sub>ambient</sub> = L<sub>ambientMambient</sub>

通常情况下，人们将环境贡献建模为场景的属性而不是光。这表明环境光并非真正来自一个来源;相反，它是从任何光源出现在场景周围反射的光的总和。为了方便和公式的一致性，我们选择使环境光成为光本身的属性。

![](1584063173.png)
<center>图 5.2：环境光期通常会做出很小的贡献</center>

##### Diffuse Light(漫反射光)

漫反射光遵循朗伯的余弦定律，该定律指出反射光的强度与入射光的方向与表面法线之间的角度的余弦成正比。法线是垂直于曲面的矢量。正面照射的光比以浅入射角到达的光反射的程度更大。

![](1584063201.png)
<center>图5.3：在漫射照明中，反射的光量与法线和入射角之间的角度成正比</center>

漫反射通过以下简化方程建模：

I<sub>diffuse</sub> = cosθ·M<sub>diffuse</sub> · L<sub>diffuse</sub>

如果我们假设法向量和入射光方向向量是单位向量，
我们可以使用熟悉的点积来计算这个术语：

I<sub>diffuse</sub> = N·L·M<sub>diffuse</sub>·L<sub>diffuse</sub>

![](1584063234.png)
<center>图5.4：漫反射术语表示表面在所有方向上均匀反射光的趋势</center>

##### Specular Light(镜面光)

镜面光项用于模拟“有光泽”的表面。它描述了材料在特定方向上反射光而不是在各个方向上散射光的趋势。闪亮的材料创造出镜面高光，这是一个强大的视觉线索，用于说明粗糙或有光泽的表面。 “光泽”由称为镜面反射功率的参数量化。例如，5的镜面反射功率对应于相当无光泽的表面，而镜面反射功率50对应于稍微有光泽的表面。

有几种流行的计算镜面术语的方法，但我们将在这里使用
Blinn-Phong近似（Blinn 1977）。 Blinn-Phong镜面术语使用一个称为`中间向量`的向量，该向量指向光源方向和观察表面的方向之间的中间位置：

H = (1/2)(D+V)

一旦我们掌握了中间矢量，我们计算表面法线和中间矢量之间的点积，并将该量增加到材料的镜面反射率。

I<sub>specular</sub> =(N·H)<sup>specularPower</sup>L<sub>specular</sub>M<sub>specular</sub>

这个取幂是控制所产生的镜面反射高度的“紧密”的原因;
镜面反射功率越高，高光越清晰。

![](1584063306.png)
<center>图5.5：镜面术语表示某些表面在特定方向上反射光的趋势</center>

结果

现在我们已经计算了光照方程的所有三个项，我们将每个像素的结果加在一起以找到它的最终颜色。在接下来的几节中，我们将讨论如何使用Metal着色语言实际实现此效果。

![](1584063329.png)
<center>图5.6：添加环境，漫反射和镜面反射条件的贡献可以生成最终图像</center>

### Lighting in Metal(Metal中的光照)

代表灯光和材料

我们将使用两个结构来包装与灯光和材质相关的属性。灯具有三种颜色属性，一个用于我们详细讨论的每个灯光项。

```C++
struct Light 
{
    float3 direction; 
    float3 ambientColor; 
    float3 diffuseColor; 
    float3 specularColor;
};
```

材料还具有三种颜色属性;这些模型模拟了材料对入射光的响应。从表面反射的光的颜色取决于表面的颜色和入射光的颜色。正如我们所见，光的颜色和表面的颜色相乘以确定表面的颜色，然后将这个颜色乘以一些强度（在环境的情况下为1，在这种情况下为*N*·*L*）在镜面反射的情况下，弥散和N·H提升到一些幂。我们将镜面反射力模型化为浮点数。

```C++
struct Material 
{
    float3 ambientColor; 
    float3 diffuseColor; 
    float3 specularColor; 
    float specularPower;
};
```

#### Uniforms(统一量)

正如我们在前一章中所做的那样，我们需要将一些统一值传递给着色器来转换顶点。然而，这一次，我们需要一些额外的矩阵来进行与照明相关的计算。

```C++
struct Uniforms 
{
    float4x4 modelViewProjectionMatrix; 
    float4x4 modelViewMatrix;
    float3x3 normalMatrix;
};
```

除了模型 - 视图 - 投影矩阵（MVP），我们还存储模型视图矩阵，它将顶点从对象空间转换为眼睛空间。我们需要这个，因为镜面高光是视图相关的。在解析下面的着色器函数时，我们将讨论更多这方面的含义。

我们还包括一个普通矩阵，用于将模型的法线转换为眼睛空间。为什么我们需要一个单独的矩阵呢？答案是法线不像位置那样变换。这看起来似乎违反直觉，但考虑到这一点：当你在空间中移动一个物体而不旋转它时，表面上任何一点的“向外”方向都是不变的。

这给了我们关于如何构造普通矩阵的第一个线索：我们需要从模型视图矩阵中去掉任何转换。我们通过仅采用模型 - 视图矩阵的左上3×3部分来实现这一点，该部分是负责旋转和缩放的部分。

事实证明，只要模型 - 视图矩阵仅由旋转和均匀尺度组成（即，没有拉伸或剪切），该左上方矩阵就是用于变换法线的正确矩阵。在更复杂的情况下，我们需要使用此矩阵的逆转置，但其原因超出了本书的范围。

#### The Vertex Function(顶点函数)

为了存储我们的照明计算所需的每顶点值，我们声明了一个单独的“投影顶点”结构，该结构将由ver-tex函数计算和返回：
```
struct ProjectedVertex { float4 position [[position]]; float3 eye; float3 normal; };
```
眼睛成员是从顶点在视图空间中的位置指向眼睛的矢量
虚拟相机。普通成员是顶点的视图空间法线。顶点函数计算这两个新的向量，以及投影（剪辑 - 
space）每个顶点的位置：
```
vertex ProjectedVertex vertex_project(device Vertex *vertices [[buffer(0)]], constant Uniforms &uniforms [[buffer(1)]],
uint vid [[vertex_id]])
{
    ProjectedVertex outVert;
    outVert.position = uniforms.modelViewProjectionMatrix * vertices[vid].position;
    outVert.eye = -(uniforms.modelViewMatrix * vertices[vid].position).xyz; outVert.normal = uniforms.normalMatrix * vertices[vid].normal.xyz;
    return outVert; 
}
```

#### The Fragment Function(片元函数)

对于每个片段，我们接收一个插值的“投影顶点”结构，并使用它来计算我们的三个照明术语：

```
fragment float4 fragment_light(ProjectedVertex vert [[stage_in]], constant Uniforms &uniforms [[buffer(0)]])
{
    float3 ambientTerm = light.ambientColor * material.ambientColor;
    float3 normal = normalize(vert.normal);
    float diffuseIntensity = saturate(dot(normal, light.direction)); float3 diffuseTerm = light.diffuseColor * material.diffuseColor *
diffuseIntensity;
    float3 specularTerm(0); 
    if (diffuseIntensity > 0) 
    {
        float3 eyeDirection = normalize(vert.eye);
        float3 halfway = normalize(light.direction + eyeDirection); 
        float specularFactor = pow(saturate(dot(normal, halfway)), material.specularPower);
        specularTerm = light.specularColor * material.specularColor * specularFactor;
    }
    return float4(ambientTerm + diffuseTerm + specularTerm, 1); 
}
```

如所讨论的，环境术语仅仅是环境光强度和材料的环境响应的乘积。

由于我们收到的法线和眼睛向量不是单位长度，因此我们在使用前将它们标准化。这必须在每个片段的基础上完成，因为即使我们在顶点函数中对它们进行了归一化，光栅化器完成的插值也不会保留单位长度属性。

为了计算漫反射项，我们首先采用视空间法线和光方向的点积。然后我们使用饱和函数将其夹在0和1之间。否则，指向远离光线的面将会消失，具有负的漫反射强度，这是非物理的。我们将这个余弦规则强度乘以光的漫反射贡献和材料漫反应的乘积，得到完整的扩散项。

为了计算镜面反射因子，我们通过对光方向和眼睛方向求和并对结果进行归一化来计算“中途”矢量。这具有平均它们的效果，因为在添加之前每个都是单位长度。我们采用表面法线和中间矢量之间的点积来得到镜面基底，然后将其提高到材料的镜面反射率以获得镜面反射强度。和以前一样，我们将它与光的镜面反射颜色和材料的镜面反应相乘，得到完整的特定术语。

最后，我们将所有三个项加在一起，并将结果作为片段的颜色值返回。

### The Sample App(样例应用)

本章的示例代码位于05-Lighting目录中。它从OBJ文件加载一个茶壶模型并显示它在3D中翻滚，就像上一章中的立方体一样。但是，这一次，每像素照明应用于模型，使其具有更加合理（并且非常闪亮）的外观。