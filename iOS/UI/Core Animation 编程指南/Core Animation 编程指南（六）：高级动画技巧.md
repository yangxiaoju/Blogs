# 高级动画技巧

有很多方法可以配置你的基于属性或关键帧的动画，为你做更多的事情。 需要一起或按顺序执行多个动画的应用可以使用更高级的行为来同步这些动画的时间或将它们链接在一起。 你还可以使用其他类型的动画对象来创建视觉转场和其他有趣的动画效果。

## 过渡动画支持对图层可见性的更改

顾名思义，过渡动画对象为图层创建动画视觉过渡。过渡对象最常见的用途是以协调的方式为一层的外观和另一层的外观消失。与基于属性的动画不同，在动画中，动画会更改图层的一个属性，而过渡动画会操纵图层的缓存图像来创建视觉效果，而单独更改属性将很难或无法完成。标准类型的转换可让你执行显示，推送，移动或交叉淡入淡出动画。在 OS X 上，你还可以使用 Core Image 滤镜来创建使用其他类型效果的过渡，例如你设计的抹布，页面卷曲，涟漪或自定义效果。

要执行过渡动画，你需要创建一个 [CATransition](https://developer.apple.com/documentation/quartzcore/catransition) 对象并将其添加到转换中涉及的图层中。你可以使用过渡对象来指定要执行的过渡类型以及过渡动画的开始点和结束点。你也不需要使用整个过渡动画。过渡对象可让你指定动画时使用的开始和结束进度值。这些值可让你在中点处开始或结束动画。

清单5-1显示了用于在两个视图之间创建动画推送转换的代码。在该示例中，myView1 和 myView2 位于同一父视图中的相同位置，但只有 myView1 当前可见。 push 过渡会导致 myView1 向左滑出并淡入，直到 myView2 从右侧滑入并变为可见时隐藏。更新两个视图的隐藏属性可确保在动画结束时两个视图的可见性都是正确的。

清单 5-1 动画 iOS 中两个视图之间的转换
```
CATransition* transition = [CATransition animation];
transition.startProgress = 0;
transition.endProgress = 1.0;
transition.type = kCATransitionPush;
transition.subtype = kCATransitionFromRight;
transition.duration = 1.0;
 
// Add the transition animation to both layers
[myView1.layer addAnimation:transition forKey:@"transition"];
[myView2.layer addAnimation:transition forKey:@"transition"];
 
// Finally, change the visibility of the layers.
myView1.hidden = YES;
myView2.hidden = NO;
```

当两个图层涉及相同的转换时，可以为两者使用相同的转换对象。 使用相同的转换对象还可以简化必须编写的代码。 但是，你可以使用不同的转换对象，并且如果每个图层的转换参数不同，则肯定会这样做。

清单5-2显示了如何使用 Core Image 滤镜在 OS X 上实现过渡效果。使用所需参数配置过滤器后，将其分配给过渡对象的 [filter](https://developer.apple.com/documentation/quartzcore/catransition/1412506-filter) 属性。 之后，应用动画的过程与其他类型的动画对象相同。

清单 5-2 使用 Core Image 滤镜为 OS X 上的转换设置动画效果
```
// Create the Core Image filter, setting several key parameters.
CIFilter* aFilter = [CIFilter filterWithName:@"CIBarsSwipeTransition"];
[aFilter setValue:[NSNumber numberWithFloat:3.14] forKey:@"inputAngle"];
[aFilter setValue:[NSNumber numberWithFloat:30.0] forKey:@"inputWidth"];
[aFilter setValue:[NSNumber numberWithFloat:10.0] forKey:@"inputBarOffset"];
 
// Create the transition object
CATransition* transition = [CATransition animation];
transition.startProgress = 0;
transition.endProgress = 1.0;
transition.filter = aFilter;
transition.duration = 1.0;
 
[self.imageView2 setHidden:NO];
[self.imageView.layer addAnimation:transition forKey:@"transition"];
[self.imageView2.layer addAnimation:transition forKey:@"transition"];
[self.imageView setHidden:YES];
```

> 注意：在动画中使用 Core Image 滤镜时，最棘手的部分是配置滤镜。 例如，通过条形滑动转换，指定太高或太低的输入角度可能会使其看起来好像没有转换发生。 如果你没有看到你期望的动画，请尝试将你的 filter 参数调整为不同的值，以查看是否更改了结果。

## 自定义动画的时间

定时是动画的重要组成部分，通过 Core Animation，你可以通过 [CAMediaTiming](https://developer.apple.com/documentation/quartzcore/camediatiming) 协议的方法和属性为动画指定精确的定时信息。两个核心动画类采用这个协议。 CAAnimation 类采用它，以便你可以在动画对象中指定时间信息。 CALayer 也采用它，以便你可以为隐式动画配置一些与时间相关的功能，但包装这些动画的隐式事务对象通常会提供优先的默认定时信息。

在考虑时间和动画时，了解层对象如何随时间工作很重要。每个图层都有自己的本地时间，用于管理动画时间。通常，两个不同图层的本地时间足够接近，你可以为每个图层指定相同的时间值，并且用户可能不会注意到任何内容。但是，图层的本地时间可以由其父图层或其自己的时间参数修改。例如，更改图层的 [speed](https://developer.apple.com/documentation/quartzcore/camediatiming/1427647-speed) 属性会导致该图层（及其子图层）上动画的持续时间按比例变化。

为了帮助你确保给定图层的时间值合适，CALayer 类定义了 [convertTime:fromLayer:](https://developer.apple.com/documentation/quartzcore/calayer/1410821-converttime) 和 [convertTime:toLayer:](https://developer.apple.com/documentation/quartzcore/calayer/1410823-converttime) 方法。你可以使用这些方法将固定时间值转换为图层的本地时间，或将时间值从一层转换为另一层。这些方法考虑了可能影响图层本地时间的媒体定时属性，并返回可用于其他图层的值。清单5-3显示了一个例子，你应该经常使用它来获取图层的当前本地时间。 [CACurrentMediaTime](https://developer.apple.com/documentation/quartzcore/1395996-cacurrentmediatime) 函数是一个方便的函数，它返回计算机当前的时钟时间，该方法将其转换为图层的本地时间。

清单 5-3 获取图层当前的本地时间
```
CFTimeInterval localLayerTime = [myLayer convertTime:CACurrentMediaTime() fromLayer:nil];
```

一旦在图层的本地时间中有时间值，就可以使用该值更新动画对象或图层的与时间相关的属性。通过这些计时属性，你可以实现一些有趣的动画行为，包括：

- 使用 [beginTime](https://developer.apple.com/documentation/quartzcore/camediatiming/1427654-begintime) 属性设置动画的开始时间。通常，动画在下一个更新周期开始。你可以使用 beginTime 参数将动画开始时间延迟几秒钟。将两个动画链接在一起的方法是将一个动画的开始时间设置为与另一个动画的结束时间相匹配。

    如果延迟动画的开始，则可能还需要将 [fillMode](https://developer.apple.com/documentation/quartzcore/camediatiming/1427656-fillmode) 属性设置为 [kCAFillModeBackwards](https://developer.apple.com/documentation/quartzcore/kcafillmodebackwards)。即使图层树中的图层对象包含不同的值，该填充模式也会使图层显示动画的起始值。如果没有这个填充模式，你会在动画开始执行之前看到跳转到最终值。其他填充模式也可用。
- [autoreverses](https://developer.apple.com/documentation/quartzcore/camediatiming/1427645-autoreverses) 属性导致动画在指定的持续时间内执行，然后返回到动画的起始值。你可以将此属性与 [repeatCount](https://developer.apple.com/documentation/quartzcore/camediatiming/1427666-repeatcount) 属性结合在起始值和结束值之间来回移动。将重复次数设置为整​​数（例如1.0）以进行自动对换动画会导致动画停止其初始值。添加额外的半步（例如重复计数为1.5）会导致动画停止在其最终值上。
- 将 [timeOffset](https://developer.apple.com/documentation/quartzcore/camediatiming/1427650-timeoffset) 属性与组动画一起使用可以在稍后时间启动一些动画。

## 暂停和恢复动画

要暂停动画，可以利用图层采用 CAMediaTiming 协议并将图层动画的速度设置为0.0的事实。 将速度设置为零将暂停动画，直到你将该值更改回非零值。 清单5-4显示了一个简单的例子，介绍如何在稍后暂停和恢复动画。

清单 5-4 暂停和恢复一个图层的动画
```
-(void)pauseLayer:(CALayer*)layer {
   CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];
   layer.speed = 0.0;
   layer.timeOffset = pausedTime;
}
 
-(void)resumeLayer:(CALayer*)layer {
   CFTimeInterval pausedTime = [layer timeOffset];
   layer.speed = 1.0;
   layer.timeOffset = 0.0;
   layer.beginTime = 0.0;
   CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
   layer.beginTime = timeSincePause;
}
```

## 显式事务让您更改动画参数

你对图层所做的每一项更改都必须是交易的一部分。 [CATransaction](https://developer.apple.com/documentation/quartzcore/catransaction) 类在适当的时候管理动画的创建和分组以及它们的执行。 在大多数情况下，你不需要创建自己的交易。 无论何时将显式或隐式动画添加到其中一个图层，Core Animation 都会自动创建隐式事务。 但是，你也可以创建显式事务来更精确地管理这些动画。

你可以使用 CATransaction 类的方法创建和管理事务。 要开始（并隐式创建）一个新事务，请调用 [begin](https://developer.apple.com/documentation/quartzcore/catransaction/1448282-begin) 类方法; 要结束该事务，请调用 [commmit](https://developer.apple.com/documentation/quartzcore/catransaction/1448255-commit) 类方法。 在这些调用之间是你想成为交易一部分的变化。 例如，要更改图层的两个属性，可以使用清单5-5中的代码。

清单 5-5 创建一个显式的事务
```
[CATransaction begin];
theLayer.zPosition=200.0;
theLayer.opacity=0.0;
[CATransaction commit];
```

使用交易的主要原因之一是，在明确交易的范围内，你可以更改持续时间，计时功能和其他参数。 你还可以为整个事务分配一个完成块，以便在动画组结束时通知你的应用程序。 更改动画参数需要使用 [setValue:forKey:](https://developer.apple.com/documentation/quartzcore/catransaction/1448278-setvalue) 方法修改事务字典中的相应键。 例如，要将默认持续时间更改为10秒，你可以更改 [kCATransactionAnimationDuration](https://developer.apple.com/documentation/quartzcore/kcatransactionanimationduration) 键，如清单5-6所示。

清单 5-6 更改动画的默认持续时间
```
[CATransaction begin];
[CATransaction setValue:[NSNumber numberWithFloat:10.0f]
                 forKey:kCATransactionAnimationDuration];
// Perform the animations
[CATransaction commit];
```

你可以在要为不同的动画集提供不同默认值的情况下嵌套事务。 要将一个事务嵌套到另一个事务中，只需再次调用 begin 类方法即可。 每次开始调用都必须通过对 commit 方法的相应调用进行匹配。 只有在你为最外层事务提交更改后，Core Animation 才会开始关联的动画。

清单5-7显示了嵌套在另一个事务中的一个事务的例子。 在这个例子中，内部事务改变了与外部事务相同的动画参数，但使用了不同的值。

清单 5-7 嵌套显式事务
```
[CATransaction begin]; // Outer transaction
 
// Change the animation duration to two seconds
[CATransaction setValue:[NSNumber numberWithFloat:2.0f]
                forKey:kCATransactionAnimationDuration];
// Move the layer to a new position
theLayer.position = CGPointMake(0.0,0.0);
 
[CATransaction begin]; // Inner transaction
// Change the animation duration to five seconds
[CATransaction setValue:[NSNumber numberWithFloat:5.0f]
                 forKey:kCATransactionAnimationDuration];
 
// Change the zPosition and opacity
theLayer.zPosition=200.0;
theLayer.opacity=0.0;
 
[CATransaction commit]; // Inner transaction
 
[CATransaction commit]; // Outer transaction
```

## 为你的动画添加透视图

应用程序可以在三个空间维度中操作图层，但为了简单起见，Core Animation 使用平行投影显示图层，这基本上将场景平整为二维平面。这种默认行为会导致具有不同 zPosition 值的相同大小的图层显示为相同的大小，即使它们在 z 轴上相距很远。你通常会在三维空间观看这样的场景的视角已经消失。但是，你可以通过修改图层的变换矩阵来包含透视信息来更改该行为。

修改场景的透视图时，需要修改包含正在查看的图层的超层的 sublayerTransform 矩阵。通过将相同的透视信息应用于所有子图层，修改超图层可简化必须编写的代码。它还确保将透视图正确应用于在不同平面上彼此重叠的同级子图层。

清单5-8给出了为父层创建简单透视变换的方法。在这种情况下，自定义 eyePosition 变量指定沿 z 轴的相对距离，从中查看图层。通常你可以为 eyePosition 指定一个正值，以预期的方式保持图层的方向。较大的值会导致更平坦的场景，而较小的值会导致层之间更加显着的视觉差异。

清单 5-8 将透视变换添加到父图层
```
CATransform3D perspective = CATransform3DIdentity;
perspective.m34 = -1.0/eyePosition;
 
// Apply the transform to a parent layer.
myParentLayer.sublayerTransform = perspective;
```

通过配置父层，你可以更改任何子图层的 zPosition 属性，并根据它们与眼睛位置的相对距离观察其大小的变化。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~