# 理解 Swift 性能

Swift 为开发者提供了广泛而强大的设计空间。Swift 有多种一级类型、代码重用和动态机制。那么，如何缩小设计空间，并未工作挑选合适的工具呢？

首先，你需要考虑 Swift 不同抽象机制的建模含义。值或引用哪一个更合适？动态能力到哪一步为止？根据经验，考虑性能影响通常有助于指导我找到更具惯用性的解决方案。

理解 Swift 抽象机制的性能影响的最好方法是理解它们的底层实现。

## 了解实现以了解性能

当构建抽象并选择抽象机制时，应该问自己三个问题：1、实例是在栈还是堆上分配？2、当传递这个实例时，将承担多少引用计数开销？3、当在这个实例上调用一个方法时， 是静态还是动态调度？

当我们想书写高性能的 Swift 代码的时候，就要避免为我们没有用到的动态机制和运行时付出额外的代价。我们需要知道何时在这些不同的维度之间切换，来获得更好的性能。

### 内存分配

Swift 会为我们自动分配和释放栈上的内存。栈是一个非常简单的数据结构，你可以对栈进行 Push 和 Pop 操作。因为你只能添加或移除到栈的末尾，所以我们可以通过保持指向栈末尾的指针来实现栈的 Push 和 Pop 操作。

当我们调用一个函数时，可以通过简单的减少栈指针来分配我们需要的内存。当完成执行函数时，可以通过将栈的指针移到调用此函数之前的位置来简单地释放该内存。始终栈的效率成本是很低的。

堆与此相反，堆更动态，但效率比栈低。堆可以动态的分配内存，但这需要更先进的数据结构。如果要在堆上分配内存，必须搜索堆数据结构，以查找适当大小的未使用块。当完成使用时，要释放内存，必须将该内存插入适当的位置。

由于多个线程可以同时在堆上分配内存 ，因此堆需要使用锁或其他同步机制来保护其完整性。这是一个相当大的成本。

让我们浏览一些代码 ，看看 Swift 做了些什么。这里有一个带有 x 和 y 存储属性的 Point 结构。它上面也有 draw 方法。我们将在（0,0）处构造点，将 point1 赋值给 point2 进行复制，并为 point2.x 赋值 5 。然后，我们将使用 point1 并使用 point2。

```
// Allocation
// Struct

struct Point {
   var x, y: Double
   func draw() { ... }
}

let point1 = Point(x: 0, y: 0)
var point2 = point1
point2.x = 5
// use `point1`
// use `point2`
```

当进入这个函数时， 在开始执行任何代码之前，已经为的 point1 实例和 point2 实例分配了栈空间。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-1.png?raw=true)

因为point是一个结构， 所以 x 和 y 属性在栈中存储在行中。

所以，当我们用 x 为 0和 y 为 0 来构造点时，所做的就是初始化已经在栈上分配的内存。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-2.png?raw=true)

当我们将 point1 分配给 point2 时，只是复制该点并再次初始化 point2 内存， 我们已经在栈上进行了分配。请注意，point1 和 point2 是独立的实例。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-3.png?raw=true)

这意味着，当我们为 point2.x 赋值 5 时，point2.x 为 5，但 point1.x 仍为 0。这称为值语义。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-4.png?raw=true)

然后继续使用 point1，使用 point2， 然后就完成了我们的功能。

因此，可以通过将该堆栈指针递增回我们进入函数时的位置来简单地为 point1 和 point2 释放该内存。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-5.png?raw=true)

让我们将它与完全相同的代码进行对比，但使用的是一个类，而不是一个结构。

```
// Allocation
// Class

class Point {
   var x, y: Double
   func draw() { ... }
}

let point1 = Point(x: 0, y: 0)
let point2 = point1
point2.x = 5
// use `point1`
// use `point2`
```

所以，当我们进入这个函数时，就像之前一样， 我们在栈上分配内存。但是，不是为了实际存储属性 ，我们将为 point1 和 point2 的引用分配内存。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-6.png?raw=true)

我们将在堆上分配对内存的引用。当我们在（0,0）处构造点时，Swift 将锁定堆并搜索该数据结构以寻找适当大小的未使用的内存块。

!()[https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-7.png?raw=true]

一旦我们拥有它，我们可以使用 x 为 0，y 为 0 初始化该内存，并且我们可以将内存地址的 point1 引用初始化为堆上的内存。注意，当我们在堆上分配它时，Swift 实际上为类分配了四个存储字长。这与 Point 作为结构时分配的两个字长形成对比。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-8.png?raw=true)

这是因为现在 Point 是一个类， 除了为 x 和 y 存储所需的空间之外，还要分配两个 Swift 将代表我们管理的空间。在图中用这些蓝色框表示。

当我们将 point1 分配给 point 时，不会像在 point1 是结构时那样复制点的内容。相反，我们将复制引用。因此，point1 和 point2 实际上是指堆上相同的点对象。

这意味着当我们去为 point2.x 分配值 5 时， point1.x 和 point2.x 都有一个值为 5。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-9.png?raw=true)

这被称为引用语义，可能导致意外的状态共享。

然后，我们将使用 point1，使用 point2， 然后 Swift 将为我们释放这个内存，锁定堆并将未使用的块重新插入到适当的位置。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-10.png?raw=true)

接着出栈。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-11.png?raw=true)

简而言之，类的使用比结构成本更好，因为类对象的内存是分配在堆上的。类又有点，如标识和简介存储，但是当我们不需要这些特性时，使用结构会更好，同时可以避免无意中的共享状态。

### 引用计数

栈是系统帮我们管理的，堆却是需要程序员自己管理。Swift 如何知道何时释放它在堆上分配的内存是安全的？答案是 Swift 保持对堆上任何实例的引用总数的计数，并且是保留在实例本身上。retain 或 release 会造成引用计数的递增或递减。当引用计数为 0 时，Swift 知道没有人再指向堆上的这个实例，那么释放它就是安全的。

引用计数是一个非常频繁的操作，它不仅仅是递增或递减整数。更重要的是，就像堆分配一样，引用计数的加减需要考虑到线程安全性，因为可以同时向多个线程上的任何堆实例 retain 或 release，实际上必须以原子方式改变引用计数。

让我们回到之前的代码，看看 Swift 实际上做了些什么。

// Reference Counting
// Class (generated code)

class Point {
   var refCount: Int
   var x, y: Double
   func draw() { ... }
}

let point1 = Point(x: 0, y: 0)
let point2 = point1
retain(point2)
point2.x = 5
// use `point1`
release(point1)
// use `point2`
release(point2)

在堆上生成了 Point 对象后，它的引用计数为1。当把 point1 分配给 point2 时，Point 对象引用计数为 2，Swift 添加了一个原子的调用来增加引用计数。一旦完成对 point1 的使用，Swift 添加了一个原子的调用来减少引用计数。类似的，一旦使用完 point2 ，Swift 就添加了一个原子的调用来减少引用计数。此时，没有更多的引用正在使用点实例，因此 Swift 知道锁定堆并将该内存块返回给它是安全的。

Point 结构在使用中，是不涉及引用计数的。但如果结构包含类，那么情况就产生了变化。

这里有一个 Label 结构，其中包含 String 类型的 text，和 UIFont 类型的 font。

```
struct Label {
   var text: String
   var font: UIFont
   func draw() { ... }
}

let label1 = Label(text: "Hi", font: font)
let label2 = label1
// use `label1`
// use `label2`
```

正如之前所述，String 将其字符的内容存储在堆上，所以需要使用到引用计数。同理，UIFont 也是如此。如果我们查看内存表示，Label 有两个引用。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-12.png?raw=true)

当我们复制它时，实际上又增加了两个引用。一个引用到 text，另一个引用到 font。Swift 跟踪这些堆分配的方式是添加对 retain 和 release 的调用。所以在这里会产生两次类的引用计数开销。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-13.png?raw=true)

所以，如果结构包含类对象，那么它们也将支付引用计数开销。事实上，结构将支付与它们包含的引用数量成比例的引用计数开销。因此，如果一个结构中有多个类对象，会引起比类更多的引用计数开销。

```
// Reference Counting
 // Struct containing references (generated code)

struct Label {
   var text: String
   var font: UIFont
   func draw() { ... }
}

let label1 = Label(text: "Hi", font: font)
let label2 = label1
retain(label2.text._storage)
retain(label2.font)
// use `label1`
release(label1.text._storage)
release(label1.font)
// use `label2`
release(label2.text._storage)
release(label2.font)
```
### 方法调度

在运行时调用方法时，Swift 需要执行正确的实现。如果可以确定在编译时执行的实现，就称之为静态调度。在运行时，我们将能够直接跳转到正确的实现。编译器可以非常积极地优化这些代码，包括内联等方式。这与动态调度形成对比。

动态调度在编译时是无法确定指向哪个实现的，因此在运行时，我们将查找实现，然后跳转到它。就其本身而言，动态调度并不比静态调度昂贵得多。虽然动态调度没有像引用计数和堆分配中那样的线程同步开销，但是这种动态调度会阻止编译器的可见性， 因此虽然编译器可以为我们的静态调度（动态调度） 执行所有这些非常酷的优化，但编译器无法通过它进行推理。比如内联。

那么什么是内联呢？我们回到熟悉的结构 Point。它有一个 x 和 y，它有一个 draw 方法。还添加了一个 drawAPoint 方法。drawAPoint 方法接受一个 Point，只是调用 draw 就可以了。

```
// Method Dispatch
// Struct (inlining)
struct Point {
    var x, y: Double
    func draw() {
        // Point.draw implementation
    } 
}

func drawAPoint(_ param: Point) {
   param.draw()
}

let point = Point(x: 0, y: 0)
drawAPoint(point)
```

然后我的程序的主体在（0, 0）构造一个点并将该点传递给 drawAPoint。drawAPoint 函数和 point.draw 方法都是静态调度的。这意味着编译器确切地知道将要执行哪些实现 ，因此它只是将其替换为 drawAPoint 的实现。

```
// drawAPoint(point)
point.draw()
```

这就解释了什么是静态调度以及静态调度为何比动态调度更快的原因。

与单个动态调度相比，单个静态调度没有那么大差别，而整个静态调度链，编译器将通过整个链来查看。动态调度链则无法被推断。编译器将能够将一系列静态方法调度折叠成一个没有调用堆栈开销的单个实现。

既然如此，那我们为什么还需要动态调度呢？其中一个原因是它可以实现像多态这样强大的功能。

### 总结

每当你阅读和编写 Swift 代码时， 你应该问自己三个问题：这个实例是在栈还是堆上分配的？当传递这个实例时， 将要承担多少引用计数修改的开销？当在这个实例上调用一个方法时，是静态还是动态调度？想清楚了这三个问题，优化你的 Swift 的代码就将不再是问题了。

## 协议

协议是现代强类型语言中，实现多态的一种方式。在 Swift 中，不仅仅是类，结构和枚举都可以遵守协议。那么这一切是怎样实现的呢？让我们来看一下。

```
protocol Drawable { func draw() }

struct Point : Drawable {
   var x, y: Double
   func draw() { ... }
}

struct Line : Drawable {
   var x1, y1, x2, y2: Double
   func draw() { ... }
}

var drawables: [Drawable]
for d in drawables {
    d.draw() 
}
```

这里有两个结构 Point 和 Line，它们同时都遵守 Drawable 协议。同时有一个数组，数组里面存储的是 Drawable 协议的类型。遍历这个数组，同时执行 draw 方法。那么，Swift 如何调度到正确的方法呢？

这个问题的答案是基于表格的机制，称为 Protocol Witness Table (PWT)。每种类型都有一张表，会在你的应用中实现协议。并且表中的条目，会链接到类型中的一个实现。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-4.png?raw=true)

如何寻找协议中方法的实现我们已经知道了，那么一个数组中存储的数据类型不一致，是如何保证对齐存储的呢？Swift 使用了一种叫做 Existential Container 的技术。存储空间中的前三个字长，是留给 valueBuffer 的。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-15.png?raw=true)

小类型——比如 Point 类型，只需要两个字，刚好能放进 valueBuffer 中。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-16.png?raw=true)

大类型——Line 类型占据了四个字长，这时会申请堆中的内存，并将指针存入 valueBuffer 中。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-17.png?raw=true)

那么 Existential Container 是如何管理这个不同点的呢？同样还是基于表的机制，叫做 Value Witness Table (VWT)。Value Witness Table 会管理值的有效期。在程序中，每种类型都会有一张表。以 Line 类型为例，数组在最开始会调用 LineVWT 中的 allocate: 方法，并从堆中申请存储空间，并将指针存入 valueBuffer 中。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-18.png?raw=true)

接下来，Swift 需要将值初始化我们的局部变量的值复制到存在容器中。同样，我们在这里有一个 Line，因此 我们的 VWT 的 copy 方法将执行正确的操作并将其复制到堆中分配的 valueBuffer 中。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-19.png?raw=true)

在销毁 Line 类型时，会对 Line 对象调用 VWT 中的 destruct 方法，这将减少可能包含在我们的类型中的值的任何引用计数。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-20.png?raw=true)

然后在最后， Swift 调用该表中的 deallocate 函数。同样，我们有一个 Line 的值见证表，所以这将取消分配在堆上为我们的值分配的内存。

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-21.png?raw=true)

那么值类型是如何找到 VWT 和 PWT 的呢？答案是这两张表的指针存储在了 Existential Container 中：

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-22.png?raw=true)
![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-23.png?raw=true)

对于超出 valueBuffer 的值，每次 copy 操作都会赋值整个结构，引起性能的损耗。对于这种情况，赋值时会对引用类型只增加引用计数。这时你可能会说，这会引起非计划内的状态共享。于是 Swift 使用了 copy-on-write 计数。比如有两个值类型：

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-24.png?raw=true)

在对其中的一个的属性发生更改时，才会真正的触发 copy 操作：

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E7%90%86%E8%A7%A3%20Swift%20%E6%80%A7%E8%83%BD/images/1-25.png?raw=true)

实现原理就是，通过检查当前赋值对象的引用计数是否为大于1，来决定是否触发 copy-on-write

```
class LineStorage { var x1, y1,  x2, y2: Double }

struct Line : Drawable {
    var storage : LineStorage
    init() { storage = LineStorage(Point(), Point()) }
    func draw() { ... }
    mutating func move() {
        if !isUniquelyReferencedNonObjc(&storage) {
            storage = LineStorage(storage)
        }
        storage.start = ... 
    } 
}
```

## 泛型

对于泛型方面，主要的性能优化点在泛型特殊化，这在2015年的 WWDC 中已经介绍过了，就不在赘述了。感兴趣的可以看这篇文章：[优化 Swift 性能](https://github.com/yangxiaoju/Blogs/blob/master/iOS/Swift/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD/%E4%BC%98%E5%8C%96%20Swift%20%E6%80%A7%E8%83%BD.md)

## 参考文献：

> [Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/)