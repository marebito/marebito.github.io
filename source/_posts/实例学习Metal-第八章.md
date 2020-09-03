---
title: 实例学习Metal-第八章 多维数据集地图的反射和折射
date: 2020-02-23 00:00:00
tags:
  - iOS/Mac
  - 图形学
  - Metal
---

在本章中，我们将讨论Metal中一些更高级的纹理功能。我们将立方体贴图应用于天空盒以模拟周围的详细环境现场。我们还将介绍一种称为立方体环境映射的技术来模拟反射和折射，以进一步增强我们虚拟世界的真实感。

<!-- more -->

# Chapter8(第八章)

## Reflection and Refraction with Cube Maps(多维数据集地图的反射和折射)

在本章中，我们将讨论Metal中一些更高级的纹理功能。我们将立方体贴图应用于天空盒以模拟周围的详细环境现场。我们还将介绍一种称为立方体环境映射的技术来模拟反射和折射，以进一步增强我们虚拟世界的真实感。

首先，我们来谈谈示例场景。

### Setting the Scene(设置场景)

示例应用程序的场景包含围绕圆环结的天空盒。天空盒是一个立方体，其内部具有纹理，以创建围绕相机的大型户外环境的错觉。你可能熟悉圆环，这是一个几何形状，由一个围绕另一个圆圈垂直扫过的圆圈组成。圆环结通过扭转轨道圆圈所采用的路径引入了额外的复杂性，创造出更具视觉趣味的图形。

我们将在程序上生成必要的几何，而不是从磁盘加载模型来构建示例场景。

![](1584066152.png)
<center>图8.1：环形图反映了圆环结。纹理由Emil Persson提供（http://humus.name）</center>

#### The Vertex Format(顶点格式)

虽然我们场景中的对象将被纹理化，但我们不需要在顶点中显式存储纹理坐标。相反，我们将在特制的顶点着色器中生成天空盒和圆环结的纹理坐标。因此，每个顶点只有两个属性：position和normal：

```
typedef struct {
vector_float4 position;
vector_float4 normal; } MBEVertex;
```

#### The Mesh Class(网格类)

正如我们之前所做的那样，我们使用网格对象来包装顶点缓冲区和索引缓冲区，以便我们可以方便地发出索引绘制调用。基类MBEMesh为这两个缓冲区提供了一个抽象接口。

#### The Skybox Mesh(天空盒网格)

生成天空盒网格很简单。天空盒是关于原点的单位立方体，因此天空盒类使用静态的位置，法线和索引列表来构建其缓冲区。请注意，索引按顺序排列以生成指向内部的立方体面，因为虚拟摄像机始终位于天空盒内。

![](1584066187.png)
<center>图8.2：应用于天空盒的立方体环境贴图可以产生详细背景的错觉。</center>

#### The Torus Knot Mesh(圆环结网)

MBETorusKnotMesh类在沿着圆环结的路径扫描时生成顶点，然后生成将顶点编织成实体网格的索引。您可以通过改变传递给网格初始化方法的参数来生成各种有趣的结。

![](1584066211.png)
<center>图8.3：这种三叶形结是一种可以由圆环结网格类产生的结。</center>

现在我们要渲染一些几何体，让我们来谈谈Metal中的一种新纹理。

### Cube Textures(立方体纹理)

立方体纹理（也称为“立方体贴图”）是一种特殊类型的纹理。与常规2D纹理相比，立方体纹理是六个图像的集合，这些图像彼此具有空间关系。特别地，每个图像对应于沿着3D坐标系的一个轴的方向。

在下图中，包含立方体贴图的六个纹理被布置为从内部显示立方体。 -Y脸在十字架的中间。

#### The Cube Map Coordinate Space(立方体地图坐标空间)

Metal中的立方体贴图是左撇子，这意味着如果您想象自己位于立方体内部，右侧是+X面，而上面是+Y面，则+Z轴将是直接前面的那个。相反，如果您位于立方体之外，则坐标系似乎是右手的，并且所有图像都是镜像的。

![](1584066232.png)
<center>图8.4：未包装的立方体贴图。</center>

#### Creating and Loading a Cube Map Texture(创建和加载多维数据集地图纹理)

创建立方体纹理并使用图像数据加载它与创建和加载2D纹理类似，但有一些例外。

第一个区别是纹理类型。 MTLTextureDescriptor有一个方便的方法来创建描述立方体纹理的纹理描述符。描述符的textureType属性设置为MTLTextureTypeCube，高度和宽度属性设置为size参数。立方体纹理的所有组成纹理的宽度和高度必须相等。

```objc
[MTLTextureDescriptor textureCubeDescriptorWithPixelFormat:MTLPixelFormatRGBA8Unorm
size:cubeSize mipmapped:NO];
```

从描述符创建纹理对象的方式与创建2D纹理时的方式相同：

```objc
id<MTLTexture> texture = [device newTextureWithDescriptor:textureDescriptor];
```
2D纹理和立方体纹理之间的另一个区别是图像数据的加载方式。因为立方体纹理实际上包含六个不同的图像，所以我们必须在为每个面设置像素数据时指定切片参数。每个切片对应一个立方体面，具有以下映射：

|  Face number |  Cube texture face |
|---|---|
|  0 | Positive X |
|  1 | Negative X |
|  2 | Positive Y |
|  3 | Negative Y |
|  4 | Positive Z |
|  5 | Negative X |

要将数据加载到每个立方体面，我们迭代切片，用适当的图像数据替换切片的整个区域：

```objc
for (int slice = 0; slice < 6; ++slice>)
{
    // ...
    // fetch image data for current cube face
    // ...
    [texture replaceREgion:region
               mipmapLevel:0
                     slice:slice
                 withBytes:imageData
               bytesPerRow:bytesPerRow
             bytesPerImage:bytesPerImage];
}

`imageData`必须采用纹理描述符中指定的任何像素格式。 `bytesPerRow`是多维数据集的宽度乘以每个像素的字节数。反过来，`bytesPerImage`是每行的字节数乘以多维数据集的高度。由于宽度和高度相等，您可以在这些计算中替换立方体边长。

#### Sampling a Cube Map(采样立方体贴图)

立方体贴图纹理坐标的工作方式与2D纹理坐标略有不同。第一个区别是需要三个坐标来采样立方体贴图，而不是两个。三个坐标被视为起源于立方体中心的射线，与立方体上特定点处的面相交。通过这种方式，立方体纹理坐标表示方向而不是特定点。

以这种方式采样时会出现一些有趣的边缘情况。例如，三角形的三个顶点可能会跨越立方体贴图的角，对三个不同的面进行采样。金属通过沿立方体的边缘和角落正确插入样本来自动处理此类情况。

### Applications of Cube Maps(立方体贴图的应用)

现在我们知道如何创建，加载和采样立方体贴图，让我们来谈谈我们可以使用立方体贴图实现的一些有趣效果。首先，我们需要纹理我们的天空盒。

#### Texturing the Skybox(纹理天空盒)

回想一下，我们从一组静态数据创建了我们的天空盒网格，这些静态数据表示关于原点的单位立方体。单位立方体的角的位置精确映射到立方体贴图的角落纹理坐标，因此可以将立方体的顶点位置重复为其纹理坐标，忽略面法线。

#### Drawing the Skybox(回执天空盒)

渲染天空盒时，一个非常重要的考虑因素是它对深度缓冲区的影响。由于天空盒旨在成为场景的背景，场景中的所有其他对象必须出现在其前面，以免打破幻觉。因此，始终在任何其他对象之前绘制天空框，并禁用写入深度缓冲区。这意味着在天空盒之后绘制的任何对象都用它们自己的像素替换它的像素。
首先绘制天空盒还会引入一个小优化：因为天空盒覆盖了屏幕上的每个像素，所以在绘制场景之前不必清除渲染缓冲区的颜色纹理。

#### The Skybox Vertex Shader(天空盒顶点着色器)

天空盒的顶点着色器非常简单。根据组合模型 - 视图 - 投影矩阵投影位置，并将纹理坐标设置为立方角的模型空间位置。
```
vertex ProjectedVertex vertex_skybox(device Vertex *vertices [[buffer(0)]], constant Uniforms &uniforms [[buffer(1)]], uint vid [[vertex_id]])
{
float4 position = vertices[vid].position;
ProjectedVertex outVert;
outVert.position = uniforms.modelViewProjectionMatrix * position; outVert.texCoords = position;
return outVert;
}
```

#### The Fragment Shader(片段着色器)

我们将对天空盒和圆环结图使用相同的片段着色器。片段着色器只是反转纹理坐标的z坐标，然后在适当的方向上对纹理立方体进行采样：

```
fragment half4 fragment_cube_lookup(ProjectedVertex vert [[stage_in]], constant Uniforms &uniforms [[buffer(0)]],
texturecube<half> cubeTexture [[texture(0)]],
sampler cubeSampler [[sampler(0)]])
{
float3 texCoords = float3(vert.texCoords.x, vert.texCoords.y,
-vert.texCoords.z);
return cubeTexture.sample(cubeSampler, texCoords);
}
```
在这种情况下我们反转z坐标，因为立方体贴图的坐标系是从内部看的左手（借用OpenGL的惯例），但我们更喜欢右手坐标系。这只是示例应用程序采用的惯例;您可能更喜欢使用左手坐标。一致性和正确性比常规更重要。

### The Physics of Reflection(反射物理)

当光从反射表面反弹时发生反射。在这里，我们只考虑完美的镜面反射，即所有光线以与入射角度相等的角度反弹的情况。为了模拟虚拟场景中的反射，我们向后运行此过程，跟踪来自虚拟相机的光线穿过场景，将其从表面反射，并确定它与我们的立方体贴图相交的位置。

![](1584066435.png)
<center>图8.5：光线以等于其入射角的角度反射</center>

#### The Reflection Vertex Shader(反射顶点着色器)

要计算给定矢量的立方体贴图纹理坐标，我们需要找到从虚拟摄像机指向当前顶点的矢量及其法线。这些向量必须都相对于世界坐标系。我们传递到顶点着色器的制服包括模型矩阵以及法线矩阵（它是模型矩阵的逆转置）。这允许我们根据上面给出的公式计算用于找到反射矢量的必要矢量。

幸运的是，Metal包含一个内置函数，可以为我们进行矢量数学运算。 reflect有两个参数：输入向量和表面法向量。它返回反射向量。为了使数学运算正确，两个输入向量都应该提前标准化。

```C++
vertex ProjectedVertex vertex_reflect(device Vertex *vertices [[buffer(0)]], constant Uniforms &uniforms [[buffer(1)]], uint vid [[vertex_id]])
{
    float4 modelPosition = vertices[vid].position; 
    float4 modelNormal = vertices[vid].normal;
    float4 worldCameraPosition = uniforms.worldCameraPosition;
    float4 worldPosition = uniforms.modelMatrix * modelPosition;
    float4 worldNormal = normalize(uniforms.normalMatrix * modelNormal); 
    float4 worldEyeDirection = normalize(worldPosition - worldCameraPosition);
    ProjectedVertex outVert;
    outVert.position = uniforms.modelViewProjectionMatrix * modelPosition; 
    outVert.texCoords = reflect(worldEyeDirection, worldNormal);
    return outVert; 
}
```

输出顶点纹理坐标由硬件插值并传递给我们已经看到的片段着色器。这会产生每像素反射的效果，因为片段着色器的每次执行都使用一组不同的纹理坐标来绘制立方体贴图。

#### The Physics of Refraction(折射物理学)

我们将建模的第二个现象是折射。当光从一种介质传递到另一种介质时发生折射。一个日常的例子是稻草如何在一杯水中弯曲。

![](1584066459.png)
<center>图8.6：通过圆环结折射的环境贴图</center>

我们不会在这里详细讨论折射的数学，但是折射的基本特征是光的弯曲量与它所经过的两种介质的折射率之间的比率成比例。物质的折射率是无单位量，表征相对于真空，它减慢了光的传播速度。空气指数接近1，而水指数约为1.333，玻璃指数约为1.5。

折射遵循一项称为Snell定律的定律，该定律指出入射角和透射角的正弦比等于介质折射率的反比：
sin(θI)/sin(θT) = η<sub>1</sub>/η<sub>0</sub>

![](1584066538.png)
<center>图8.7：当光线在具有不同折射率的物质之间通过时，光线会弯曲</center>

#### The Refraction Vertex Shader(折射顶点着色器)

用于折射的顶点着色器与用于反射的顶点着色器相同，不同之处在于我们使用不同的内置函数refract。我们提供了入射矢量，表面法线和两种物质之间的比例（空气和玻璃，在样本代码的情况下）。返回折射矢量并将其用作立方体贴图纹理，如前所述。
```
vertex ProjectedVertex vertex_refract(device Vertex *vertices [[buffer(0)]], constant Uniforms &uniforms [[buffer(1)]], uint vid  [[vertex_id]])
{
    float4 modelPosition = vertices[vid].position; 
    float4 modelNormal = vertices[vid].normal;

    float4 worldCameraPosition = uniforms.worldCameraPosition;
    float4 worldPosition = uniforms.modelMatrix * modelPosition;
    float4 worldNormal = normalize(uniforms.normalMatrix * modelNormal); 
    float4 worldEyeDirection = normalize(worldPosition - worldCameraPosition);

    ProjectedVertex outVert;
    outVert.position = uniforms.modelViewProjectionMatrix * modelPosition; 
    outVert.texCoords = refract(worldEyeDirection, worldNormal, kEtaRatio);
   
    return outVert; 
}
```

### Using Core Motion to Orient the Scene(使用Core Motion定位场景)

本章的示例代码位于09-CubeMapping目录中。

示例应用程序中的场景包含一个被天空盒包围的圆环结。圆环结反射或折射周围的场景;你可以点击两者之间交替。

#### The Motion Manager(运动管理器)

应用程序使用设备的方向旋转场景，以便环面始终显示在中心，环境围绕它旋转。这是通过Core Motion框架中的`CMMotionManager`类实现的。
要使用运动管理器跟踪设备的运动，首先应检查设备是否可用。如果是，您可以继续设置更新间隔并请求运动管理器开始跟踪运动。以下代码执行这些步骤，每秒请求更新60次。

```
CMMotionManager *motionManager = [[CMMotionManager alloc] init]; 
if (motionManager.deviceMotionAvailable)
{
    motionManager.deviceMotionUpdateInterval = 1 / 60.0; 
    CMAttitudeReferenceFrame frame = CMAttitudeReferenceFrameXTrueNorthZVertical;
    [motionManager startDeviceMotionUpdatesUsingReferenceFrame:frame]; 
}
```

#### Reference Frames and Device Orientation(参考框架和设备方向)

参考系是指定设备方向的一组轴。有几种不同的参考框架可供选择。它们中的每一个都将Z标识为垂直轴，而X轴的对齐可以根据应用的需要进行选择。我们选择将X轴与磁北对齐，以便虚拟场景的方向与外界直观对齐，并在应用程序运行之间保持一致。

设备方向可以被认为是相对于参考系的旋转。旋转的基础使其轴与设备的轴对齐，+X指向设备的右侧，+Y指向设备的顶部，+Z指向前方的屏幕。

#### Reading the Current Orientation(读取当前的方向)

每一帧，我们都想获得更新的设备方向，因此我们可以将虚拟场景与现实世界对齐。我们通过从动画管理器中请求CMDeviceMotion的实例来完成此操作。

```
CMDeviceMotion *motion = motionManager.deviceMotion;
```

设备运动对象以几种等效形式提供设备方向：欧拉角，旋转矩阵和四元数。由于我们已经使用矩阵来代表我们的Metal代码中的旋转，因此它们最适合将Core Motion的数据合并到我们的应用程序中。

每个人都有自己对于哪个视图坐标空间最直观的看法，但我们总是选择右手系统，+ Y朝上，+ X朝右，+ Z指向观察者。这与Core Motion的惯例不同，因此为了使用设备方向数据，我们按照以下方式解释Core Motion的轴：Core Motion的X轴成为我们世界的Z轴，Z变为Y，Y变为X.我们不必镜像任何轴，因为Core Motion的参考框架都是右手的。

```
CMRotationMatrix m = motion.attitude.rotationMatrix;

vector_float4 X = { m.m12, m.m22, m.m32, 0 }; 
vector_float4 Y = { m.m13, m.m23, m.m33, 0 }; 
vector_float4 Z = { m.m11, m.m21, m.m31, 0 };
vector_float4 W = { 0, 0, 0, 1 };

matrix_float4x4 orientation = { X, Y, Z, W }; renderer.sceneOrientation = orientation;
```

渲染器将​​场景方向矩阵合并到它为场景构造的视图矩阵中，从而使场景与物理世界保持一致。

### The Sample App(示例应用)

本章的示例代码位于08-CubeMapping目录中。