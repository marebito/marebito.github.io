---
title: 理解ARKit
date: 2018-06-03 13:46:41
tags:
  - AR
  - 增强现实
top_img: http://up.bizhitupian.com/pic_source/87/90/d5/8790d5468b68ebdd3321b0bbe6a5534b.jpg
---

近些年最流行的手游之一是Pokemon Go(精灵宝可梦Go), 它展示通过iPhone相机查看覆盖在真实地点的宝可梦角色。将iPhone摄像头对准公园长椅或灌木丛，你可以看到一个宝可梦卡通角色，好像它真的在那。

这种在实际物理对象上显示虚拟对象的技术被称为增强现实（AR）。增强现实背后的想法是让您将真实世界的物体与出现在iPhone屏幕上的虚拟物体相结合。

<!-- more -->

增强现实的一个用途是将iPhone指向街道，以便您可以看到周边地区的街道名称和企业名称。另一个用途是向您展示带箭头的步行路线，向您展示如何在大型公共场所（如机场航站楼或博物馆）进行导航。

以前，创建增强现实应用需要编写数学方程式来跟踪真实世界物体和虚拟物体在真实世界中的位置。幸运地，苹果已经通过一个新的叫做ARKit的软件框架使得增强现实更简单地去创建。通过使用ARKit和其他框架，例如SceneKit，你可以简单快捷地创建增强现实应用。

> **注意** 你仅可以测试和运行ARKit应用在iPhone6s或者更高，或者一台iPad Pro上。

## ARKit如何工作

在最简单的层面上，ARKit通过识别称为特征点的周围区域来工作。一旦ARKit了解iOS设备摄像头查看的物理对象，它就可以将虚拟对象叠加在摄像头显示的实际图像的顶部。

查看实例中的ARKit示例，创建一个新的iOS项目。但是，请确保您单击了增强现实应用程序项目，如图1所示。然后点击下一步。

![](https://i.loli.net/2018/06/10/5b1cc34f55d74.jpg)

**图 1** 增强现实应用模板

当Xcode对于你的项目要求一个产品名称，输入ARTest。确保Content Technology弹出菜单显示SceneKit。（Content Technology的其他两个选项是Metal和SpriteKit。SpriteKit为2D图像而设计，然而Metal专为喜欢创建自己的代码来创建图形的高级用户而设计。在大多数情况下，你将使用SceneKit来显示3D图像。）

当你点击Finish按钮时，Xcode创建了一个标准的iOS项目，它包含一个AppDelegate.swift文件，一个ViewController.swift文件，和一个Main.storyboard用户界面文件。

点击ViewController.swift文件，你将看到已经为你编写好创建增强现实应用的代码。注意ViewController.swift包含三个重要的语句：

```swift
import UIKit
import SceneKit
import ARKit
```

UIKit创建一个iOS应用。使用SceneKit可以显示三维物体。ARKit可让您将增强现实添加到您的应用中。

接下来，注意ViewController累采用ARSCNViewDelegate：

```swift
class ViewController : UIViewController, ARSCNViewDelegate {
```

该协议允许您将SceneKit图像显示为覆盖真实世界对象的增强现实对象。然后，注意ViewController.swift文件已经包含一个单独的名为sceneView的IBOutlet：

```swift
@IBOutlet var sceneView : ARSCNView
```

如果你点击了Main.storyboard文件，你会看到一个ARKit SceneKit View已经在用户界面中，如图2所示。此ARSCNView在相机背景上显示3D SceneKit图像。

![](https://i.loli.net/2018/06/10/5b1cc84dc3ca5.jpg)

**图 2** 一个ARKit SceneKit View已经出现在用户界面上

在ViewController.swift文件中，请查看viewWillAppear函数，您将看到两行代码可帮助在应用程序中创建增强现实。第一行代码创建了一个叫做configuration的常量，它代表一个ARWorldTrackingConfiguration对象。该对象跟踪iOS设备的方向和位置，以及检测真实世界的表面。

```swift
let configuration = ARWorldTrackingConfiguration()
```

第二行实际显示叠加在相机显示的视图上的增强现实图像：

```swift
sceneView.session.run(configuration)
```

现在来看ViewController.swift文件中的viewDidLoad方法。首先，在ViewController.swift文件中有一行定义了其delegate（代理）。其次，有一行显示了在屏幕底部的每秒帧数（fps）和时间数据。

```swift
sceneView.showStatistics = true
```

如果你看Navigator面板，你将会看到art.scnassets文件夹。如果你展开这个文件夹，你将会看到它包含了两个文件：ship.scn和texture.png。ship.scn文件包含了一个三维的对象，你可以通过修改来改变其外观。texture。png文件包含了出现在ship.scn模型中的图像，如图3.

![](https://i.loli.net/2018/06/10/5b1ccb2f458d4.jpg)

**图 3** 查看Xcode中的ship.scn文件

> **注意** .scn文件代表了一个SceneKit文件格式。大多数3D数字图像程序可以保存文件为一个.dae（数字资产交换）文件格式。如果你添加一个.dae文件到Xcode中，你可以转换它成为一个.scn文件格式，通过添加一个.dae文件到Navigator面板，然后选择File ➤ Export，保存文件为一个.scn文件。如果你想创建.dae文件，你可以使用免费，开源的Blender程序（[www.blender.org]()）。你也同样可以在互联网上找到免费的公共域的.dae文件。

虚拟对象由一个模型（在这种情况下，ship.scn文件）和一个纹理（texture.png）组成，并应用于该模型。如果你点击texture.png文件，你将看到出现在ship.scn文件中的颜色和图像。

点击ship.scn文件，点击Xcode窗口中的飞机图像。点击Show the Materials Inspector图标（或者选择View ➤ Utilities ➤ Show the Materials Inspector)。这将显示“材质检查器”窗格，该窗格允许您修改模型的外观。

在Properties中点击Diffuse弹出菜单，你可以看到texture.png被选择如果4所示。点击黑或者白来移除纹理，以便于你可以看到如何从模型中移除texture.png文件来改变ship.scn文件的外观。

![](https://i.loli.net/2018/06/10/5b1cce952bed5.jpg)

**图 4** “漫反射“弹出菜单定义ship.scn文件的外观

确保Diffuse弹出菜单再次先吃了texture.png.通过USB线连接一台iOS设备到你的Mac电脑并将您的iOS设备相机指向任何地方。ship.scn文件的虚拟镜像现在应该出现在摄像机查看的真实世界对象上，如图5所示。

![](https://i.loli.net/2018/06/10/5b1d081d2fb1b.jpg)

**图5** 在iPhone上运行ARTest项目

当你四处移动iOS设备时，你应当看到了ship.scn的不同角度图像好像它是你面前的一个真实物体一样。注意屏幕底部显示的关于增强显示图像的统计，比如它的帧率（fps）。

回到Xcode，选择Product ➤ Stop或者点击停止图标来停止运行中的ARTest项目。在此刻，你已经看到了增强现实如何工作的一个简单的演示。通过添加不同纹理文件或者使用不同的图片替换ship.scn文件，你可以显示通过iOS设备摄像头看到的自定义图像。

返回并且编辑ViewController.swift文件。首先，注释掉定义ship.scn文件的两行，然后加载该场景：

```swift
// let scene = SCNScene(named: "art.scnassets/ship.scn")!
// sceneView.scene = scene
```

现在添加下面的行来显示覆盖在相机图像上的特征点和世界原点：

```swift
sceneView.debugOptions = [ARSCNDebugOpitions.showWorldOrigin, ARSCNDebugOptions.showFeaturePoints]
```

特征点显示为突出显示ARKit识别的表面区域的黄色点。世界原点显示为x轴，y轴，z轴，其中x轴向右，y轴向上，并且z轴指向用户的屏幕外。

> **注意** ARKit在清晰的照明条件下工作得最好，多个物体可见，因此它可以检测桌子，地板和墙壁的表面区域。较差的照明条件会阻碍ARKit识别表面区域并将摄像机指向空白墙壁或地板的能力。

你的整个viewDidLoad方法应该看起来像这样：

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    // 设置视图的代理
    sceneView.Delegate = self
    
    // 显示统计例如fps和时间信息
    sceneView.showStatistics = true
    
    // 创建一个新的场景
    // let scene = SCNScene(named:"art.scnassets/ship.scn")!
    
    // 设置视图的场景
    // sceneView.scene = scen
    
    sceneView.debugOptions = [ARSCNDebugOptions.showWorldOrigin, ARSCNDebugOptions.showFeaturePoints]
}
```

当你运行这个ARTest项目的修改版本时，你将在附近区域看到原点显示以及黄点，表示特征点，如图6所示。

![](https://i.loli.net/2018/06/10/5b1d0c2c95bcb.jpg)

**图 6** debugOptions行显示原点和特征点

## 绘制增强现实物体

通过用你自己的图像替换ship.scn文件，你可以显示任何你想作为一个增强现实的物体。然而，你也可以现实简单的几何体。一些你可以绘制的可用模型包括：

* SCNBox - 绘制一个盒子
* SCNCapsule - 绘制一个圆柱体，其两端用半球盖住
* SCNCone - 绘制一个圆锥体
* SCNCylinder - 绘制一个圆柱体
* SCNFloor - 绘制一个可以选择反映场景的无限平面
* SCNPlane - 绘制一个指定宽高的矩形平面
* SCNPyramid - 绘制一个角锥体
* SCNSphere - 绘制一个球体
* SCNTorus - 绘制一个圆环状的物体
* SCNTube - 沿其中心轴绘制一个带有孔的圆柱体
显示几何模型时，您需要定义三个特征：
* 物体的物理尺寸例如宽高
* 物体的外观例如颜色
* 物体相对于世界原点的位置

一旦你定义一个物体的大小，外观，和位置，您需要将其放置在相机显示的视图上。为了做到这点，你需要去定义一个节点。要查看它是如何工作的，我们需要将以下代码添加到ViewController.swift文件中现有的viewDidLoad函数中：

在sceneView.debugOptions行的下面，添加以下代码：

```swift
let node = SCNNode()
node.geometry = SCNPyramid(width: 0.1, height: 0.2, length: 0.1)
node.geometry?.firstMaterial?.diffuse.contents = UIColor.cyan
node.position = SCNVector3(0, -0.2, 0)
sceneView.scene.rootNode.addChildNode(node)
```

第一行创建一个节点，它定义几何模型将会出现在哪里。
第二行定义了一个有宽，高和长的角锥体。
第三行定义了模型的颜色，它是青色。
第四行定义了角锥体相对于世界原点的位置。在这种情况下，金字塔底部出现在原点下面-0.2处。
第五行将节点添加到场景中，以便青色金字塔直接出现在世界原点的下方如图7所示。

![](https://i.loli.net/2018/06/10/5b1d10f979470.jpg)

**图7** 显示一个角锥体作为增强现实物体

通过更改node.position的值以及角锥体的宽度，高度和长度来进行实验。还要将其颜色从青色更改为红色或黄色。选择不同的模型，如SCNBox，SCNTub或SCNCone，而不是显示角锥体，

## 重置世界原点

我们创建ARTest项目使用增强现实应用模板，但是我们可以通过添加ARKit和SceneKit框架轻松地为任何项目提供增强现实功能。创建一个新的项目并创建一个新的单视图应用。给它一个ARRest的名字。

当你第一次运行任何AR应用时，它会在您的iPhone或iPad的当前位置定义世界原点。如果稍后退出，您会看到屏幕上显示的原点（请参见图6）。不幸地，世界原点将仍然固定直到你再次运行应用。

为了修复这个问题，你将创建的下一个项目将显示一个重置按钮，该按钮可让你将iPhone / iPad移动到新位置并在新位置重新定义原点。这样，你可以重新定义世界原点位置，而无需重新启动应用程序。

点击ViewController.swift文件并添加SceneKit和ARKit框架像这样：

```swift
import SceneKit
import ARKit
```

像这样采用ARSCNViewDelegate修改ViewController类：

```swift
class ViewController : UIViewController, ARSCNViewDelegate {
```

点击Info.plist文件并添加一个Privacy – Camera Usage Description的键。如果你没有做到这一点，您的应用将无法访问相机，并且无法运行。

点击Main.storyboard文件并添加一个ARKit SceneKit View到用户界面如图8所示。

![](https://i.loli.net/2018/06/10/5b1d3ef4a9b00.jpg)

**图 8** ARKit SceneKit View现实增强现实物体

将UIButton拖放到ARKit SceneKit下面的屏幕底部查看以便延伸视图的宽度。给这个按钮一个“复位”的标题，如图9所示。然后选择Editor ➤ Resolve Auto Layout Issue ➤ Reset to Suggested Constraints。

![](https://i.loli.net/2018/06/10/5b1d3faa7c584.jpg)

**图 9** 在用户界面上放置一个UIButton

打开Assistant Editor并按住Control从ARSCNView拖拽到创建的名为sceneView的IBOutlet:

```swift
@IBOutlet var sceneView : ARSCNView!
```

现在，从UIButton拖动Control来创建一个名为resetAR的IBAction方法，并添加如下代码：

```swift
@IBAction func resetAR(_ sender : UIButton) {
    sceneView.session.pause()
    sceneView.session.run(configuration, options : [.resetTracking, .removeExistingAnchors])
```

整个ViewController.swift文件应当看起来像这样：

```swift
import UIKit
import SceneKit
import ARKit

class ViewController: UIViewController, ARSCNViewDelegate {

    @IBOutlet weak var sceneView: ARSCNView!
    
    // 创建一个session配置
    let configuration = ARWorldTrackingConfiguration()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        sceneView.delegate = self
        
        // 显示统计例如fps和时间信息
        sceneView.showsStatistics = true
        
        // 显示原点和特征点
        sceneView.debugOptions = [ARSCNDebugOptions.showWorldOrigin, ARSCNDebugOptions.showFeaturePoints]
    }
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        
        //运行视图的会话
        sceneView.session.run(configuration)
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        
        // 暂停视图的会话
        sceneView.session.pause()
    }

    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        // Dispose of any resources that can be recreated.
    }

    @IBAction func resetAR(_ sender: UIButton) {
        sceneView.session.pause()
        sceneView.session.run(configuration, options: [.resetTracking, .removeExistingAnchors])
    }
}
```

当你运行这个应用时，稍后退一步，你会看到世界的起源显示在你面前。移至新位置并点击重置按钮，然后向后退出。 您现在应该看到在新位置定义的世界原点。

## 绘制自定义模型

ARKit提供了可以创建的常见几何模型，例如圆柱体，锥体，金字塔，盒子和球体。如果这些几何模型都不符合您的需求，您可以通过定义一个起点并添加线条来创建一个模型来绘制自己的图形。绘制线来定义模型创建了所谓的贝塞尔路径。

创建贝塞尔路径的四个主要步骤包括：

* 定义贝塞尔路径
* 定义绘制起点
* 绘制一条或多条线
* 根据您定义的Bezier路径定义SCNShape

为了创建一个贝塞尔路径，你需要定义一个BezierPath对象像这样:

```swift
let path = UIBezierPath()
```

一旦你已经创建了一个贝塞尔路径，你需要定义它的起点像这样：

```swift
path.move(to: CGPoint(x: 0, y: 0))
```

现在我们需要绘制一条或者多条线使用addLine方法，它定义了终点像这样：

```swift
path.addLine(to: CGPoint(x: 0.2, y: 0.2))
```

最终我们需要将贝塞尔曲线变成一个图形：

```swift
let shape = SCNShape(path: path, extrusionDepth: 0.1)
```

一旦我们有一个图形，我们可以显示它作为一个增强现实物体，通过定义它作为一个带有颜色和位置的节点。然后我们可以最终添加节点到增强现实视图：

```swift
let node = SCNNode()
node.geometry = shape
node.geometry?.firstMaterial?.diffuse.contents = UIColor.yellow
node.position = SCNVector3(0, 0, -0.4)
sceneView.scene.rootNode.addChildNode(node)
```

在你的ARReset项目中修改ViewController.swift文件，它的全部内容看起来像下面这样：

```swift
import UIKit
import SceneKit
import ARKit

class ViewController: UIViewController, ARSCNViewDelegate {

    @IBOutlet var sceneView: ARSCNView!
    
    //创建一个会话配置
    let configuration = ARWorldTrackingConfiguration()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        sceneView.delegate = self
        
        // 显示统计例如帧率和时间信息
        sceneView.showStatistics = true
        
        // 显示原点和特征点
        sceneView.debugOptions = [ARSCNDebugOptions.showWorldOrigin, ARSCNDebugOptions.showFeaturePoints]
        
        let path = UIBezierPath()
        path.move(to: CGPoint(x: 0, y: 0))
        path.addLine(to: CGPoint(x: 0.2, y: 0.2))
        path.addLine(to: CGPoint(x: 0.4, y: -0.2))
        let shape = SCNShape(path: path, extrusionDepth: 0.1)
        let node = SCNNode()
        node.geometry = shape
        node.geometry?.firstMaterial?.diffuse.contents = UIColor.yellow
        node.position = SCNVector3(0, 0, -0.4)
        sceneView.scene.rootNode.addChildNode(node)
    }
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        
        // 运行视图的会话
        sceneView.session.run(configuration)
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        
        // 暂停视图会话
        sceneView.session.pause()
    }
    
    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        // 处置任何可以重新创建的资源
    }
    
    @IBAction func resetAR(_ sender: UIButton) {
        sceneView.session.pause()
        sceneView.session.run(configuration, options: [.resetTracking, .removeExistingAnchors])
    }
}
```

如果你运行这个修改过的ARReset项目在连接到你的Mac的一台iOS设备上，你应当看到一个黄色的三角形超过了世界原点如图10所示。

![](https://i.loli.net/2018/06/13/5b2056ffc51ad.jpg)

**图 10** 使用贝塞尔曲线绘制一个自定义图形

## 修改模型外观

到现在为止，我们仅仅创建了一个图形和应用颜色到它上面，但是有其他方式修改模型的外观。修改一个模型的外观一些方式包括改变光照，透明度，或者文理。光照使得模型看起来不同，这取决于光照的类型和光源的位置。透明度定义了模型是否显示为实心或透明。纹理在模型的两侧应用图形图像，例如使模型看起来像用砖块或沙子制成。通过修改模型的外观，可以使该模型在视觉上更有趣。

要试验修改模型的外观，请创建一个新的增强现实应用项目并将其命名为ARAppearance。确保Content Technology使用SceneKit。我们要修改对象外观的第一种方法是使用显示在该模型上的图形图像。

在互联网上搜索“公共领域纹理图像”，你会发现许多图像，你可以自由下载和使用。纹理图像通常显示一个规则图案，如砖，水，田地，或如图11所示的材料，如木头或石头。

![](https://i.loli.net/2018/06/14/5b21aa8771c7f.jpg)

**图 11** 搜索公共领域的纹理图像

下载一个纹理图片并确保他被存储为png或者jpg文件格式。然后拖拽它到Navigator面板如图12所示。

![](https://i.loli.net/2018/06/14/5b21aabac5519.jpg)

**图 12** 放置一张文理图片文件在Navigator面板

修改ViewController.swift文件显示如下：

```swift
import UIKit
import SceneKit
import ARKit

class ViewController: UIViewController, ARSCNViewDelegate {

    @IBOutlet var sceneView: ARSCNView!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Set the view's delegate
        sceneView.delegate = self
        
        // Show statistics such as fps and timing information
        sceneView.showsStatistics = true
        
        sceneView.debugOptions = [ARSCNDebugOptions.showWorldOrigin, ARSCNDebugOptions.showFeaturePoints]
        
        let box = SCNBox(width: 0.1, height: 0.1, length: 0.2, chamferRadius: 0.01);
        let node = SCNNode()
        let material = SCNMaterial()
        material.diffuse.contents = UIImage(named: "stone.jpg")
        box.materials = [material]
        
        node.geometry = box
        node.position = SCNVector3(0, 0, -0.3)
        
        sceneView.scene.rootNode.addChildNode(node)
        
        // Create a new scene
        let scene = SCNScene(named: "art.scnassets/ship.scn")!
        
        // Set the scene to the view
        sceneView.scene = scene
    }
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        
        // Create a session configuration
        let configuration = ARWorldTrackingConfiguration()

        // Run the view's session
        sceneView.session.run(configuration)
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        
        // Pause the view's session
        sceneView.session.pause()
    }
    
    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        // Release any cached data, images, etc that aren't in use.
    }

    // MARK: - ARSCNViewDelegate
    
/*
    // Override to create and configure nodes for anchors added to the view's session.
    func renderer(_ renderer: SCNSceneRenderer, nodeFor anchor: ARAnchor) -> SCNNode? {
        let node = SCNNode()
     
        return node
    }
*/
    
    func session(_ session: ARSession, didFailWithError error: Error) {
        // Present an error message to the user
        
    }
    
    func sessionWasInterrupted(_ session: ARSession) {
        // Inform the user that the session has been interrupted, for example, by presenting an overlay
        
    }
    
    func sessionInterruptionEnded(_ session: ARSession) {
        // Reset tracking and/or remove existing anchors if consistent tracking is required
        
    }
}
```

这份代码定义了一个SCNBox的几何模型，同时也定义了一个SCNMaterial的数组。那是因为一个模型可以有多个材质。然后代码定义了图形文件叫做"stone.jpg"作为它的第一材质，创建一个石头图像在模型周围如图13所示：

![](https://i.loli.net/2018/06/14/5b21b9b7477dc.jpg)

**图 13** 一块石头图像以包围盒子模型的材料形式出现

另外一种修改模型外观的方式是改变它的透明度，使用一个0（不可见）到1（纯色）。添加一行代码来定义一个透明度值，所以你整个viewDidLoad方法看起来像这样：

```swift
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Set the view's delegate
        sceneView.delegate = self
        
        // Show statistics such as fps and timing information
        sceneView.showsStatistics = true
        
        sceneView.debugOptions = [ARSCNDebugOptions.showWorldOrigin, ARSCNDebugOptions.showFeaturePoints]
        
        let box = SCNBox(width: 0.1, height: 0.1, length: 0.2, chamferRadius: 0.01);
        let node = SCNNode()
        let material = SCNMaterial()
        material.diffuse.contents = UIImage(named: "stone.jpg")
        material.transparency = 0.7
        box.materials = [material]
        
        node.geometry = box
        node.position = SCNVector3(0, 0, -0.3)
        
        sceneView.scene.rootNode.addChildNode(node)
    }
    ```
    
    上面的代码用材质数组定义了两个特性。首先，它显示了“stone.jpg”图像在盒子周围。其次，它定义了一个0.7的透明度，所以盒子如果14所示出现半透明。
    
    ![](https://i.loli.net/2018/06/14/5b21b9b7477dc.jpg)
**图 14** 改变透明度使得图形看起来减少纯色

另外应用纹理和定义一个透明度层次，另外一种去改变模型外观的方式是通过光照。光照让你创建一个光源，它照亮附近的模型。依赖你选择的光照和光照位置，你可以在一个图形上创建不同类型可视效果。

创建一个光源，你需要做以下这些：

* 定义一个SCNLight对象
* 定义SCNLight类型
* 分配SCNLight对象到一个SCNNode上
* 定义SCNNode位置
* 添加SCNNode到场景

定义一个SCNLight对象，你只需要像这样创建一个常量：
```swift
let spotLight = SCNLight()
```

现在定义下列光照类型中的一个：

* ambient
* directional
* IES
* probe
* spot

每种光照类型以不同的方式突出模型，所以让我们通过直射光类型来开始实验：

```swift
spotLight.type = .directional
```

既然你已经定义了一个光照类型，你需要像这样创建一个SCNNode：

```swift
let spotNode = SCNNode()
spotNode.light = spotLight
```

第一行创建一个SCNNode对象，第二行定义了它的光源作为一个聚光灯（SCNLight）我们前面创建的对象。

最终，我们可以防止一个SNNode对象基于世界原点。那意味着你需要定义一个X，Y和Z值像这样：

```swift
spotNode.position = SCNVector3(0, 0.2, 0)
```

上述代码放置光源在原点上面0.2米的位置，所以光照耀下来在我们将要创建的盒子上。

最后一步是添加这个光源节点到增强现实场景上：

```swift
sceneView.scene.rootNode.addChildNode(spotNode)
```

整个viewDidLoad函数应该看起来像这样：

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    // Set the view's delegate
    sceneView.delegate = self;
    
    // Show statistics such as fps and timing information
    sceneView.showStatistics = true
    
    sceneView.debugOptions = [ARSCNDebugOptions.showWorldOrigin, ARSCNDebugOptions.showFeaturePoints]
    
    let box = SCNBox(width: 0.1, height: 0.1, length: 0.1, chamferRadius: 0.01)
    let node = SCNNode()
    let material = SCNMaterial()
    // material.diffuse.contents = UIImage(named: "stone.jpg")
    // material.transparency = 0.7
    
    let spotLight = SCNLight()
    spotLight.type = .directional
    let spotNode = SCNNode()
    spotNode.light = spotLight
    spotNode.position = SCNVector3(0, 0.2, 0)
    
    material.diffuse.contents = UIColor.orange
    box.materials = [material]
    
    node.geometry = box
    node.position = SCNVector3(0, 0, -0.3)
    
    sceneView.scene.rootNode.addChildNode(node)
    sceneView.scene.rootNode.addChildNode(spotNode)
}
```

如果你运行这个项目，它将创建一个橙色的盒子放置在原点后面0.3米的地方。然后光源将出现在原点上面0.2米的地方照耀下来到这个橙色的盒子上。因为光照类型是directional，它仅仅照亮了盒子的表面如果15所示。

    ![](https://i.loli.net/2018/06/14/5b21b9b7477dc.jpg)
**图 15** 定向光聚焦在橙色盒子前面

查看不同类型光如何改变模型外观，改变光源类型从directional到omni像这样：
// spotLight.type = .directional // 只照亮盒子前面
spotLight.type = .omni //照亮盒子的顶部和前面

现在如果你运行这个项目，omni光照类型照亮盒子的前面和顶部如果16所示：

    ![](https://i.loli.net/2018/06/14/5b21b9b7477dc.jpg)
**图 16** omni光照类型照亮橙色盒子前面和顶部

苹果的文档定义了不同光照类型应有的表现，所以用改变光照类型和光源位置来实验。通过改变光源位置，你可以照亮模型的不同区域。通过简单的改变光照类型，你可以用不同的方式照亮一个物体如图17所示。

    ![](https://i.loli.net/2018/06/14/5b21b9b7477dc.jpg)
**图 16** 不同光照类型在照亮模型上起不同的作用

# 总结

在这个章节，你已经学习了使用ARKit创建增强现实物体的基础。你已经学会了如果放置一个增强现实物体在一个场景和如何改变一个物体外观，通过改变它们的颜色，透明度和纹理。另外，你也学习了如何绘制你自己的物体和使用一个光源照亮一个物体。

增强现实给予你的应用在一个真实场景中覆盖真实物体的能力。在下一章中，你将学习如何用增强现实对象去交互，以便于你可以控制和操作他们。





