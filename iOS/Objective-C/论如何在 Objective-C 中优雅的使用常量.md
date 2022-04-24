# 论如何在 Objective-C 中优雅的使用常量

在编写代码时经常要使用常量，来替代 magic number。比较简单的做法是通过预处理指令 `#define` 来实现。

```objc
#define ANIMATION_DURATION 0.3 
```

上述预处理指令会在编译时的预处理阶段会将代码中 `ANIMATION_DURATION` 字符串替换为 `0.3`。这种定义常量的方式比较简便，但是存在两个问题：

1. 丢失了类型信息。
2. 若该预处理指令声明在头文件中，引入该头文件的代码，`ANIMATION_DURATION` 都会被替换，可能出现冲突。

## Objective-C 的常量声明方式

幸运的是，`Objective-C` 中提供了 `const` 关键字，可以用来定义常量。`const` 关键字可以对变量加以限定，使其值不能被改变，在整个作用域中都保持固定。

```objc
const NSTimeInterval kAnimationDuration = 0.3;
```

这种方式定义的常量包含类型信息，且在编译时即可检查是否与其他常量出现冲突。如果试图修改由 `const` 修饰符所声明的变量，那么编译器就会报错。

如果常量仅在某个实现文件中使用，还应该加上 `static` 关键字，否则会被视为全局常量。若不使用 `static`，编译器会为它创建一个外部符号，若另一个编译单元中也声明了同名变量，就会报错。

```objc
static const NSTimeInterval kAnimationDuration = 0.3;
```

当一个变量同时使用了 `static` 和 `const`，那么编译器并不会创建符号，而是会像 `#define` 预处理指令一样，把所有遇到的变量替换为常值。

有时候需要把一个常量暴露给外界使用，比如通知，此类常量需放在全局符号表中。可以使用 `extern` 关键字，在头文件中进行声明：

```objc
// .h
extern NSString * const AFNetworkingTaskDidResumeNotification;
// .m
NSString * const AFNetworkingTaskDidResumeNotification = @"com.alamofire.networking.task.resume";
```

该常量在头文件中声明，在实现文件中定义。需要注意的是 `const` 写在指针类型的右边意味着该指针的指向不可被改变，若写在左边意味着该指针指向的内容不可被改变。

按上述方式实现并定义后，在编译时生成目标文件时，编译器会在数据段为字符串分配存储空间。

在 `Foundation` 框架中，苹果为了兼容 `C++` 中对 `extern` 的使用，提供了宏：

```objc
#if defined(__cplusplus)
#define FOUNDATION_EXTERN extern "C"
#else
#define FOUNDATION_EXTERN extern
#endif

#define FOUNDATION_EXPORT FOUNDATION_EXTERN
#define FOUNDATION_IMPORT FOUNDATION_EXTERN
```

一个 `C++` 程序中可能包含其他语言编写的部分代码，同样，`C++` 编写的代码片段也可能被用在其他语言编写的代码中。但是，不同语言编写的代码相互调用是困难的，更何况用同一种语言编写，使用不同编译器进行编译的情况。因为，不同语言或者同种语言在不同编译器上编译时，在注册变量，传递参数和参数在栈上的布局上可能存在差异。

为了使它们遵守统一规则，可以使用 `extern` 指定一个编译和链接规约。`extern "C"` 指令中的 `C`，表示的是一种编译和链接规约，而不是一种语言。`C` 表示符合 `C` 语言的编译和链接规约的任何语言。

还要说明的是，`extern "C"` 指令指定的编译和链接规约，不会影响语义，只是改变编译和链接的方式。

而 `FOUNDATION_EXPORT` 和 `FOUNDATION_IMPORT` 是用来兼容 `Win32` 应用程序的，移动端开发可以忽略。

所以上述对全局常量的声明，可以写成：

```objc
// .h
FOUNDATION_EXPORT NSString * const AFNetworkingTaskDidResumeNotification;
// .m
NSString * const AFNetworkingTaskDidResumeNotification = @"com.alamofire.networking.task.resume";
```

## 在 Objective-C 中使用 let 来声明常量

使用过 `Swift` 的同学，一定对其声明常量的方式的简洁性印象深刻，在 `Swift` 中声明常量的方式如下所示：

```swift
let kAnimationDuration = 0.3
```

之所以能如此简洁，是因为 `Swift` 具有 `let` 关键字和类型推断的能力，但其实在 `Objective-C` 中也可以通过类似的方式来书写常量。

`Objective-C` 中有一个关键字，是 `__auto_type`，可以实现类似 `Swift` 中类型推断能力的关键字，如下所示：

```objc
const __auto_type kAnimationDuration = 0.3;
```

可能对于简单的数据类型，这样的优势不是很明显，但是对于具有复杂泛型的类型来说，可以说优势很大了：

```objc
// 旧方式
NSArray<NSDictionary<NSString *, NSString *> *> *models = ...;
// 新方式
__auto_type models = ...;
```

同时，可以通过宏的方式，来减少 `__auto_type` 的书写，即可实现通过 `let` 声明常量，`var` 声明变量。

```objc
#if defined(__cplusplus)
#define let auto const
#else
#define let const __auto_type
#endif

#if defined(__cplusplus)
#define var auto
#else
#define var __auto_type
#endif
```

声明了上面的宏，就可以直接使用了：

```objc
let kAnimationDuration = 0.3;
```