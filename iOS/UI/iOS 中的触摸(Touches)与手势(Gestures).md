# iOS 中的触摸(Touches)与手势(Gestures)

如果你使用标准 UIKit 视图和控件构建应用程序，UIKit 会自动为你处理触摸事件（包括多点触控事件）。但是，如果使用自定义视图显示内容，则必须处理视图中发生的所有触摸事件。有两种方法可以自己处理触摸事件。

- 使用手势识别器来跟踪触摸事件;
- 直接跟踪 UIView 子类中的触摸事件;

## 响应者(Responders)与响应者(Responder Chain)

应用程序使用响应者对象接收和处理事件。响应者对象是 UIResponder 类的任何实例，公共子类包括 UIView，UIViewController 和 UIApplication。响应者接收原始事件数据，并且必须处理事件或将其转发给另一个响应者对象。当你的应用收到事件时，UIKit 会自动将该事件定向到最合适的响应者对象，称为第一响应者。

未处理的事件从响应者传递到活动响应者链中的响应者，这是应用程序的响应者对象的动态配置。图1显示了应用程序中的响应者，其界面包含 label，text field，button 和两个 background views。该图还显示了事件如何在响应者链之后从一个响应者移动到下一个响应者。

图 1 应用程序中的响应者链

![](https://docs-assets.developer.apple.com/published/7c21d852b9/f17df5bc-d80b-4e17-81cf-4277b1e0f6e4.png)

如果 text field 不处理事件，UIKit 会将事件发送到 text field 的父 UIView 对象，然后是 window 的根视图。从根视图开始，响应器链在将事件定向到窗口之前转移到拥有的视图控制器。如果 window 无法处理事件，UIKit 会将事件传递给 UIApplication 对象，如果该对象是 UIResponder 的实例而不是响应者链的一部分，则可能传递给 app delegate。

### 确定事件的第一响应者

UIKit 根据事件的类型将对象指定为事件的第一响应者。

> 注意

> 与加速计，陀螺仪和磁力计相关的运动事件不遵循响应链。相反，Core Motion 将这些事件直接传递给指定对象。

控件使用操作消息直接与其关联的目标对象进行通信。当用户与控件交互时，控件会向其目标对象发送操作消息。动作消息不是事件，但它们仍然可以利用响应者链。当控件的目标对象为 nil 时，UIKit 从目标对象开始并遍历响应者链，直到找到实现相应操作方法的对象。例如，UIKit 编辑菜单使用此行为来搜索响应器对象，这些对象实现具有 cut(_:), copy(_:), 或 paste(_:) 等名称的方法。

手势识别器在其视图之前接收触摸和按压事件。如果视图的手势识别器无法识别触摸序列，则 UIKit 会将触摸发送到视图。如果视图没有处理触摸，UIKit 会将它们传递给响应者链。

### 确定哪个响应者包含触摸事件

UIKit 使用基于视图的命中测试来确定触摸事件发生的位置。具体来说，UIKit 将触摸位置与视图层次结构中视图对象的边界进行比较。UIView 的 hitTest(_:with:) 方法遍历视图层次结构，查找包含指定触摸的最深的子视图，该子视图成为触摸事件的第一个响应者。

> 注意

> 如果触摸位置在视图边界之外，则 hitTest(_:with:) 方法会忽略该视图及其所有子视图。因此，当视图的 clipsToBounds 属性为 false 时，即使它们恰好包含触摸，也不会返回该视图边界之外的子视图。

触摸发生时，UIKit 会创建一个 UITouch 对象并将其与视图关联。当触摸位置或其他参数发生变化时，UIKit 会使用新信息更新相同的 UITouch 对象。唯一不改变的属性是视图。（即使触摸位置移动到原始视图之外，触摸视图属性中的值也不会改变。）触摸结束时，UIKit 会释放 UITouch 对象。

### 改变响应者链

你可以通过覆盖响应者对象的 next 属性来更改响应者链。执行此操作时，下一个响应者是你返回的对象。

许多 UIKit 类已经覆盖此属性并返回特定对象，包括：

- UIView 对象。如果视图是视图控制器的根视图，则下一个响应者是视图控制器; 否则，下一个响应者是视图的父视图。
- UIViewController 对象。
  - 如果视图控制器的视图是窗口的根视图，则下一个响应者是窗口对象。
  - 如果视图控制器由另一个视图控制器 presented，则下一个响应者是 the presenting view controller。
- UIWindow 对象。窗口的下一个响应者是 UIApplication 对象。
- UIApplication 对象。下一个响应者是 app delegate，但仅当 app delegate 是 UIResponder 的实例且不是视图，视图控制器或应用程序对象本身时。

## 处理视图中的触摸

如果你不打算在自定义视图中使用手势识别器，则可以直接从视图本身处理触摸事件。由于视图是响应者，因此他们可以处理 Multi-Touch 事件和许多其他类型的事件。当 UIKit 确定视图中发生触摸事件时，它会调用视图的 touchesBegan(_:with:), touchesMoved(_:with:), 或 touchesEnded(_:with:)方法。你可以在自定义视图中覆盖这些方法，并使用它们来提供对触摸事件的响应。

你在视图中（或在任何响应者中）覆盖以处理触摸的方法对应于触摸事件处理过程的不同阶段。例如，图2说明了触摸事件的不同阶段。 当手指（或 Apple Pencil）触摸屏幕时，UIKit 会创建一个 UITouch 对象，将触摸位置设置为适当的点，并将其相位属性设置为 UITouch.Phase.began。当同一个手指在屏幕上移动时，UIKit 会更新触摸位置并将触摸对象的相位属性更改为 UITouch.Phase.moved。当用户从屏幕上抬起手指时，UIKit 会将相位属性更改为 UITouch.Phase.ended，触摸序列结束。

图 2 触摸事件的阶段

![](https://docs-assets.developer.apple.com/published/7c21d852b9/08b952fe-6f46-41eb-8b8a-4830c1d48842.png)

类似地，系统可以在任何时间取消正在进行的触摸序列; 例如，当来电打电话中断应用程序时。如果是这样，UIKit 会通过调用 touchesCancelled(_:with:) 方法通知你的视图。你可以使用该方法执行视图数据结构的任何所需清理。

UIKit 为每个接触屏幕的新手指创建一个新的 UITouch 对象。触摸本身随当前的 UIEvent 对象一起提供。UIKit 区分源自手指和 Apple Pencil 的触摸，你可以区别对待它们。

> 重要

> 在其默认配置中，视图仅接收与事件关联的第一个 UITouch 对象，即使多个手指正在触摸视图。要接收其他触摸，必须将视图的 isMultipleTouchEnabled 属性设置为 true。

## 处理 UIKit 手势

手势识别器是处理视图中触摸或按下事件的最简单方法。你可以将一个或多个手势识别器附加到任何视图。手势识别器封装了处理和解释该视图的传入事件所需的所有逻辑，并将它们与已知模式相匹配。检测到匹配时，手势识别器会通知其分配的目标对象，该目标对象可以是视图控制器，视图本身或应用程序中的任何其他对象。

手势识别器使用 target-action 设计模式发送通知。当 UITapGestureRecognizer 对象在视图中检测到单指点击时，它会调用视图的视图控制器的操作方法，你可以使用该方法提供响应。

图 3 手势识别器通知其目标

![](https://docs-assets.developer.apple.com/published/7c21d852b9/0c8c5e29-c846-4a16-988b-3d809eafbb6b.png)

手势识别器有两种类型：离散和连续。在识别手势后，离散手势识别器会准确调用你的操作方法一次。在满足其初始识别标准后，连续手势识别器会多次调用你的操作方法，并在手势事件中的信息发生变化时通知你。例如，每次触摸位置更改时，UIPanGestureRecognizer 对象都会调用你的操作方法。

### 配置手势识别器

要配置手势识别器：

1. 在故事板中，将手势识别器拖到视图上。

2. 实现在识别手势时要调用的动作方法; 见清单1。

3. 将你的操作方法连接到手势识别器。

你可以在 Interface Builder 中通过右键单击手势识别器并将其“已发送操作”选择器连接到界面中的相应对象来创建此连接。你还可以使用手势识别器的 addTarget(_:action:) 方法以编程方式配置操作方法。

清单 1 显示了手势识别器的操作方法的通用格式。如果你愿意，可以更改参数类型以匹配特定的手势识别器子类。

清单 1 手势识别器动作方法

```swift
@IBAction func myActionMethod(_ sender: UIGestureRecognizer)
```

### 响应手势

与手势识别器关联的操作方法提供应用对该手势的响应。对于离散手势，你的操作方法类似于按钮的操作方法。调用操作方法后，你将执行适合该手势的任何任务。对于连续手势，你的动作方法可以响应手势的识别，但它也可以在识别手势之前跟踪事件。跟踪事件可让你创建更具互动性的体验。例如，你可以使用 UIPanGestureRecognizer 对象中的更新来重新定位应用中的内容。

手势识别器的状态属性传达对象的当前识别状态。对于连续手势，手势识别器将此属性的值从 UIGestureRecognizer.State.began 更新为 UIGestureRecognizer.State.changed 为 UIGestureRecognizer.State.ended，或更新为 UIGestureRecognizer.State.cancelled。你的操作方法使用此属性来确定适当的操作过程。例如，你可以使用开始和更改状态对内容进行临时更改，使用已结束状态使这些更改成为永久更改，并使用已取消状态来放弃更改。在执行操作之前，请始终检查手势识别器的 state 属性的值。

有关如何处理特定类型手势的示例，请参阅以下信息：

- [Handling Tap Gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_tap_gestures)
- [Handling Long-Press Gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_long-press_gestures)
- [Handling Pan Gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_pan_gestures)
- [Handling Swipe Gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_swipe_gestures)
- [Handling Pinch Gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_pinch_gestures)
- [Handling Rotation Gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_rotation_gestures)

## 协调多个手势识别器

手势识别器分别跟踪传入的触摸事件，但 UIKit 通常允许在单个视图上一次仅识别一个手势。通常最好只识别一个手势，因为它可以防止用户输入一次触发多个动作。但是，此默认行为可能会引入意外的副作用。例如，在包含平移和滑动手势识别器的视图中，永远不会识别滑动。因为平移手势识别器是连续的，所以它总是在滑动手势识别器之前识别其手势，该手势识别器是离散的。

为了防止默认识别行为的意外副作用，你可以告诉 UIKit 使用 delegate 对象以特定顺序识别手势。UIKit 使用你的 delegate 对象的方法来确定手势识别器是否必须在其他手势识别器之前或之后。例如，你的 delegate 可以告诉 UIKit，在允许平移手势识别器执行操作之前，滑动手势识别器必须失败。你的 delegate 也可以告诉 UIKit 可以同时识别两个手势。

## 实现自定义手势识别器

当内置的 UIKit 手势识别器不提供你想要的行为时，你可以定义自定义手势识别器。UIKit 定义了高度可配置的手势识别器，用于处理 taps, long presses, pans, swipes, rotations, 和 pinches 的触摸序列。对于其他触摸序列，或处理涉及按钮按下的手势，你可以定义自定义手势识别器。

你还可以使用自定义手势识别器来简化应用中的事件处理代码。例如，Leveraging Touch Input for Drawing Apps 示例使用手势识别器捕获输入并将其显示在屏幕上，如图 4 所示。

图 4 触摸自定义手势识别器捕获的输入

![](https://docs-assets.developer.apple.com/published/7c21d852b9/1dac7b87-cd89-4bb7-aa19-7110bb17987e.png)

要定义自定义手势识别器，请继承 UIGestureRecognizer（或其子类之一）。在源文件的顶部，导入 UIGestureRecognizerSubclass.h 头文件（对于 Objective-C）或 UIKit.UIGestureRecognizerSubclass 模块（对于 Swift），如清单 2 所示。此头文件定义必须覆盖的方法和属性实现自定义手势识别器。

清单 2 导入 UIGestureRecognizerSubclass 行为

```swift
import UIKit
import UIKit.UIGestureRecognizerSubclass
```

在自定义子类中，实现处理事件所需的任何方法。例如，如果你的手势由触摸事件组成，请实现 touchesBegan(_:with:), touchesMoved(_:with:), touchesEnded(_:with:), 和 touchesCancelled(_:with:) 方法。使用传入事件更新手势识别器的状态属性。UIKit 使用手势识别器状态来协调与界面中其他对象的交互。