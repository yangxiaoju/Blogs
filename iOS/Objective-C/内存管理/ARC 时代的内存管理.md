# ARC 时代的内存管理

自动引用计数（ARC）是一种编译器功能，它提供 Objective-C 对象的自动内存管理。ARC 不是必须考虑 retain 和 release 操作，而是让你专注于有趣的代码，对象图以及应用程序中对象之间的关系。

![](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/RN-TransitioningToARC/Art/ARC_Illustration.jpg)

## 摘要

ARC 的工作原理是在编译时添加代码，以确保对象在必要时存活，但不会一直存活。从概念上讲，它通过为你添加适当的内存管理调用，遵循与手动引用计数相同的内存管理约定。

为了让编译器生成正确的代码，ARC 限制了你可以使用的方法以及如何使用免费桥接(toll-free bridging)。ARC 还为对象引用和声明的属性引入了新的生命周期限定符。

ARC 在 Xcode 4.2 编译的 OS X v10.6 和 v10.7 (64位应用程序) 以及 iOS 4 和 iOS 5 应用程序中提供支持。OS X v10.6 和 iOS 4 不支持弱引用。

Xcode 提供了一个工具，可以自动执行 ARC 转换的机械部分（例如删除 retain 和 release 调用），并帮助你修复迁移器无法自动处理的问题（选择 Edit > Refactor > Convert to Objective-C ARC）。迁移工具将项目中的所有文件转换为使用 ARC。如果对某些文件使用手动引用计数更方便，也可以选择在每个文件的基础上使用 ARC。

## ARC 概述

ARC 不会记住何时使用 retain，release 和 autorelease，而是评估对象的生命周期要求，并在编译时自动为你插入适当的内存管理调用。编译器还会为你生成适当的 dealloc 方法。通常，如果你只使用 ARC，则只有在需要与使用手动引用计数的代码进行互操作时，传统的 Cocoa 命名约定才是重要的。

Person 类的完整且正确的实现可能如下所示：

```objc
@interface Person : NSObject
@property NSString *firstName;
@property NSString *lastName;
@property NSNumber *yearOfBirth;
@property Person *spouse;
@end
 
@implementation Person
@end
```

（默认情况下，对象属性是 strong。）

使用 ARC，你可以实现这样一种人为的方法：

```objc
- (void)contrived {
    Person *aPerson = [[Person alloc] init];
    [aPerson setFirstName:@"William"];
    [aPerson setLastName:@"Dudney"];
    [aPerson setYearOfBirth:[[NSNumber alloc] initWithInteger:2011]];
    NSLog(@"aPerson: %@", aPerson);
}
```

ARC 负责内存管理，以便 Person 和 NSNumber 对象都不会泄露。

你还可以安全地实现 Person 的 takeLastNameFrom: 方法，如下所示：

```objc
- (void)takeLastNameFrom:(Person *)person {
    NSString *oldLastname = [self lastName];
    [self setLastName:[person lastName]];
    NSLog(@"Lastname changed from %@ to %@", oldLastname, [self lastName]);
}
```

ARC 确保在 NSLog 语句之前不释放 oldLastName。

### ARC 实施新规则

为了工作，ARC 强制使用其他编译器模式时不存在的一些新规则。规则旨在提供完全可靠的内存管理模型; 在某些情况下，他们只是执行最佳实践，在其他一些情况下，他们简化了代码，或者是你不必处理内存管理的明显推论。如果违反这些规则，则会立即得到编译时错误，而不是在运行时可能会出现的细微错误。

- 你无法显式调用 dealloc，也无法实现或调用 retain，release，retainCount 或 autorelease。

  禁止扩展到使用 @selector(retain)，@selector(release) 等。

  如果需要管理除释放实例变量之外的资源，则可以实现 dealloc 方法。你不必（实际上你不能）释放实例变量，但你可能需要在系统类和未使用 ARC 编译的其他代码上调用 [systemClassInstance setDelegate:nil]。

  ARC 中的自定义 dealloc 方法不需要调用 [super dealloc]（它实际上会导致编译器错误）。链接到 super 是由编译器自动执行的。

  你仍然可以将 CFRetain，CFRelease 和其他相关功能与 Core Foundation 样式的对象一起使用。

- 你不能使用 NSAllocateObject 或 NSDeallocateObject。

  你可以使用 alloc 创建对象; 运行时负责 deallocating 对象。

- 你不能在 C 结构中使用对象指针。

  你可以创建一个 Objective-C 类来管理数据，而不是使用结构。

- id 和 void * 之间不是简单的映射。

  你必须使用特殊的强制转换来告诉编译器有关对象的生命周期。你需要在 Objective-C 对象和作为函数参数传递的 Core Foundation 类型之间进行转换。

- 你不能使用 NSAutoreleasePool 对象。

  ARC 提供了 @autoreleasepool 块。它们具有比 NSAutoreleasePool 更有效的优点。

- 你不能使用内存区域(memory zones)。

  不再需要使用 NSZone - 无论如何它们都会被现代的 Objective-C 运行时忽略。

为了允许与 manual retain-release 代码进行互操作，ARC 对方法命名施加了约束：

- 你不能为访问者提供以 new 开头的名称。这反过来意味着你不能，例如，声明一个名称以 new 开头的属性，除非你指定一个不同的 getter：

```objc
// Won't work:
@property NSString *newTitle;
 
// Works:
@property (getter=theNewTitle) NSString *newTitle;
```

### ARC 推出新的生命周期限定符

ARC 为对象和弱引用引入了几个新的生命周期限定符。弱引用不会延长它指向的对象的生命周期，并且在没有对该对象的强引用时自动变为 nil。

你应该利用这些限定符来管理程序中的对象图。特别是，ARC不能防范强引用循环（以前称为 retain cycles）。明智地使用弱饮用将有助于确保你不会创建循环引用。

#### 属性特性

关键字 weak 和 strong 将作为新声明的属性特性引入，如以下示例所示。

```objc
// The following declaration is a synonym for: @property(retain) MyClass *myObject;
@property(strong) MyClass *myObject;
 
// The following declaration is similar to "@property(assign) MyClass *myObject;"
// except that if the MyClass instance is deallocated,
// the property value is set to nil instead of remaining as a dangling pointer.
@property(weak) MyClass *myObject;
```

在 ARC 下，strong 是对象类型的默认值。

#### 变量限定符

你可以像使用 const 一样为变量使用以下生命周期限定符。

```objc
__strong
__weak
__unsafe_unretained
__autoreleasing
```

- __strong 是默认值。只要存在指向它的强指针，对象就会保持“活着”。
- __weak 指定不使引用对象保持活动状态的引用。当没有对象的强引用时，弱引用设置为 nil。
- __unsafe_unretained 指定一个引用，该引用不会使引用的对象保持活动状态，并且在没有对该对象的强引用时不会设置为 nil。如果它引用的对象被释放，则指针悬空(dangling)。
- __autoreleasing 用于表示通过引用（id *）传递的参数，并在返回时自动释放。

你应该正确装饰变量。在对象变量声明中使用限定符时，正确的格式为：

```objc
ClassName * qualifier variableName;
```

例如：

```objc
MyClass * __weak myWeakReference;
MyClass * __unsafe_unretained myUnsafeReference;
```

其他变体在技术上是不正确的，但编译器“原谅”了。

在栈上使用 __weak 变量时要小心。请考虑以下示例：

```objc
NSString * __weak string = [[NSString alloc] initWithFormat:@"First Name: %@", [self firstName]];
NSLog(@"string: %@", string);
```

尽管在初始赋值之后使用了字符串，但在赋值时没有对字符串对象的其他强引用; 因此，它立即被释放。log 语句显示 string 的值为 null。（在这种情况下，编译器会提供警告。）

你还需要注意通过引用传递的对象。以下代码将起作用：

```objc
NSError *error;
BOOL OK = [myObject performOperationWithError:&error];
if (!OK) {
    // Report the error.
    // ...
```

但是，错误声明是隐式的：

```objc
NSError * __strong e;
```

方法声明通常是：

```objc
-(BOOL)performOperationWithError:(NSError * __autoreleasing *)error;
```

因此编译器重写代码：

```objc
NSError * __strong error;
NSError * __autoreleasing tmp = error;
BOOL OK = [myObject performOperationWithError:&tmp];
error = tmp;
if (!OK) {
    // Report the error.
    // ...
```

局部变量声明（__strong）和参数（__autoreleasing）之间的不匹配会导致编译器创建临时变量。当你获取 __strong 变量的地址时，可以通过声明参数 id __strong * 来获取原始指针。或者，你可以将变量声明为 __autoreleasing。

#### 使用生命周期限定符来避免强引用循环

你可以使用生命周期限定符来避免强引用循环。例如，通常如果你有一个以父子层次结构排列的对象图表，并且父对象需要引用他们的孩子，反之亦然，那么你将使父对象指向子对象之间的引用为强引用，并且父对象与子对象之间的引用为弱引用。其他情况可能更微妙，特别是当它们涉及 block 对象时。

在手动引用计数模式下，__block id x; 具有不保留 x 的效果。在 ARC 模式下，__block id x; 默认为保留 x（就像所有其他值一样）。要在 ARC 下获得手动引用计数模式行为，可以使用 __unsafe_unretained __block id x;。然而，正如名称 __unsafe_unretained 所暗示的那样，具有非保留变量是危险的（因为它可以悬挂），因此不鼓励。两个更好的选择是使用 __weak（如果你不需要支持 iOS 4 或 OS X v10.6），或者将 __block 值设置为 nil 以打破引用循环。

以下代码片段使用有时在手动引用计数中使用的模式说明了此问题。

```objc
MyViewController *myController = [[MyViewController alloc] init…];
// ...
myController.completionHandler =  ^(NSInteger result) {
   [myController dismissViewControllerAnimated:YES completion:nil];
};
[self presentViewController:myController animated:YES completion:^{
   [myController release];
}];
```

如上所述，你可以使用 __block 限定符并在完成处理程序中将 myController 变量设置为 nil：

```objc
MyViewController * __block myController = [[MyViewController alloc] init…];
// ...
myController.completionHandler =  ^(NSInteger result) {
    [myController dismissViewControllerAnimated:YES completion:nil];
    myController = nil;
};
```

或者，你可以使用临时 __weak 变量。以下示例说明了一个简单的实现：

```objc
MyViewController *myController = [[MyViewController alloc] init…];
// ...
MyViewController * __weak weakMyViewController = myController;
myController.completionHandler =  ^(NSInteger result) {
    [weakMyViewController dismissViewControllerAnimated:YES completion:nil];
};
```

但是，对于 non-trivial cycles，你应该使用：

```objc
MyViewController *myController = [[MyViewController alloc] init…];
// ...
MyViewController * __weak weakMyController = myController;
myController.completionHandler =  ^(NSInteger result) {
    MyViewController *strongMyController = weakMyController;
    if (strongMyController) {
        // ...
        [strongMyController dismissViewControllerAnimated:YES completion:nil];
        // ...
    }
    else {
        // Probably nothing...
    }
};
```

在某些情况下，如果类与 __weak 不兼容，则可以使用 __unsafe_unretained。然而，这对于 nontrivial cycles 来说变得不切实际，因为很难或不可能验证 __unsafe_unretained 指针仍然有效并且仍然指向相同的对象。

### ARC 使用新语句来管理自动释放池

使用 ARC，你无法使用 NSAutoreleasePool 类直接管理自动释放池。相反，你使用 @autoreleasepool 块：

```objc
@autoreleasepool {
     // Code, such as a loop that creates a large number of temporary objects.
}
```

这种简单的结构允许编译器推断引用计数状态。在进入时，推送自动释放池。在正常退出（break，return，goto，fall-through 等）时，将弹出自动释放池。为了与现有代码兼容，如果 exit 是由异常引起的，则不会弹出自动释放池。

所有 Objective-C 模式都提供此语法。它比使用 NSAutoreleasePool 类更有效; 因此，我们鼓励你采用它代替使用 NSAutoreleasePool。

### 管理 Outlets 的模式在平台上变得一致

用于在 iOS 和 OS X 中声明 outlets 的模式随 ARC 而变化，并且在两个平台上变得一致。你应该采用的模式是：outlets 应该是弱的，除了从文件所有者到 nib 文件（或故事板场景）中应该很强的顶级对象的出口。

### 栈变量初始化为 nil

使用 ARC，现在使用 nil 隐式初始化 strong，weak 和 autoreleasing 栈变量。例如：

```objc
- (void)myMethod {
    NSString *name;
    NSLog(@"name: %@", name);
}
```

将为 name 的值记录 null，而不是崩溃。

### 使用编译器标志启用和禁用 ARC

使用新的 -fobjc-arc 编译器标志启用 ARC。如果对某些文件使用手动引用计数更方便，也可以选择在每个文件的基础上使用 ARC。对于使用 ARC 作为默认方法的项目，可以使用该文件的新 -fno-objc-arc 编译器标志为特定文件禁用 ARC。

Xcode 4.2 及更高版本的 OS X v10.6 及更高版本（64位应用程序）以及 iOS 4 及更高版本支持 ARC。OS X v10.6 和 iOS 4 不支持弱引用。Xcode 4.1 及更早版本中没有 ARC 支持。

## 管理 Toll-Free Bridging

在许多 Cocoa 应用程序中，你需要使用 Core Foundation 样式的对象，无论是来自 Core Foundation 框架本身（例如 CFArrayRef 还是 CFMutableDictionaryRef），还是来自采用 Core Foundation 约定的框架（如 Core Graphics）（你可以使用类似 CGColorSpaceRef 和 CGGradientRef 的类型））。

编译器不会自动管理 Core Foundation 对象的生命周期; 你必须根据 Core Foundation 内存管理规则调用 CFRetain 和 CFRelease（或相应的特定于类型的变体）。

如果在 Objective-C 和 Core Foundation 风格的对象之间进行转换，则需要使用强制转换（在 objc/runtime.h 中定义）或 Core Foundation 样式宏（在 NSObject.h 中定义）来告诉编译器对象的所有权语义：

- __bridge 在 Objective-C 和 Core Foundation 之间传输指针，不转让所有权。
- __bridge_retained 或 CFBridgingRetain 将一个 Objective-C 指针强制转换为 Core Foundation 指针，并将所有权转移给你。

  你有责任调用 CFRelease 或相关函数来放弃对象的所有权。

- __bridge_transfer 或 CFBridgingRelease 将非 Objective-C 指针移动到 Objective-C，并将所有权转移到 ARC。

  ARC 负责放弃对象的所有权。

例如，如果你有这样的代码：

```objc
- (void)logFirstNameOfPerson:(ABRecordRef)person {
 
    NSString *name = (NSString *)ABRecordCopyValue(person, kABPersonFirstNameProperty);
    NSLog(@"Person's first name: %@", name);
    [name release];
}
```

你可以用以下代替它：

```objc
- (void)logFirstNameOfPerson:(ABRecordRef)person {
 
    NSString *name = (NSString *)CFBridgingRelease(ABRecordCopyValue(person, kABPersonFirstNameProperty));
    NSLog(@"Person's first name: %@", name);
}
```

### 编译器处理从 Cocoa 方法返回的 CF 对象

编译器理解返回 Core Foundation 类型的 Objective-C 方法遵循历史 Cocoa 命名约定。例如，编译器知道，在 iOS 中，UIColor 的 CGColor 方法返回的 CGColor 不归属。你仍必须使用适当的类型转换，如此示例所示：

```objc
NSMutableArray *colors = [NSMutableArray arrayWithObject:(id)[[UIColor darkGrayColor] CGColor]];
[colors addObject:(id)[[UIColor lightGrayColor] CGColor]];
```

### 使用所有权关键字转换函数参数

在函数调用中在 Objective-C 和 Core Foundation 对象之间进行转换时，需要告诉编译器有关传递对象的所有权语义。Core Foundation 对象的所有权规则是 Core Foundation 内存管理规则中指定的规则; Objective-C 对象的规则在 Advanced Memory Management Programming Guide 中指定。

在下面的代码片段中，传递给 CGGradientCreateWithColors 函数的数组需要适当的强制转换。arrayWithObjects: 返回的对象的所有权不会传递给函数，因此强制转换为 __bridge。

```objc
NSArray *colors = <#An array of colors#>;
CGGradientRef gradient = CGGradientCreateWithColors(colorSpace, (__bridge CFArrayRef)colors, locations);
```

代码片段在以下方法实现中的上下文中示出。另请注意 Core Foundation 内存管理功能的使用，这些功能由 Core Foundation 内存管理规则决定。

```objc
- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceGray();
    CGFloat locations[2] = {0.0, 1.0};
    NSMutableArray *colors = [NSMutableArray arrayWithObject:(id)[[UIColor darkGrayColor] CGColor]];
    [colors addObject:(id)[[UIColor lightGrayColor] CGColor]];
    CGGradientRef gradient = CGGradientCreateWithColors(colorSpace, (__bridge CFArrayRef)colors, locations);
    CGColorSpaceRelease(colorSpace);  // Release owned Core Foundation object.
    CGPoint startPoint = CGPointMake(0.0, 0.0);
    CGPoint endPoint = CGPointMake(CGRectGetMaxX(self.bounds), CGRectGetMaxY(self.bounds));
    CGContextDrawLinearGradient(ctx, gradient, startPoint, endPoint,
                                kCGGradientDrawsBeforeStartLocation | kCGGradientDrawsAfterEndLocation);
    CGGradientRelease(gradient);  // Release owned Core Foundation object.
}
```

## 转换项目时的常见问题

迁移现有项目时，你可能会遇到各种问题。以下是一些常见问题以及解决方案。

- 你无法调用 retain，release 或 autorelease。

  这是一个功能。你也写不出来：

  ```objc
  while ([x retainCount]) { [x release]; }
  ```

- 你不能调用 dealloc。

  如果要在 init 方法中实现单例或替换对象，通常会调用 dealloc。对于单例，请使用共享实例模式。在 init 方法中，你不必再调用 dealloc，因为在覆盖 self 时将释放该对象。

- 你不能使用 NSAutoreleasePool 对象。

  请改用新的 @autoreleasepool {} 结构。这会强制你的自动释放池上的块结构，并且比 NSAutoreleasePool 快约六倍。@autoreleasepool 甚至适用于非 ARC 代码。因为 @autoreleasepool 比 NSAutoreleasePool 快得多，所以许多旧的“性能黑客”可以简单地用无条件的 @autoreleasepool 替换。

  迁移器处理 NSAutoreleasePool 的简单用法，但它无法处理复杂的条件情况，或者在新的 @autoreleasepool 的主体内定义变量并在其后使用的情况。

- ARC 要求你在 init 方法中将 [super init] 的结果分配给 self。

  以下在 ARC init 方法中无效：

  ```objc
  [super init];
  ```

  简单的解决方法是将其更改为：

  ```objc
  self = [super init];
  ```

  正确的解决方法是这样做，并在继续之前检查结果为 nil：

  ```objc
  self = [super init];
  if (self) {
     ...
   ```

- 你无法实现自定义 retain 或 release 方法。

  实现自定义 retain 或 release 方法会打破弱指针。想要提供自定义实现有几个常见原因：

  - 性能。

    请不要再这样做了; NSObject 的 retain 和 release 的实现现在要快得多。如果你仍然发现问题，请提交错误。

  - 实现自定义弱指针系统。

    请改用 __weak。

  - 实现单例类。

    请改用共享实例模式。或者，使用类而不是实例方法，这样就不必分配对象了。

- “Assigned”实例变量变得 strong。

  在 ARC 之前，实例变量是非拥有引用 - 直接将对象分配给实例变量并不延长对象的生命周期。为了使属性变强，你通常会实现或合成调用适当内存管理方法的访问器方法; 相反，你可能已经实现了以下示例中显示的访问器方法来维护弱属性。

  ```objc
  @interface MyClass : Superclass {
      id thing; // Weak reference.
  }
  // ...
  @end
 
  @implementation MyClass
  - (id)thing {
      return thing;
  }
  - (void)setThing:(id)newThing {
      thing = newThing;
  }
  // ...
  @end
  ```

  对于 ARC，实例变量默认是强引用 - 将对象直接分配给实例变量会延长对象的生命周期。迁移工具无法确定实例变量何时弱。 要保持与以前相同的行为，必须将实例变量标记为弱，或使用声明的属性。

  ```objc
  @interface MyClass : Superclass {
      id __weak thing;
  }
  // ...
  @end
 
  @implementation MyClass
  - (id)thing {
      return thing;
  }
  - (void)setThing:(id)newThing {
      thing = newThing;
  }
  // ...
  @end
  ```

  或：

  ```objc
  @interface MyClass : Superclass
  @property (weak) id thing;
  // ...
  @end
 
  @implementation MyClass
  @synthesize thing;
  // ...
  @end
  ```

- 你不能在 C 结构中使用 strong ids。

  例如，以下代码将无法编译：

  ```objc
  struct X { id x; float y; };
  ```

  这是因为 x 默认 strong 引用，编译器无法安全地合成使其正常工作所需的所有代码。例如，如果通过一些最终执行自由的代码将指针传递给其中一个结构，则必须在释放结构之前释放每个 id。编译器无法可靠地执行此操作，因此在 ARC 模式下不允许完全禁用结构中的 strong ids。有几种可能的解决方案：

  1. 使用 Objective-C 对象而不是结构。

     无论如何，这被认为是最佳实践。

  2. 如果使用 Objective-C 对象是次优的（可能你需要这些结构的密集数组），那么请考虑使用 void *。

     这需要使用下面描述的显式强制转换。

  3. 将对象引用标记为 __unsafe_unretained。

     这种方法对于这样的半常见模式可能很有用：

  ```objc
  struct x { NSString *S;  int X; } StaticArray[] = {
    @"foo", 42,
    @"bar, 97,
  ...
  };
  ```

  你将结构声明为：

  ```objc
  struct x { NSString * __unsafe_unretained S; int X; }
  ```

  如果对象可以从指针下面释放出来，这可能是有问题的并且是不安全的，但是对于已知永远像常量字符串文字一样的事物来说它非常有用。

- 你不能直接在 id 和 void * 之间进行转换（包括 Core Foundation 类型）。

  这在 Managing Toll-Free Bridging 中有更详细的讨论。

## 经常问的问题

我如何看待 ARC？retains/releases 的位置在哪里？

尝试不再考虑 retain/release 调用的位置，而是考虑应用程序算法。考虑对象中的“强弱”指针，对象所有权以及可能的引用循环。

我还需要为我的对象编写 dealloc 方法吗？

也许。

由于 ARC 不会自动执行 malloc/free，管理 Core Foundation 对象，文件描述符等的生命周期，因此你仍然可以通过编写 dealloc 方法来释放这些资源。

你不必（实际上不能）释放实例变量，但你可能需要在系统类和其他未使用 ARC 编译的代码上调用 [self setDelegate:nil]。

ARC 中的 dealloc 方法不需要 - 或允许调用[super dealloc]; 链接到父类由运行时处理和强制执行。

ARC 中仍然有引用循环吗？

是。

ARC 自动 retain/release，并继承引用循环的问题。幸运的是，迁移到 ARC 的代码很少开始泄漏，因为属性已经声明属性是否 retaining。

blocks 如何在 ARC 中工作？

在 ARC 模式下向块传递块时块“正常工作”，例如在返回中。你不必再调用 Block Copy。

需要注意的一件事是 NSString * __block myString 保留在 ARC 模式下，而不是可能悬空的指针。要获取先前的行为，请使用 __block NSString * __unsafe_unretained myString 或（更好的是）使用 __block NSString * __weak myString。

我可以使用 Snow Leopard 为 ARC 开发 OS X 应用程序吗？

不可以。Xcode 4.2 的 Snow Leopard 版本在 OS X 上根本不支持 ARC，因为它不包括 10.7 SDK。Snow Leopard 的 Xcode 4.2 确实支持 ARC for iOS，而 Xcode 4.2 for Lion 支持 OS X 和 iOS。这意味着你需要 Lion 系统来构建在 Snow Leopard 上运行的 ARC 应用程序。

我可以在 ARC 下创建一个 retained 指针的C数组吗？

是的，你可以，如此示例所示：

```objc
// Note calloc() to get zero-filled memory.
__strong SomeClass **dynamicArray = (__strong SomeClass **)calloc(entries, sizeof(SomeClass *));
for (int i = 0; i < entries; i++) {
     dynamicArray[i] = [[SomeClass alloc] init];
}
 
// When you're done, set each entry to nil to tell ARC to release the object.
for (int i = 0; i < entries; i++) {
     dynamicArray[i] = nil;
}
free(dynamicArray);
```

有许多方面需要注意：

- 在某些情况下，你需要编写 __strong SomeClass **，因为默认值是 __autoreleasing SomeClass **。
- 分配的内存必须为零填充。
- 在释放数组之前，必须将每个元素设置为 nil（memset 或 bzero 将不起作用）。
- 你应该避免使用 memcpy 或 realloc。

ARC 慢吗？

这取决于你正在测量什么，但通常是“不”。编译器有效地消除了许多无关的 retain/release 调用，并且已经投入了大量精力来加速 Objective-C 运行时。特别是，当方法的调用者是 ARC 代码时，常见的“返回 retain/autoreleased 对象”模式要快得多，并且实际上并不将对象放入自动释放池中。

需要注意的一个问题是优化器不在常见的调试配置中运行，因此期望在-O0处看到比在-Os更多的 retain/release 流量。

ARC 是否在 ObjC++ 模式下工作？

是。你甚至可以在类和容器中放置 strong/weak idsD。ARC 编译器在复制构造函数和析构函数等中合成 retain/release 逻辑以使其工作。

哪些类不支持弱引用？

你当前无法创建对以下类的实例的弱引用：

NSATSTypesetter，NSColorSpace，NSFont，NSMenuView，NSParagraphStyle，NSSimpleHorizontalTypesetter 和 NSTextView。

> 注意：此外，在 OS X v10.7 中，你无法创建对 NSFontManager，NSFontPanel，NSImage，NSTableCellView，NSViewController，NSWindow 和 NSWindowController 实例的弱引用。此外，在 OS X v10.7中，AV Foundation 框架中的任何类都不支持弱引用。

对于声明的属性，你应该使用 assign 而不是 weak; 对于变量，你应该使用 __unsafe_unretained 而不是 __weak。

此外，你无法在 ARC 下从 NSHashTable，NSMapTable 或 NSPointerArray 的实例创建弱引用。

在对 NSCell 或使用 NSCopyObject 的其他类进行子类化时，我该怎么办？

没什么特别的。ARC 负责处理你必须明确添加额外 retains 的情况。使用 ARC，所有复制方法都应该只复制实例变量。

我可以选择退出 ARC 以获取特定文件吗？

可以。

迁移项目以使用 ARC 时，-fobjc-arc 编译器标志将设置为所有 Objective-C 源文件的缺省值。你可以使用该类的 -fno-objc-arc 编译器标志为特定类禁用 ARC。在 Xcode 中，在目标 Build Phases 选项卡中，打开 Compile Sources 组以显示源文件列表。双击要为其设置标志的文件，在弹出式面板中输入 -fno-objc-arc，然后单击“完成”。

![](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/RN-TransitioningToARC/Art/fno-objc-arc.png)

是否在 Mac 上弃用了GC（垃圾收集）？

OS X Mountain Lion v10.8 中不推荐使用垃圾收集，并且将在 OS X 的未来版本中删除垃圾收集。自动引用计数是推荐的替代技术。为了帮助迁移现有应用程序，Xcode 4.3 及更高版本中的 ARC 迁移工具支持将垃圾收集的 OS X 应用程序迁移到 ARC。

> 注意：对于面向 Mac App Store 的应用，Apple 强烈建议你尽快使用 ARC 替换垃圾回收，因为 Mac App Store 指南禁止使用已弃用的技术。