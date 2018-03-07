# Swift 中的 Switch 语句
Swift 语句会考虑一个值并将其与几种可能的匹配模式进行比较。然后根据成功匹配的第一个模式执行适当的代码块。Switch 语句为 if 语句提供了对多个潜在状态进行响应的替代方法。

以最简单的形式，switch 语句将一个值与一个或多个相同类型的值进行比较。
```
switch some value to consider {
case value 1:
    respond to value 1
case value 2,
     value 3:
    respond to value 2 or 3
default:
    otherwise, do something else
}
```
每个 switch 语句都由多个可能的情况组成，每个情况都以 case 关键字开始。除了与特定值进行比较外，swift 还为每种情况提供了几种方法来指定更复杂的匹配模式。这些选项在本章稍后介绍。

像 if 语句的主体一样，每个 case 都是一个独立的代码执行分支。Switch 语句确定应该选择哪个分支。这个过程被称为匹配正在考虑的值。

每个 switch 语句都必须是详尽的。也就是说，所考虑类型的每个可能的值必须与其中一个 case 情况相匹配。如果不适合为每个可能的值提供一个 case，则可以定义一个 default case 来覆盖任何未明确解决的值。这个默认值是由 default 关键字指示的，并且必须始终出现在最后。

此示例使用 switch 语句来考虑一个名为 someCharacter 的单个小写字符：
```
let someCharacter: Character = "z"
switch someCharacter {
case "a":
    print("The first letter of the alphabet")
case "z":
    print("The last letter of the alphabet")
default:
    print("Some other character")
}
// Prints "The last letter of the alphabet"
```
Switch 语句的第一个 case 英文字母的第一个字母 a，第二个 case 匹配最后一个字母 z。由于 switch 必须为每个可能的字符而不是每个字母字符都包含一个 case，所以这个 switch 语句使用 default 来匹配 a 和 z 以外的所有字符。这个规定确保了 switch 语句是详尽无疑的。
## 无隐式穿透（No Implicit Fallthrough）
与 C 和 Objective-C 中的 switch 语句相比，swift 中的 switch 语句不会穿透每个 case 的底部，默认情况是下一个。相反，只要第一个匹配的 switch case 完成，整个 switch 语句就结束它的执行，而不需要明确的 break 语句。这使得 switch 语句比 C 中的 switch 语句更安全，更易于使用，并避免错误地执行多个 switch case。

注意：虽然在 switch 中不需要 break，但是可以使用 break 语句来匹配和忽略特定的情况，或者在该情况已经完成执行前打破匹配的 case。

每个 case 的正文应包含一个可执行语句。写下面的代码是无效的，因为第一种情况是空的：
```
et anotherCharacter: Character = "a"
switch anotherCharacter {
case "a": // Invalid, the case has an empty body
case "A":
    print("The letter A")
default:
    print("Not the letter A")
}
// This will report a compile-time error.
```
与 C 中的 switch 语句不同，此 switch 语句不能同时匹配 “a” 和 “A”。相反，它会报告编译时错误，即 “a”：不包含任何可执行语句。这种方法避免了从一种情况到另一种情况的意外损失，并且使得安全的代码更加清晰。

要使用与 “a” 和 “A” 匹配的单个 case 进行 switch，请将两个值合并为一个复合 case，用逗号分隔这些值。
```
let anotherCharacter: Character = "a"
switch anotherCharacter {
case "a", "A":
    print("The letter A")
default:
    print("Not the letter A")
}
// Prints "The letter A"
```
为了便于阅读，复合大小写也可以写在多行上。

注意：要在特定的 switch case 的末尾明确穿透，请使用 fallthough 关键字。
## 区间匹配
可以检查 switch case 中的值是否包含在区间中。此示例中使用数字区间为任意大小的数字提供自然语言计数。
```
let approximateCount = 62
let countedThings = "moons orbiting Saturn"
let naturalCount: String
switch approximateCount {
case 0:
    naturalCount = "no"
case 1..<5:
    naturalCount = "a few"
case 5..<12:
    naturalCount = "several"
case 12..<100:
    naturalCount = "dozens of"
case 100..<1000:
    naturalCount = "hundreds of"
default:
    naturalCount = "many"
}
print("There are \(naturalCount) \(countedThings).")
// Prints "There are dozens of moons orbiting Saturn."
```
在上面的例子中，approximateCount 是在 switch 语句中求值的。 每种情况都会将该值与数字或区间进行比较。 由于 approximateCount 的值介于12和100之间，因此 naturalCount 被分配值 “dozens of”，并且将执行从 switch 语句中跳出。
## 元组
你可以使用元组在同一个 switch 语句中测试多个值。 元组中的每个元素都可以针对不同值或值的间隔进行测试。 或者，使用下划线字符（_）（也称为通配符模式）来匹配任何可能的值。

下面的示例采用（x，y）点，表示为类型（Int，Int）的简单元组，并将其分类到示例后面的图上。
```
let somePoint = (1, 1)
switch somePoint {
case (0, 0):
    print("\(somePoint) is at the origin")
case (_, 0):
    print("\(somePoint) is on the x-axis")
case (0, _):
    print("\(somePoint) is on the y-axis")
case (-2...2, -2...2):
    print("\(somePoint) is inside the box")
default:
    print("\(somePoint) is outside of the box")
}
// Prints "(1, 1) is inside the box"
```
![](http://upload-images.jianshu.io/upload_images/1694407-e616e221e410d29f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Switch 语句确定点是位于原点（0，0），红色x轴，橙色y轴，位于原点中心的蓝色4*4框内还是框外。

与 C 不同，Swift 允许多个 switch case 考虑相同的值或值。 事实上，在这个例子中，点（0,0）可以匹配所有四种情况。 但是，如果可能有多个匹配项，则始终使用第一个匹配的 case。 点（0,0）将首先匹配 case（0,0），因此所有其他匹配的 case 都将被忽略。
## 值绑定
Switch case 可以命名与临时常量或变量匹配的值，以便在 case 的主体中使用。 这种行为被称为值绑定，因为值被绑定到 case 正文中的临时常量或变量。

下面的例子将一个（x，y）点表示为一个类型为（Int，Int）的元组，并将其分类到下面的图表中：
```
let anotherPoint = (2, 0)
switch anotherPoint {
case (let x, 0):
    print("on the x-axis with an x value of \(x)")
case (0, let y):
    print("on the y-axis with a y value of \(y)")
case let (x, y):
    print("somewhere else at (\(x), \(y))")
}
// Prints "on the x-axis with an x value of 2"
```
![](http://upload-images.jianshu.io/upload_images/1694407-7a635fa5b5f24d73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Switch case 确定点是位于红色 x 轴上，橙色 y 轴上还是其他位置（在两个轴上）。

这三个 switch case 声明占位符常量 x 和 y，它们暂时从另一个点接受一个或两个元组值。第一种情况，case（let x，0）匹配任何 y 值为0的点，并将该点的 x 值赋给临时常量 x。类似地，第二种情况，case（0，let y）匹配 x 值为0的任意点，并将该点的 y 值赋给临时常量 y。

临时常量声明后，可以在 case 的代码块中使用它们。在这里，它们用于打印点的分类。

这个 switch 语句没有默认情况。最后一种情况，let（x，y）声明了一个可以匹配任何值的两个占位符常量的元组。由于另一个点总是一个包含两个值的元组，所以这种情况下所有可能的剩余值都是匹配的，并且不需要使用默认情况下的 switch 语句。
## Where
Switch case 可以使用 where 子句来检查附加条件。

下面的示例在下面的图中对（x，y）点进行分类：
```
let yetAnotherPoint = (1, -1)
switch yetAnotherPoint {
case let (x, y) where x == y:
    print("(\(x), \(y)) is on the line x == y")
case let (x, y) where x == -y:
    print("(\(x), \(y)) is on the line x == -y")
case let (x, y):
    print("(\(x), \(y)) is just some arbitrary point")
}
// Prints "(1, -1) is on the line x == -y"
```
![](http://upload-images.jianshu.io/upload_images/1694407-fdddfda5ed903508.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
switch 语句确定点是否位于 x == y 的绿色对角线上，在 x == -y 的紫色对角线上，或者两者都不是。

这三个 switch case 声明了占位符常量 x 和 y，它们暂时从 yetAnotherPoint 中获取两个元组值。 这些常量用作 where 子句的一部分，以创建动态过滤器。 只有当 where 子句的条件评估为 true 时，switch case 才匹配当前的点值。

和前面的例子一样，最后一个例子匹配所有可能的剩余值，所以不需要使用默认情况来完成 switch 语句。
## 复合 case
共享相同主体的多个 switch case 可以通过在 case 之后写入多个模式来组合，每个模式之间以逗号分隔。 如果任何模式匹配，则认为该情况匹配。 如果列表很长，模式可以写入多行。 例如：
```
let someCharacter: Character = "e"
switch someCharacter {
case "a", "e", "i", "o", "u":
    print("\(someCharacter) is a vowel")
case "b", "c", "d", "f", "g", "h", "j", "k", "l", "m",
     "n", "p", "q", "r", "s", "t", "v", "w", "x", "y", "z":
    print("\(someCharacter) is a consonant")
default:
    print("\(someCharacter) is not a vowel or a consonant")
}
// Prints "e is a vowel"
```
switch 语句的第一个例子与英语中的所有五个小写元音匹配。 同样，它的第二个 case 匹配所有小写英语辅音。 最后，默认情况下匹配任何其他字符。

复合 case 也可以包含值绑定。 复合 case 的所有模式都必须包含相同的一组数值绑定，并且每个绑定必须从复合 case 中的所有模式中获取相同类型的值。 这可以确保：无论 case 的哪个部分匹配，案件正文中的代码都可以访问绑定的值，并且该值始终具有相同的类型。
```
let stillAnotherPoint = (9, 0)
switch stillAnotherPoint {
case (let distance, 0), (0, let distance):
    print("On an axis, \(distance) from the origin")
default:
    print("Not on an axis")
}
// Prints "On an axis, 9 from the origin"
```
上面的情况有两种模式：(let distance, 0) 匹配 x 轴上的点，(0, let distance) 匹配 y 轴上的点。 两种模式都包含 distance 和 distance 的绑定，这两种模式都是一个整数 - 这意味着 case 正文中的代码始终可以访问 distance 的值。

如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~

