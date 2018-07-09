# 优化 Swift 性能

随着时间的推进，Swift 从一门“每过一年重学一次”的语言，演进到了越来越稳定的一门工业级语言。经过了几年的摸索和尝试，越来越多的公司选择以 Swift 为基础作为构建 APP 的主要语言。那么接下来我们将从性能优化的角度来看一看，如何写出高性能的 Swift。

## Whole Module Optimizations

Whole Module Optimizations 是一种新的编译器优化模式，它可以使程序运行的更快。要想了解 Whole Module Optimizations，就要先了解 Xcode 编译文件的方式。

Xcode 独立编译文件，并且可以在机器中的多个核心上并行编译文件，与此同时， Xcode 只会重新编译需要更新的文件。但问题在于，Xcode 对于编译的优化能力仅限于一个文件的范围。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-1.png?raw=true)

通过 Whole Module Optimizations， 编译器能够立即优化整个模块，它可以分析所有内容并进行积极的优化。唯一的问题在于，采用了 Whole Module Optimizations 会导致整个构建过程需要更长的时间，但生成的二进制文件通常运行得更快。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-2.png?raw=true)

所以最佳的方式是，在 Debug 模式下不启动优化，仅在 Release 模式下启用 Whole Module Optimizations。

开启 Whole Module Optimizations，的方式也很简单，在 Xcode 中的 Optimization Level 中选择含有 Whole Module Optimizations 的选项即可。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-3.png?raw=true)

## 高性能 Swift 代码

我们将从引用计数，泛型和动态派发三个方面来了解如何优化 Swift 代码的性能。

### 引用计数

Swift 与 Objective-C 相同，都是用过引用计数机制来实现内存管理的。引用计数有其方便性，但是同样也会引起性能的消耗，Objective-C 中的结构使用很不方便，到了 Swift 时代，为了规避引用技术的消耗，我们往往会通过用结构来替换类。

现在让我们考虑一个稍微更详细的例子，考虑一个带有引用的结构。例如，在应用程序中有一个用来对点建模的结构 Point，在迭代的过程中，我希望让我的点能够记录颜色。所以我再 Point 结构中添加了一个 UIColor 属性。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-4.png?raw=true)

虽然结构本身并不需要在赋值时进行引用计数修改，但如前所述， 如果结构包含引用，它确实需要进行修改。这是因为分配结构等同于彼此独立地分配其每个属性。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-5.png?raw=true)

当然，如此简单的例子可能看不出什么问题来，接下来看一个更复杂的例子。

假如我们有一个 User 结构，我们用 User 结构来为应用程序中的单个用户建模。并且每个用户实例都有一些与之关联的数据，即三个字符串，一个用于用户的名字， 一个用于用户的姓氏，一个用于用户的地址。与此同时，还有一个数组字段和一个存储有关用户的特定于应用程序的数据的字典。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-6.png?raw=true)

尽管所有这些属性都是值类型，但在内部，它们包含一个用于管理其内部数据生命周期的类。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-7.png?raw=true)

所以这意味着每次我分配其中一个结构时，每次将它传递给函数时，实际上必须执行五次引用计数修改。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-8.png?raw=true)

我们可以通过使用包装类来解决这个问题。即我们将 User 结构包含在一个包装类中。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-9.png?raw=true)

我仍然可以使用类引用来操作结构，更重要的是，如果我将此引用传递给函数，或者我声明或者使用引用初始化变量，只执行一个引用计数增量。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-10.png?raw=true)

### 泛型

这里有一个泛型函数 min:

```
func min<T : Comparable>(x: T, y: T) -> T {
    return y < x ? y : x
}
```

这只是一个非常简单的函数。但实际上，幕后工作比你想象的要多。用伪代码来表示编译器处理后的代码如下：

```
func min<T : Comparable>(x: T, y: T, FTable: FunctionTable) -> T {
    let xCopy = FTable.copy(x)
    let yCopy = FTable.copy(y)
    let m = FTable.lessThan(yCopy, xCopy) ? y : x
    FTable.release(x)
    FTable.release(y)
    return m
}
```

首先注意编译器使用间接 来比较 x 和 y。这是因为我们可以将两个整数传递给min函数，我们也可以传入两个浮点数或两个字符串——我们可以传递任何类似的类型。所以编译器在所有情况下都必须是正确的，并且能够处理它们中的任何一个。

另外，因为编译器不能知道 T 是否需要引用计数修改， 所以它必须插入额外的代码， 因此 min 函数可以处理需要引用计数的类型 T 和不需要引用计数的类型 T。在这两种情况下，编译器都是保守的，因为它必须能够处理任何类型 T。

对于这样的情况，编译器通过“泛型特殊化”来对代码进行优化。

例如，这里有一个函数 foo，它将两个整数传递给泛型 min 函数。

```
func foo()  {
   let x: Int = ...
   let y: Int = ...
   let r = min(x, y)
   ...
}
```

当编译器执行泛型特殊化时，首先会查看 foo 对 min 的调用，并看到传递给 min 函数的是两个整数。接着，由于编译器可以看到泛型 min 函数的定义，它可以克隆 min 函数并通过特定类型 Int 替换泛型类型 T 来特殊化这个泛型函数。

```
func min<Int>(x: Int, y: Int, FTable: FunctionTable) -> Int { 
    let xCopy = FTable.copy(x)
    let yCopy = FTable.copy(y)
    let m = FTable.lessThan(yCopy, xCopy) ? y : x 
    FTable.release(x)
    FTable.release(y)
    return m 
}
```

然后这个特殊化了的函数针对 Int 进行了优化，并且删除了与此函数相关的所有额外开销（引用计数相关）。最后，编译器通过调用特殊化过的 min 函数替换原有的泛型 min 函数的调用，从而实现进一步的优化。

```
func foo() {
    let x: Int = ...
    let y: Int = ...
    let r = min<Int>(x, y) ...
}
 
func min<Int>(x: Int, y: Int) -> Int {
    return y < x ? y : x
}
```

泛型特殊化是一个非常强大的优化，但它确实有一个限制：泛型定义的可见性。如果编译文件时看不到泛型 min 函数的定义，就无法实施此项优化。但如果启用了 Whole Module Optimizations，即使是位于不同的两个文件中的定义也是互相可见的，优化就可以顺利的进行下去了。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-11.png?raw=true)

虽然通用专业化是一个非常强大的优化， 但它确实有一个限制; 即， 即通用定义的可见性。

例如，这种情况， min-T函数的通用定义。

这里我们有一个函数compute ，它调用一个带有两个整数的泛型min-T函数。

在这种情况下，我们可以执行通用专业化吗？

好吧，即使编译器可以看到 两个整数被传递 给泛型min-T函数， 因为我们分别编译文件1.Swift 和文件2.Swift， 编译器看不到文件2中 函数的定义当编译器正在编译文件1时。

所以在这种情况下，编译器 在编译文件1时 看不到泛型min-T函数的定义，因此我们必须调用泛型min-T函数。

但是，如果我们启用了整个模块优化怎么办？

好吧，如果我们启用了整个模块优化， 则文件1.Swift和文件2.Swift会一起编译。

这意味着 当您一起编译文件1或文件2时，文件1 和文件2中的 定义都是可见的。

所以基本上，这意味着 当我们编译文件1时， 可以看到通用的min-T函数，即使它在文件2中。

因此，我们能够将通用min-T函数专门 化为min int，并用min Int替换对min-T的调用。

这只是 整个模块优化的力量明显的一种情况。

在这种情况下 ，编译器可以执行通用规范的唯一原因是，通过启用整个模块优化提供了额外的信息。

### 动态派发

这里有一个 Pet 类的类层次结构。Pet 有一个方法 noise，一个属性 name 和一个方法 noiseimpl 用于实现方法 noise。Pet 有一个名为 Dog 的子类，它可以覆写 noise 方法。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-12.png?raw=true)

现在看 makeNoise 函数：

```
func makeNoise(p: Pet) {
    print("My name is \(p.name)")
    p.noise()
}
```

这是一个非常简单的函数，它有一个参数 p ，它是 Pet 类的一个实例。尽管这段代码只涉及少量的源代码，但在幕后发生的事情要比我们想象的要多得多。

例如，以下伪 Swift 代码不是编译器实际生成的代码。name 和 noise 方法不会被直接调用。

```
func makeNoise(p: Pet) {
    print("My name is \(p.name)")
    p.noise()
}
```

相反，编译器会生成此代码。注意这里的间接用于调用 name getter 或方法 noise。

```
func makeNoise(p: Pet) {
    let nameGetter = Pet.nameGetter(p)
    print("My name is \(nameGetter(p))")
    let noiseMethod = Pet.noiseMethod(p)
    noiseMethod(p)
}
```

编译器必须插入此间接调用， 因为它无法知道给定当前类层次结构是否要由子类重写属性 name 或方法 noise。只有当编译器可以证明任何子类都没有覆盖 name 和 noise 时，才会发出直接调用。

Swift 中有两种语言功能，可以用来约束 API 的类层次结构。第一个是对继承的约束，第二个是通过访问控制进行访问的约束。

让我们首先谈谈继承约束， 即 final 关键字。当 API 包含附加了 final 关键字的声明时，API 正在传达该声明永远不会被子类覆盖。

考虑一下刚才的例子，默认情况下，编译器必须使用间接调用 name 的 getter。这是因为没有更多信息，编译器无法知道 name 是否被子类覆盖。但是我们知道在这个 API 中，name 不能被覆盖。因此，可以通过将 final 关键字附加到 name 来强制执行此操作并进行通信。然后编译器可以查看 name，并发现 name 永远不会被子类覆盖，并且可以消除动态派发。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-13.png?raw=true)

接下来看一下访问控制。原来在这个 API 中，Pet 和 Dog 都在单独的文件中（Pet.swift 和 Dog.swift），但是在同一个模块中（模块 A）。另外，在另一个模块中有另一个名为 Cat 的 Pet 的子类，在 cat.swift 文件中。

默认情况下，编译器不可以直接调用 noiseimpl，因为编译器必须假定此 API（noiseimpl） 用于在像 Cat 和 Dog 这样的子类中重写。但我们知道 noiseimpl 是 Pet.swift 的私有实现细节，并且它不应该在 Pet.swift 之外可见。

我们可以通过将 private 关键字附加到 noiseimpl 来强制执行此操作。一旦我们将 private 关键字附加到 noiseimpl，在 Pet.swift 之外就不再可以看到 noiseimpl。这意味着编译器可以立即知道在 Cat 或 Dog 中不能覆盖 noiseimpl，因为它们不在 Pet.swift 中。并且因为 Pet.swift 中只有一个类实现了 noiseimpl，即 Pet，在这种情况下，编译器可以直接调用 noiseimpl。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-14.png?raw=true)

接下来来看一下 Whole Module Optimizations 访问控制之间的相互作用。我们一直在讨论 Pet 类的问题，但 Dog 的情况呢？

请记住，Dog 是 Pet 的子类，具有内部访问权限而不是公共访问权限。如果我们在 Dog 类的实例上调用 noise， 没有更多信息，编译器必须插入间接调用， 因为它无法知道模块 A 的不同文件中是否存在 Dog 的子类。但是，当我们启用 Whole Module Optimizations 时，编译器具有模块范围的可见性——它可以一起查看模块中的所有文件——所以编译器可以直接在 Dog 类的实例上调用 noise。通过给编译器提供更多信息，允许编译器理解我的类层次结构，我们可以免费获得这种优化，而无需做任何额外的工作。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/images/1-15.png?raw=true)

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~

> 参考文献：

> 1、[Optimizing Swift Performance](https://developer.apple.com/videos/play/wwdc2015/409/)