# Objective-C 转 Swift 的第一道坎——论如何正确的处理可选类型

从 `Objective-C` 转 `Swift` 开发已经有一段时间了，这两门语言在整体的理念上差异还是蛮大的，在这之中可选类型的处理是每一个使用 `Swift` 的开发者每天都要面临的问题，理解并正确处理好可选类型对于写出高质量的 `Swift` 代码和保证 `iOS` 项目的健壮性都是至关重要的。

## 可选类型

要想处理好可选类型，就要先理解可选类型。

一个可选类型代表有两种可能性：有一个值，你可以解包可选类型来访问该值；或者根本没有值。

在 `Objective-C` 中不存在可选类型的概念，`Objective-C` 中最接近的东西就是 `nil`, `nil` 的意思是“没有有效的对象”。但是，这只适用于对象——它不适用于结构、基本数据类型或枚举值。对于这些类型，`Objective-C` 方法通常会返回一个特殊值（如 `NSNotFound`）来指示缺少值。这种方法假设方法的调用者知道有一个特殊的值来测试，并记得检查它。`Swift` 的可选值可以让你指出可能为 `nil` 的任何类型的值，而不需要特殊的常量。

例如：

```
var serverResponseCode: Int? = 404
```

可选的 `Int` 被写为 `Int?`，而不是 `Int`。问号表示它所包含的值是可选的，这意味着它可能包含一个 `Int` 值，或者它可能根本不包含任何值。

### nil

通过赋值给它一个特殊的值 `nil` 来设置一个可选变量为无值状态：

```
var serverResponseCode: Int? = 404
// serverResponseCode 包含一个实际的 Int 值为 404
serverResponseCode = nil
// serverResponseCode 现在不包含任何值 
```

如果你定义了一个可选变量而不提供默认值，则该变量会自动设置为 `nil`：

```
var surveyAnswer: String?
// surveyAnswer 自动设置为 nil
```

`Swift` 的 `nil` 与 `Objective-C` 中的 `nil` 不相同。在 `Objective-C` 中，`nil` 是一个指向不存在对象的指针。在 `Swift` 中，`nil` 不是一个指针，它是缺少某种类型的值。任何类型的可选值都可以被设置为 `nil`，而不仅仅是对象类型。

## 处理可选类型

在 `Swift` 中，处理可选类型总体而言有四种方式：强制解包、可选绑定、隐式解包和可选链。接下来我们将简要介绍一下这四种方式：

### 强制解包

可以通过在可选值名称的末尾添加感叹号（`!`）来访问其内部值。这被称为强制解包一个可选的值。

```
print("serverResponseCode is (\serverResponseCode!)")
```  

试着用 `!` 访问不存在的可选值会触发运行时错误。在使用强制解包之前，一定要确保一个可选值不为 `nil`

```
if serverResponseCode != nil {
    print("serverResponseCode is (\serverResponseCode!)")
}
```

### 可选绑定

你可以使用可选绑定来发现可选值是否包含值，如果有，则使用该值用作临时常量或变量。可选绑定可以与 `if` 和 `while` 语句一起使用，以检查可选值内部的值，并将该值提取为常量或变量，作为单次操作的一部分。

使用 `if` 语句编写一个可选绑定，如下所示： 

```
if let code = serverResponseCode {
    print("serverResponseCode is (\code)")
} else {
    print("serverResponseCode is nil")
}
```

如果转换成功，那么 `code` 常量可以在 `if` 语句的第一个分支中使用。它已经被初始化为包含在非可选的值中，所以没有必要使用 `!` 后缀来访问它的值。

你可以使用可选绑定的常量和变量。如果你想在 `if` 语句的第一个分支内操作 `code` 的值，你可以写 `if var code`，使得可选值作为一个变量而非常量。

你可以根据需要在单个 `if` 语句中包含尽可能多的可选绑定和布尔条件，并用逗号分隔。如果可选绑定中的任何值为 `nil`，或者任何布尔条件的计算结果为 `false`，则整个 `if` 语句的条件被认为是错误的。一下 `if` 语句是等价的：

```
if let firstNumber = Int("4"), let secondNumber = Int("42"), firstNumber < secondNumber && secondNumber < 100 {
    print("\(firstNumber) < \(secondNumber) < 100")
}
// 打印 "4 < 42 < 100"
 
if let firstNumber = Int("4") {
    if let secondNumber = Int("42") {
        if firstNumber < secondNumber && secondNumber < 100 {
            print("\(firstNumber) < \(secondNumber) < 100")
        }
    }
}
// 打印 "4 < 42 < 100"
```

### 隐式解包可选类型

有时从程序的结构中可以清楚的看到，在第一次设置值之后，可选值将始终有一个值。在这些情况下，每次访问时都不需要检查和解包可选值，因为可以安全地假定所有的时间都有一个值。

这些可选值被定义为隐式解包可选值。你写一个隐式解包的可选值，在你想要的可选类型之后放置一个感叹号（`String!`）而不是一个问号（`String?`）

隐式解包可选值的背后是普通可选值，但也可以像非可选值一样使用，而不必在每次访问时解包可选值。

```
let possibleString: String? = "An optional string."
let forcedString: String = possibleString! // 需要感叹号
 
let assumedString: String! = "An implicitly unwrapped optional string."
let implicitString: String = assumedString // 不需要感叹号
```

如果隐式解包可选值为 `nil`，并且你尝试访问其包装的值，则会触发运行时错误。

你仍然可以对隐式解包可选值使用强制解包和可选绑定。

### 可选链

可选链是查询和调用可能当前为 `nil` 的可选属性，方法和下标的过程。如果可选值包含一个值，那么属性，方法和下标调用将会成功；如果可选值为 `nil`，则属性，方法和下标调用返回 `nil`。多个查询可以链接在一起，如果链中的任何链接有一个为 `nil`，则整个链接将优雅的失败。

可选链可以作为强制解包的替代。

定义两个名为 `Person` 和 `Residence` 的类：

```
class Person {
    var residence: Residence?
}
 
class Residence {
    var numberOfRooms = 1
}
```

创建一个新的 `Person` 示例，由于是它是可选类型，所以它的 residence 属性默认初始化为 `nil`。

```
let john = Person()
```

如果采用强制解包的方式访问 `john` 的 `numberOfRooms` 属性，则会触发运行时错误：

```
let roomCount = john.residence!.numberOfRooms
// 这会触发运行时错误
```  

可选链提供了另一种访问 `numberOfRooms` 值的方法。要使用可选链，请使用问号代替感叹号：

```
if let roomCount = john.residence?.numberOfRooms {
    print("John's residence has \(roomCount) room(s).")
} else {
    print("Unable to retrieve the number of rooms.")
}
// 打印 "Unable to retrieve the number of rooms."
```

可选链可以访问属性：

```
if let roomCount = john.residence?.numberOfRooms {
    print("John's residence has \(roomCount) room(s).")
} else {
    print("Unable to retrieve the number of rooms.")
}

john.residence?.numberOfRooms = 2
```

可选链可以调用方法：

```
class Person {
    var residence: Residence?
}
 
class Residence {
    var numberOfRooms = 1
    func printNumberOfRooms () {
        print("John's residence has \(numberOfRooms) room(s).")
    }
}
...
john.residence?.printNumberOfRooms()
```

可选链可以访问下标：

```
var testScores = ["Dave": [86, 82, 84], "Bev": [79, 94, 81]]
testScores["Dave"]?[0] = 91
testScores["Bev"]?[0] += 1
testScores["Brian"]?[0] = 72
// the "Dave" array is now [91, 82, 84] and the "Bev" array is now [80, 94, 81]
```

就是这样。

## Swift 中处理可选类型的建议

因为 `Objective-C` 中的 `nil` 对于开发者来说是相对安全的，向集合类型中添加 `nil` 会造成异常，但给 `nil` 发送消息并不会有任何的问题(当然业务上可能会有问题)。但在 `Swift` 中，就像大多数其他语言一样，向 `nil` 发送消息会造成 `crash`。而且作为典型的现代强类型语言，可选类型的加入更是给之前长期使用 `Objective-C` 这种算是弱类型语言的 `iOS` 开发者带来了困扰。再此给开发者们一些处理可选类型的建议：

### 尽可能避免声明可选类型的实例

除非一些必要的场景（例如代理模式），尽可能的使用非可选类型，包括但不限于属性声明和方法参数。

### 多使用可选绑定和可选链处理可选类型，避免使用强制解包和隐式解包

`Swift` 选用 “`!`” 作为强制解包和隐式解包的标志是有原因的，这是在提醒我们这是一种很危险的操作，往往会在意想不到的时候给我们的应用带来额外的 `crash`。

### 不仅要处理可选绑定和可选链的命中分支，对于 else 的情况也要进行额外情况的处理

在进行可选绑定时，可选类型不为 `nil` 的场景我们都会进行处理，但往往会忽视 `else` 的情况，尽可能也进行处理，那怕只是一句 `log`。

### 进行可选绑定时，尽量使用同名的局部变量

最佳实践是用同名的局部变量来可选绑定可选值，这样可以保证上下文清晰，不会因为出现了新的局部变量导致阅读代码的人反复对照。

```
if let serverResponseCode = serverResponseCode {
    print("serverResponseCode is (\serverResponseCode)")
} else {
    print("serverResponseCode is nil")
}
```

至此，关于 `Swift` 中可选类型的处理就告一段落了，由于 `Swift` 是一门强类型的语言，如果有哪些场景是我们处理的不正确的，编译器也会给出相应的提示，但是真正的危险可能不仅止于此……

## Objective-C 和 Swift 混编时如何正确的处理可选类型

除了一些在最近一段时间刚刚从零启动的项目，绝大多数的项目都是处于从 `Objective-C` 向 `Swift` 代码过渡的阶段，这里面涉及到了对原有 `Objective-C` 代码进行可选非可选区分的问题。如果读过一些进行过适配 `Swift` 的 `Objective-C` 写的三方库的源代码之后会发现，很多都用到了这样的一对宏：

```
NS_ASSUME_NONNULL_BEGIN
...
NS_ASSUME_NONNULL_END
```

这对宏的意思是，在这对宏之间声明的属性和方法，其中涉及到的类型都是非可选类型的。很多开发的同学发现这样一种简单而又粗暴的将 `Objective-C` 一键适配到 `Swift` 的方法之后，果断的在所有的头文件中的开始和结尾处加上这对宏。然后悲剧就发生了：

```
NS_ASSUME_NONNULL_BEGIN

// 如果设备的内存处于极小的情况下，会返回 nil
@property (nonatomic, strong) DataBase *dataBase;

NS_ASSUME_NONNULL_END
```

在寻常的情况下调用数据库属性并不会有任何的问题，如果设备的内存处于极小的情况下，会返回 `nil`，这在纯 `Objective-C` 的代码中也不会有什么问题，但是当混编时：

```
let x = object.dataBase().fetchUserInfo() // 当 dataBase 返回 nil 时，会 crash。
```

因为你已经通过宏声明了 `dataBase` 属性是非可选的，所以编译器就会认为这个属性是非可选的，不会给出任何处理可选的提示。

看到这里你可能会说，对于这类情况，可以通过判断是否为 `nil` 来进行处理，比如：

```
if object.dataBase() != nil {
    ...
}
```

这在 `debug` 模式下是行的通的，但是在 `release` 模式下，`iOS` 系统为了优化性能，会对所有标记了非可选类型的对象的与 `nil` 的比较直接认为是 `true`，直接落入了括号中，造成更不可查的 `crash`。所以最好的处理方式是对任何可能出现 `nil` 可能的属性或方法参数都加上 `nullable`：

```
@property (nonatomic, strong, nullable) DataBase *dataBase;
```

这样就可以通知编译器这是一个可选类型属性，该有的一些提示和处理也会由编译器来提供。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~