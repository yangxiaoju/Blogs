# 使用 Bézier 路径绘制图形

在 iOS 3.2 及更高版本中，你可以使用 UIBezierPath 类创建基于矢量的路径。UIBezierPath 类是 Core Graphics 框架中与路径相关的功能的 Objective-C 包装器。你可以使用此类定义简单图形，例如椭圆和矩形，以及包含多个直线和曲线线段的复杂形状。

你可以使用路径对象在应用程序的用户界面中绘制形状。你可以绘制路径的轮廓，填充它所包含的空间，或两者兼有。你还可以使用路径为当前图形上下文定义剪切区域，然后可以使用该剪辑区域修改该上下文中的后续绘制操作。

## Bézier 路径基础知识

UIBezierPath 对象是 CGPathRef 数据类型的包装器。路径是使用直线和曲线段构建的基于矢量的形状。你可以使用线段创建矩形和多边形，并且可以使用曲线段创建圆弧，圆和复杂的曲线形状。每个段由一个或多个点（在当前坐标系中）和一个绘图命令组成，该命令定义如何解释这些点。

每组连接的线和曲线段形成所谓的子路径。子路径中一行或曲线段的末尾定义下一行的开头。单个 UIBezierPath 对象可以包含一个或多个定义整个路径的子路径，由 moveToPoint: 命令分隔，这些命令可以有效地提升绘图笔并将其移动到新位置。构建和使用路径对象的过程是分开的。构建路径是第一个过程，涉及以下步骤：

1、创建路径对象。
2、设置 UIBezierPath 对象的任何相关绘图属性，例如描边路径的 lineWidth 或 lineJoinStyle 属性或已填充路径的 usesEvenOddFillRule 属性。这些绘图属性适用于整个路径。
3、使用 moveToPoint: 方法设置初始段的起始点。
4、添加线和曲线段以定义子路径。
5、（可选）通过调用 closePath 关闭子路径，closePath 从最后一个段的末尾到第一个段的开头绘制一条直线段。
6、（可选）重复步骤3,4和5以定义其他子路径。

构建路径时，应该相对于原点（0,0）排列路径的点。这样做可以更轻松地在以后移动路径。在绘制过程中，路径的点将按原样应用于当前图形上下文的坐标系。如果你的路径相对于原点定向，则只需重新定位它就可以将具有平移因子的仿射变换应用于当前图形上下文。修改图形上下文（与路径对象本身相对）的优点是，你可以通过保存和恢复图形状态轻松撤消转换。

要绘制路径对象，请使用 stroke 和 fill 方法。这些方法在当前图形上下文中呈现路径的线段和曲线段。渲染过程涉及使用路径对象的属性栅格化线和曲线段。栅格化过程不会修改路径对象本身。因此，你可以在当前上下文或另一个上下文中多次呈现相同的路径对象。

## 在你的路径中添加线条和多边形

线条和多边形是使用 moveToPoint: 和 addLineToPoint: 方法逐点构建的简单形状。moveToPoint: 方法设置要创建的形状的起点。从那时起，你可以使用 addLineToPoint: 方法创建形状的线条。你可以连续创建线条，每条线条都在前一个点和你指定的新点之间形成。

清单 2-1 显示了使用单独的线段创建五边形形状所需的代码。（图2-1显示了使用适当的笔触和填充颜色设置绘制此形状的结果，如 Rendering the Contents of a Bézier Path Object 中所述。）此代码设置形状的初始点，然后添加四个连接的线段。通过调用 closePath 方法添加第五个段，该方法将最后一个点（0,40）与第一个点（100,0）连接起来。

清单 2-1 创建五边形形状
```objc
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

图 2-1 使用 UIBezierPath 类的方法绘制的形状
![](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/bezier_pentagon_2x.png)

使用 closePath 方法不仅会结束描述形状的子路径，还会在第一个和最后一个点之间绘制一条线段。这是完成多边形而不必绘制最终线的便捷方法。

## 将弧添加到路径中

UIBezierPath 类支持使用弧段初始化新路径对象。bezierPathWithArcCenter:radius:startAngle:endAngle:clockwise: 方法的参数定义包含所需弧的圆以及弧本身的起点和终点。图2-2显示了创建弧的组件，包括定义弧的圆和用于指定弧的角度测量。在这种情况下，弧沿顺时针方向创建。（以逆时针方向绘制圆弧将绘制圆形的虚线部分。）创建此弧的代码如清单2-2所示。

图 2-2 默认坐标系中的圆弧
![](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/arc_layout_2x.png)

清单 2-2 创建一个新的弧形路径
```objc
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

如果要将弧段合并到路径的中间，则必须直接修改路径对象的 CGPathRef 数据类型。有关使用 Core Graphics 函数修改路径的更多信息，请参阅 Modifying the Path Using Core Graphics Functions。

## 将曲线添加到路径中

UIBezierPath 类支持将三次和二次 Bézier 曲线添加到路径。曲线段从当前点开始，到你指定的点结束。使用起点和终点之间的切线以及一个或多个控制点来定义曲线的形状。图2-3显示了两种曲线类型的近似值以及控制点与曲线形状之间的关系。每个段的精确曲率涉及所有点之间的复杂数学关系，并且在线和维基百科都有详细记录。

图 2-3 路径中的曲线段
![](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/Art/curve_segments_2x.png)

要将曲线添加到路径，请使用以下方法：

- 立方曲线：curve:addCurveToPoint:controlPoint1:controlPoint2:
- 二次曲线：addQuadCurveToPoint:controlPoint:

由于曲线依赖于路径的当前点，因此必须在调用上述任一方法之前设置当前点。完成曲线后，当前点将更新为你指定的新结束点。

## 创建椭圆和矩形路径

椭圆和矩形是使用曲线和线段组合构建的常见路径类型。UIBezierPath 类包含 bezierPathWithRect: 和 bezierPathWithOvalInRect: 方便方法，用于创建椭圆或矩形形状的路径。这两种方法都创建了一个新的路径对象，并使用指定的形状对其进行初始化。你可以立即使用返回的路径对象，也可以根据需要添加更多形状。

如果要将矩形添加到现有路径对象，则必须使用 moveToPoint:，addLineToPoint: 和 closePath 方法，就像对任何其他多边形一样。对矩形的最后一侧使用 closePath 方法是添加路径的最后一行并且还标记矩形子路径的末尾的便捷方式。

如果要在现有路径中添加椭圆，最简单的方法是使用 Core Graphics。虽然你可以使用 addQuadCurveToPoint:controlPoint: 来近似椭圆曲面，但 CGPathAddEllipseInRect 函数使用起来更简单，更准确。有关更多信息，请参阅 Modifying the Path Using Core Graphics Functions。

## 使用 Core Graphics 函数修改路径

UIBezierPath 类实际上只是 CGPathRef 数据类型的包装器以及与该路径关联的绘图属性。虽然你通常使用 UIBezierPath 类的方法添加线段和曲线段，但该类还公开了一个 CGPath 属性，你可以使用该属性直接修改基础路径数据类型。当你希望使用 Core Graphics 框架的功能构建路径时，可以使用此属性。

有两种方法可以修改与 UIBezierPath 对象关联的路径。你可以使用 Core Graphics 函数完全修改路径，也可以混合使用 Core Graphics 函数和 UIBezierPath 方法。在某些方面，使用 Core Graphics 调用完全修改路径更容易。你可以创建可变 CGPathRef 数据类型并调用修改其路径信息所需的任何函数。完成后，将路径对象分配给相应的 UIBezierPath 对象，如清单2-3所示。

清单 2-3 为 UIBezierPath 对象分配新的 CGPathRef
```objc
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

如果选择使用 Core Graphics 函数和 UIBezierPath 方法的混合，则必须在两者之间来回小心地移动路径信息。因为 UIBezierPath 对象拥有其基础 CGPathRef 数据类型，所以你不能简单地检索该类型并直接修改它。相反，你必须制作可变副本，修改副本，然后将副本分配回 CGPath 属性，如清单2-4所示。

清单 2-4 混合 Core Graphics 和 UIBezierPath 调用
```objc
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

## 渲染 Bézier 路径对象的内容

创建 UIBezierPath 对象后，可以使用其 stroke 和 fill 方法在当前图形上下文中渲染它。但是，在调用这些方法之前，通常还需要执行一些其他任务来确保正确绘制路径：

- 使用 UIColor 类的方法设置所需的 stroke 和 fill 颜色。
- 将形状放在目标视图中所需的位置。

  如果你创建了相对于点（0,0）的路径，则可以将适当的仿射变换应用于当前绘图上下文。例如，要从点（10,10）开始绘制形状，你将调用 CGContextTranslateCTM 函数并为水平和垂直平移值指定10。首选调整图形上下文（而不是路径对象中的点），因为你可以通过保存和恢复以前的图形状态来更轻松地撤消更改。
- 更新路径对象的绘图属性。在渲染路径时，UIBezierPath 实例的绘图属性会覆盖与图形上下文关联的值。

清单2-5显示了 drawRect: 方法的示例实现，该方法在自定义视图中绘制椭圆。椭圆的边界矩形的左上角位于视图坐标系中的点（50,50）处。因为 fill 操作直接绘制到路径边界，所以此方法在 stroke 之前填充路径。这可以防止填充颜色遮挡一半的描边线。

清单 2-5 在视图中绘制路径
```objc
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

## 在路径上执行命中检测

要确定是否在路径的填充部分上发生了触摸事件，可以使用 UIBezierPath 的 containsPoint: 方法。此方法针对路径对象中所有已关闭的子路径测试指定点，如果它位于任何这些子路径上或内部，则返回 YES。

> 重要说明：containsPoint: 方法和 Core Graphics 命中测试功能仅在封闭路径上运行。对于打开的子路径上的命中，这些方法始终返回 NO。如果要在打开的子路径上执行命中检测，则必须先创建路径对象的副本，然后在测试点之前关闭打开的子路径。

如果要对路径的描边部分（而不是填充区域）进行命中测试，则必须使用 Core Graphics。CGContextPathContainsPoint 函数允许你测试当前分配给图形上下文的路径的填充或描边部分上的点。清单 2-6 显示了一个方法，用于测试指定的点是否与指定的路径相交。inFill 参数允许调用者指定是否应针对路径的填充或描边部分测试该点。调用者传入的路径必须包含一个或多个已关闭的子路径，以使命中检测成功。

清单 2-6 针对路径对象的测试点
```objc
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