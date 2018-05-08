# 使用 Bézier 路径绘制形状

在 iOS 3.2及更高版本中，你可以使用 [UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 类来创建基于矢量的路径。 UIBezierPath 类是 Core Graphics 框架中与路径相关的特性的 Objective-C 包装器。 你可以使用此类来定义简单的形状，如椭圆和矩形，以及包含多个直线和曲线线段的复杂形状。

你可以使用路径对象在应用的用户界面中绘制形状。 你可以绘制路径的轮廓，填充其包围的空间，或两者。 你还可以使用路径为当前图形上下文定义剪切区域，然后可以使用该区域修改该上下文中的后续绘制操作。

## Bézier 路径基础

[UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 对象是 [CGPathRef](https://developer.apple.com/documentation/coregraphics/cgpath) 数据类型的包装器。 路径是使用线段和曲线段构建的基于矢量的形状。 你可以使用线段来创建矩形和多边形，并且可以使用曲线段来创建弧线，圆形和复杂的曲线形状。 每个段由一个或多个点（在当前坐标系中）和一个绘图命令组成，这些命令定义了这些点的解释方式。

每组连接的线段和曲线段形成了所谓的子路径。 子路径中一条线或曲线段的末端定义下一条线的开始。 单个 `UIBezierPath` 对象可能包含一个或多个定义总体路径的子路径，它们之间用 [moveToPoint:](https://developer.apple.com/documentation/uikit/uibezierpath/1624343-move) 命令分隔，这些命令有效地提升绘图笔并将其移动到新的位置。

构建和使用路径对象的过程是分开的。 构建路径是第一个过程，涉及以下步骤：

1、创建路径对象。

2、设置 `UIBezierPath` 对象的任何相关图形属性，例如描边路径的 [lineWidth](https://developer.apple.com/documentation/uikit/uibezierpath/1624349-linewidth) 或 [lineJoinStyle](https://developer.apple.com/documentation/uikit/uibezierpath/1624378-linejoinstyle) 属性或填充路径的 [usesEvenOddFillRule](https://developer.apple.com/documentation/uikit/uibezierpath/1624360-usesevenoddfillrule) 属性。 这些图形属性适用于整个路径。

3、使用 [moveToPoint:](https://developer.apple.com/documentation/uikit/uibezierpath/1624343-move) 方法设置初始片段的起始点。

4、添加直线段和曲线段以定义子路径。

5、或者，通过调用 [closePath](https://developer.apple.com/documentation/uikit/uibezierpath/1624338-close) 关闭子路径，该路径从最后一个段的末尾到第一个段的开头绘制一条直线段。

6、或者，重复步骤3，4和5以定义其他子路径。

当建立你的路径时，你应该排列你的路径相对于原点（0，0）的点。这样做可以让后面的路径更容易移动。在绘制过程中，路径中的点将按原样应用于当前图形上下文的坐标系。如果你的路径相对于原点定向，则重新定位所需的所有操作都是将转换因子的仿射变换应用于当前图形上下文。修改图形上下文（与路径对象本身相反）的优点是可以通过保存和恢复图形状态轻松地撤销转换。

要绘制路径对象，请使用 [stroke](https://developer.apple.com/documentation/uikit/uibezierpath/1624365-stroke) 和 [fill](https://developer.apple.com/documentation/uikit/uibezierpath/1624371-fill) 方法。这些方法在当前图形上下文中呈现路径的线段和曲线段。渲染过程包括使用路径对象的属性对线段和曲线段进行栅格化。光栅化过程不会修改路径对象本身。因此，你可以在当前上下文或其他上下文中多次渲染相同的路径对象。

## 将线条和多边形添加到你的路径

线条和多边形是使用 [moveToPoint:](https://developer.apple.com/documentation/uikit/uibezierpath/1624343-move) 和 [addLineToPoint:](https://developer.apple.com/documentation/uikit/uibezierpath/1624354-addline) 方法逐点构建的简单形状。 `moveToPoint:` 方法设置要创建的形状的起点。 从这一点开始，你可以使用 `addLineToPoint:` 方法创建形状的线条。 你可以连续创建线条，每条线都在前一个点和你指定的新点之间形成。

清单2-1显示了使用单独线段创建五边形形状所需的代码。 （图2-1显示了使用适当的笔触和填充颜色设置绘制此形状的结果，如[渲染 Bézier 路径对象的内容](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/BezierPaths/BezierPaths.html#//apple_ref/doc/uid/TP40010156-CH11-SW13)所述。）此代码设置形状的初始点，然后添加四个连接的线段。 通过调用 [closePath](https://developer.apple.com/documentation/uikit/uibezierpath/1624338-close) 方法添加第五个段，该方法将最后一个点（0，40）与第一个点（100，0）连接起来。

清单 2-1 创建五角形形状
```
UIBezierPath *aPath = [UIBezierPath bezierPath];
 
// Set the starting point of the shape.
[aPath moveToPoint:CGPointMake(100.0, 0.0)];
 
// Draw the lines.
[aPath addLineToPoint:CGPointMake(200.0, 40.0)];
[aPath addLineToPoint:CGPointMake(160, 140)];
[aPath addLineToPoint:CGPointMake(40.0, 140)];
[aPath addLineToPoint:CGPointMake(0.0, 40.0)];
[aPath closePath];
```

图 2-1 使用 `UIBezierPath` 类的方法绘制的形状

![](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/bezier_pentagon_2x.png)

使用 `closePath` 方法不仅会结束描述形状的子路径，还会在第一个点和最后一个点之间绘制一条线段。 这是一个方便的方法来完成一个多边形，而不必画出最后一行。

## 添加弧到你的路径

[UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 类为使用弧段初始化新路径对象提供支持。 [bezierPathWithArcCenter:radius:startAngle:endAngle:clockwise:](https://developer.apple.com/documentation/uikit/uibezierpath/1624358-init) 方法的参数定义了包含所需圆弧的圆以及圆弧本身的起点和终点。 图2-2显示了创建弧的组件，包括用于定义弧的圆和用于指定它的角度测量。 在这种情况下，弧线按顺时针方向创建。 （逆时针方向绘制圆弧会代替圆圈的虚线部分。）创建此圆弧的代码如[代码清单2-2](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/BezierPaths/BezierPaths.html#//apple_ref/doc/uid/TP40010156-CH11-SW7)所示。

图 2-2 默认坐标系中的弧

![](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/arc_layout_2x.png)

清单 2-2 创建一个新的弧路径
```
// pi is approximately equal to 3.14159265359.
#define   DEGREES_TO_RADIANS(degrees)  ((pi * degrees)/ 180)

- (UIBezierPath *)createArcPath
{
   UIBezierPath *aPath = [UIBezierPath bezierPathWithArcCenter:CGPointMake(150, 150)
                           radius:75
                           startAngle:0
                           endAngle:DEGREES_TO_RADIANS(135)
                           clockwise:YES];
   return aPath;
}
```

如果要将弧段合并到路径的中间，则必须直接修改路径对象的 [CGPathRef](https://developer.apple.com/documentation/coregraphics/cgpath) 数据类型。 有关使用 Core Graphics 函数修改路径的更多信息，请参阅[使用核心图形函数修改路径](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/BezierPaths/BezierPaths.html#//apple_ref/doc/uid/TP40010156-CH11-SW10)。

## 添加曲线到你的路径

[UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 类提供了将三次和二次 Bézier 曲线添加到路径的支持。 曲线段从当前点开始并在你指定的点结束。 曲线的形状是使用起点和终点以及一个或多个控制点之间的切线定义的。 图2-3显示了两种类型曲线的近似值以及控制点与曲线形状之间的关系。 每个部分的确切曲率涉及所有点之间的复杂数学关系，并在网上和[维基百科](http://en.wikipedia.org/wiki/Bezier_curve)上有详细记录。

图 2-3 路径中的曲线段

![](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/curve_segments_2x.png)

要将曲线添加到路径中，请使用以下方法：

- Cubic curve：[addCurveToPoint:controlPoint1:controlPoint2:](https://developer.apple.com/documentation/uikit/uibezierpath/1624357-addcurvetopoint)
- Quadratic curve:[addQuadCurveToPoint:controlPoint:](https://developer.apple.com/documentation/uikit/uibezierpath/1624351-addquadcurvetopoint)

由于曲线依赖于路径的当前点，因此必须在调用上述任一方法之前设置当前点。 完成曲线后，当前点将更新为你指定的新终点。

## 创建椭圆和矩形路径

椭圆和矩形是使用曲线和线段组合构建的常见路径类型。 [UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 类包含 [bezierPathWithRect:](https://developer.apple.com/documentation/uikit/uibezierpath/1624359-init) 和 [bezierPathWithOvalInRect:](https://developer.apple.com/documentation/uikit/uibezierpath/1624379-bezierpathwithovalinrect) 便捷方法，用于创建具有椭圆或矩形形状的路径。 这两种方法都会创建一个新的路径对象并使用指定的形状对其进行初始化。 你可以立即使用返回的路径对象，或根据需要添加更多形状。

如果要将矩形添加到现有路径对象，则必须像使用其他多边形一样，使用 [moveToPoint:](https://developer.apple.com/documentation/uikit/uibezierpath/1624343-move)，[addLineToPoint:](https://developer.apple.com/documentation/uikit/uibezierpath/1624354-addline) 和 [closePath](https://developer.apple.com/documentation/uikit/uibezierpath/1624338-close) 方法来完成此操作。 对矩形的最后一边使用 `closePath` 方法是添加路径的最后一行并标记矩形子路径的结尾的一种便捷方式。

如果你想添加一个椭圆到现有的路径，最简单的方法是使用 Core Graphics。 尽管可以使用 [addQuadCurveToPoint:controlPoint:](https://developer.apple.com/documentation/uikit/uibezierpath/1624351-addquadcurvetopoint) 近似椭圆曲面，但 [CGPathAddEllipseInRect](https://developer.apple.com/documentation/coregraphics/1411222-cgpathaddellipseinrect) 函数使用起来更加简单并且更加精确。 有关更多信息，请参阅[使用 Core Graphics 函数修改路径](https://developer.apple.com/library/content/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/BezierPaths/BezierPaths.html#//apple_ref/doc/uid/TP40010156-CH11-SW10)。

## 使用 Core Graphics 函数修改路径

[UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 类实际上只是 [CGPathRef](https://developer.apple.com/documentation/coregraphics/cgpath) 数据类型的封装器，以及与该路径关联的绘图属性。 尽管通常使用 UIBezierPath 类的方法添加线段和曲线段，但该类还会公开 [CGPath](https://developer.apple.com/documentation/uikit/uibezierpath/1624342-cgpath) 属性，你可以使用该属性直接修改底层路径数据类型。 如果你希望使用 Core Graphics 框架的功能构建路径，则可以使用此属性。

有两种方法可以修改与 `UIBezierPath` 对象关联的路径。 你可以使用 Core Graphics 函数完全修改路径，也可以使用 Core Graphics 函数和 `UIBezierPath` 方法的混合。 在某些方面，使用 Core Graphics 调用完全修改路径更容易。 你创建一个可变的 `CGPathRef` 数据类型并调用你需要修改其路径信息的任何函数。 完成后，将路径对象分配给相应的 `UIBezierPath` 对象，如清单2-3所示。

清单 2-3 将一个新的 `CGPathRef` 分配给一个 `UIBezierPath` 对象
```
// Create the path data.
CGMutablePathRef cgPath = CGPathCreateMutable();
CGPathAddEllipseInRect(cgPath, NULL, CGRectMake(0, 0, 300, 300));
CGPathAddEllipseInRect(cgPath, NULL, CGRectMake(50, 50, 200, 200));
 
// Now create the UIBezierPath object.
UIBezierPath *aPath = [UIBezierPath bezierPath];
aPath.CGPath = cgPath;
aPath.usesEvenOddFillRule = YES;
 
// After assigning it to the UIBezierPath object, you can release
// your CGPathRef data type safely.
CGPathRelease(cgPath);
```

如果你选择使用 Core Graphics 函数和 UIBezierPath 方法的混合，则必须小心地在两者之间来回移动路径信息。 由于 UIBezierPath 对象拥有其基础 CGPathRef 数据类型，因此不能简单地检索该类型并直接对其进行修改。 相反，你必须制作可变副本，修改副本，然后将副本分配回 CGPath 属性，如清单2-4所示。

清单 2-4 混合 Core Graphics 和 UIBezierPath 调用
```
UIBezierPath *aPath = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(0, 0, 300, 300)];
 
// Get the CGPathRef and create a mutable version.
CGPathRef cgPath = aPath.CGPath;
CGMutablePathRef  mutablePath = CGPathCreateMutableCopy(cgPath);
 
// Modify the path and assign it back to the UIBezierPath object.
CGPathAddEllipseInRect(mutablePath, NULL, CGRectMake(50, 50, 200, 200));
aPath.CGPath = mutablePath;
 
// Release both the mutable copy of the path.
CGPathRelease(mutablePath);
```

## 呈现 Bézier 路径对象的内容

创建 [UIBezierPath](https://developer.apple.com/documentation/uikit/uibezierpath) 对象后，可以使用其中 [stroke](https://developer.apple.com/documentation/uikit/uibezierpath/1624365-stroke) 和 [fill](https://developer.apple.com/documentation/uikit/uibezierpath/1624371-fill) 方法在当前图形上下文中渲染它。 但是，在调用这些方法之前，通常还需要执行其他一些任务才能确保正确绘制路径：

- 使用 [UIColor](https://developer.apple.com/documentation/uikit/uicolor) 类的方法设置所需的 stroke 和 fill 颜色。

- 将形状放置在目标视图中所需的位置。

  如果你创建了相对于点（0，0）的路径，则可以将适当的仿射变换应用于当前绘图上下文。 例如，要从点（10,10）开始绘制形状，你应该调用 CGContextTranslateCTM 函数，并为水平和垂直平移值指定10。 调整图形上下文（而不是路径对象中的点）是首选，因为你可以通过保存和恢复以前的图形状态更容易地撤消更改。

- 更新路径对象的绘图属性。 绘制路径时，UIBezierPath 实例的绘图属性会覆盖与图形上下文相关联的值。

清单2-5显示了一个在自定义视图中绘制椭圆的 [drawRect:](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect) 方法的示例实现。 椭圆的边界矩形的左上角位于视图坐标系中的点（50,50）处。 由于填充操作直至路径边界绘制，此方法在抚摸它之前填充路径。 这可以防止填充颜色模糊描边线的一半。

清单 2-5 在视图中绘制路径
```
- (void)drawRect:(CGRect)rect
{
    // Create an oval shape to draw.
    UIBezierPath *aPath = [UIBezierPath bezierPathWithOvalInRect:
                                CGRectMake(0, 0, 200, 100)];
 
    // Set the render colors.
    [[UIColor blackColor] setStroke];
    [[UIColor redColor] setFill];
 
    CGContextRef aRef = UIGraphicsGetCurrentContext();
 
    // If you have content to draw after the shape,
    // save the current state before changing the transform.
    //CGContextSaveGState(aRef);
 
    // Adjust the view's origin temporarily. The oval is
    // now drawn relative to the new origin point.
    CGContextTranslateCTM(aRef, 50, 50);
 
    // Adjust the drawing options as needed.
    aPath.lineWidth = 5;
 
    // Fill the path before stroking it so that the fill
    // color does not obscure the stroked line.
    [aPath fill];
    [aPath stroke];
 
    // Restore the graphics state before drawing any other content.
    //CGContextRestoreGState(aRef);
}
```

## 在路径上进行点击检测

要确定路径的填充部分是否发生触摸事件，可以使用 UIBezierPath 的 [containsPoint:](https://developer.apple.com/documentation/uikit/uibezierpath/1624345-containspoint) 方法。此方法针对路径对象中的所有已关闭的子路径测试指​​定的点，并且如果位于任何这些子路径之上或之内，则返回 YES。

重要提示：`containsPoint:` 方法和 Core Graphics 命中测试功能仅在封闭路径上运行。这些方法总是在打开的子路径上返回 NO。如果要在打开的子路径上执行命中检测，则必须在测试点之前创建路径对象的副本并关闭打开的子路径。

如果您想对路径的描边部分（而不是填充区域）执行命中测试，则必须使用 Core Graphics。 CGContextPathContainsPoint 函数允许你在当前分配给图形上下文的路径的填充或笔划部分上测试点。 清单2-6显示了一个方法，用于测试查看指定点是否与指定路径相交。 inFill 参数让调用者指定是否应该对路径的填充或描边部分进行测试。 调用者传入的路径必须包含一个或多个关闭的子路径才能使命中检测成功。

清单 2-6 针对路径对象的测试点

```
- (BOOL)containsPoint:(CGPoint)point onPath:(UIBezierPath *)path inFillArea:(BOOL)inFill
{
   CGContextRef context = UIGraphicsGetCurrentContext();
   CGPathRef cgPath = path.CGPath;
   BOOL    isHit = NO;
 
   // Determine the drawing mode to use. Default to
   // detecting hits on the stroked portion of the path.
   CGPathDrawingMode mode = kCGPathStroke;
   if (inFill)
   {
      // Look for hits in the fill area of the path instead.
      if (path.usesEvenOddFillRule)
         mode = kCGPathEOFill;
      else
         mode = kCGPathFill;
   }
 
   // Save the graphics state so that the path can be
   // removed later.
   CGContextSaveGState(context);
   CGContextAddPath(context, cgPath);
 
   // Do the hit detection.
   isHit = CGContextPathContainsPoint(context, point, mode);
 
   CGContextRestoreGState(context);
 
   return isHit;
}
```

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~