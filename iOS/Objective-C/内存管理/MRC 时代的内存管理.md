# MRC 时代的内存管理

## 简介

应用程序内存管理是在程序运行时分配内存，使用它并在完成后释放内存的过程。编写良好的程序使用尽可能少的内存。在 Objective-C 中，它还可以被看作是在许多数据和代码之间分配有限内存资源的所有权的一种方式。内存管理的核心思想是：明确管理对象的生命周期，并在不再需要时释放它们。

虽然内存管理通常被视为单个对象的级别，但你的目标实际上是管理对象图。你希望确保内存中没有比实际需要的对象更多的对象。

![](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Art/memory_management_2x.png)

Objective-C 提供了两种应用程序内存管理方法。

1. “manual retain-release” 或 MRR，你通过跟踪你拥有的对象来明确管理内存。这是使用一个称为引用计数的模型实现的，Foundation 框架中的 NSObject 类与运行时环境一起提供。

2. 在自动引用计数或 ARC 中，系统使用与 MRR 相同的引用计数系统，但它在编译时为你插入适当的内存管理方法调用。如果你使用 ARC，通常不需要理解底层实现，尽管在某些情况下它可能会有所帮助。

### 良好实践可防止与内存相关的问题

这里有两种主要导致内存管理不正确的问题：

- 释放或覆盖仍在使用的数据

  这会导致内存损坏，并且通常会导致应用程序崩溃，甚至导致用户数据损坏。

- 不释放不再使用的数据会导致内存泄漏

  内存泄漏是指未释放已分配内存，即使它从未再次使用过。泄漏会导致应用程序不断增加的内存使用量，从而导致系统性能下降或应用程序被终止。

但是，从引用计数的角度考虑内存管理通常会适得其反，因为你倾向于根据实现细节而不是实际目标来考虑内存管理。相反，你应该从对象所有权和对象图的角度考虑内存管理。

Cocoa 使用简单的命名约定来指示你拥有方法返回的对象的时间。

虽然基本策略很简单，但你可以采取一些实际步骤来简化内存管理，并帮助确保你的程序保持可靠和健壮，同时最大限度地减少其资源需求。

自动释放池提供了一种机制，你可以通过该机制向对象发送“延迟(deferred)”释放消息。这在你想要放弃对象的所有权但希望避免立即释放它的可能性（例如从方法返回对象时）的情况下非常有用。有时你可能会使用自己的自动释放池。

### 使用分析工具调试内存问题

要在编译时识别代码问题，可以使用 Xcode 中内置的 Clang Static Analyzer。

如果仍然出现内存管理问题，你可以使用其他工具和技术来识别和诊断问题。

- 许多工具和技术都在 Technical Note TN2239，iOS Debugging Magic 中进行了描述，特别是使用 NSZombie 来帮助找到过度释放的对象。

- 你可以使用 Instruments 跟踪引用计数事件并查找内存泄漏。

## 内存管理策略

在引用计数环境中用于内存管理的基本模型由 NSObject 协议中定义的方法和标准方法命名约定的组合提供。NSObject 类还定义了一个方法 dealloc，该方法在对象销毁时自动调用。

### 基本内存管理规则

内存管理模型基于对象所有权。任何对象都可能拥有一个或多个所有者。只要一个对象至少有一个所有者，它就会继续存在。如果对象没有所有者，则运行时系统会自动销毁它。为了确保何时你拥有对象何时你不拥有对象的时机是清晰的，Cocoa 设置以下策略：

- 你拥有自己创建的任何对象

  使用名称以“alloc”，“new”，“copy”或“mutableCopy”开头的方法（例如，alloc，newObject 或 mutableCopy）创建对象。

- 你可以使用 retain 获取对象的所有权

  通常保证接收到的对象在接收到的方法中保持有效，并且该方法也可以安全地将对象返回给其调用者。在两种情况下使用 retain：（1）在 accessor 方法或 init 方法的实现中，获取要存储为对象属性的对象的所有权;（2）防止对象因某些其他操作的副作用而失效。

- 当你不再需要它时，你必须放弃你拥有的对象的所有权

  你通过向对象发送 release 消息或 autorelease 消息来放弃对象的所有权。因此，在 Cocoa 术语中，放弃对象的所有权通常被称为“释放(releasing)”对象。

- 你不得放弃你不拥有的对象的所有权

  这只是先前明确规定的策略规则的必然结果。

#### 一个简单的例子

要说明策略，请考虑以下代码片段：

```objc
{
    Person *aPerson = [[Person alloc] init];
    // ...
    NSString *name = aPerson.fullName;
    // ...
    [aPerson release];
}
```

Person 对象是使用 alloc 方法创建的，因此在不再需要时会发送一条释放消息。不使用任何拥有方法检索此人的姓名，因此不会发送 release 消息。但请注意，该示例使用的是 release 而不是 autorelease。

#### 使用 autorelease 发送延迟 release

当你需要发送延迟 release 消息时，通常在从方法返回对象时使用 autorelease。例如，你可以像这样实现 fullName 方法：

```objc
- (NSString *)fullName {
    NSString *string = [[[NSString alloc] initWithFormat:@"%@ %@",
                                          self.firstName, self.lastName] autorelease];
    return string;
}
```

你拥有 alloc 返回的字符串。要遵守内存管理规则，你必须在丢失对该字符串的引用之前放弃该字符串的所有权。但是，如果使用 release，则在返回之前将释放该字符串（并且该方法将返回无效对象）。使用 autorelease，表示你要放弃所有权，但允许方法的调用者在释放之前使用返回的字符串。

你还可以像这样实现 fullName 方法：

```objc
- (NSString *)fullName {
    NSString *string = [NSString stringWithFormat:@"%@ %@",
                                 self.firstName, self.lastName];
    return string;
}
```

遵循基本规则，你不拥有 stringWithFormat: 返回的字符串，因此你可以安全地从方法返回字符串。

相比之下，以下实现是错误的：

```objc
- (NSString *)fullName {
    NSString *string = [[NSString alloc] initWithFormat:@"%@ %@",
                                         self.firstName, self.lastName];
    return string;
}
```

根据命名约定，没有任何东西可以表示 fullName 方法的调用者拥有返回的字符串。因此调用者没有理由释放返回的字符串，因此它将被泄露。

#### 你不拥有通过引用返回的对象

Cocoa 中的一些方法指定通过引用返回一个对象（即，它们采用 ClassName ** 或 id * 类型的参数）。常见的模式是使用 NSError 对象，该对象包含有关错误的信息（如果发生），如 initWithContentsOfURL:options:error: (NSData) 和 initWithContentsOfFile:encoding:error: (NSString) 所示。

在这些情况下，适用的规则与已经描述的相同。当你调用这些方法中的任何一个时，你不会创建 NSError 对象，因此你不拥有它。因此无需释放它，如下例所示：

```objc
NSString *fileName = <#Get a file name#>;
NSError *error;
NSString *string = [[NSString alloc] initWithContentsOfFile:fileName
                        encoding:NSUTF8StringEncoding error:&error];
if (string == nil) {
    // Deal with error...
}
// ...
[string release];
```

### 实现 dealloc 以放弃对象的所有权

NSObject 类定义了一个方法 dealloc，该方法在对象没有所有者并且其内存被回收时自动调用 - 在 Cocoa 术语中它被“释放(freed)”或“解除分配(deallocated)”。dealloc 方法的作用是释放对象自己的内存，并释放它拥有的任何资源，包括任何对象实例变量的所有权。

```objc
@interface Person : NSObject
@property (retain) NSString *firstName;
@property (retain) NSString *lastName;
@property (assign, readonly) NSString *fullName;
@end
 
@implementation Person
// ...
- (void)dealloc
    [_firstName release];
    [_lastName release];
    [super dealloc];
}
@end
```

重要说明：永远不要直接调用另一个对象的 dealloc 方法。你必须在实现结束时调用超类的实现。你不应该将系统资源的管理与对象生命周期联系起来。当应用程序终止时，可能不会向对象发送 dealloc 消息。因为进程的内存在退出时自动清除，所以仅仅允许操作系统清理资源比调用所有内存管理方法更有效。

### Core Foundation 使用相似但不同的规则

Core Foundation 对象有类似的内存管理规则。但是，Cocoa 和 Core Foundation 的命名约定是不同的。特别是，Core Foundation 的 Create Rule 不适用于返回 Objective-C 对象的方法。例如，在以下代码片段中，你不负责放弃 myInstance 的所有权：

```objc
MyClass *myInstance = [MyClass createInstance];
```

## 实用的内存管理

虽然内存管理策略中描述的基本概念很简单，但你可以采取一些实际步骤来简化内存管理，并帮助确保你的程序保持可靠和健壮，同时最大限度地减少其资源需求。

### 使用访问器方法使内存管理更容易

如果你的类具有作为对象的属性，则必须确保在你使用它时不会释放任何设置为该值的对象。因此，你必须在设置对象时声明对象的所有权。你还必须确保放弃任何当前持有的值的所有权。

有时它可能看起来很乏味或迂腐，但如果你一直使用访问器方法，那么内存管理问题的可能性会大大降低。如果你在整个代码中使用实例变量的 retain 和 release，那么你几乎肯定会做错事。

考虑一个你想要设置其计数的 Counter 对象。

```objc
@interface Counter : NSObject
@property (nonatomic, retain) NSNumber *count;
@end;
```

该属性声明了两种访问方法。通常，你应该要求编译器合成方法; 但是，了解它们如何实现是有益的。

在“get”访问器中，你只需返回合成的实例变量，因此不需要 retain 或 release：

```objc
- (NSNumber *)count {
    return _count;
}
```

在“set”方法中，如果其他所有人都按照相同的规则进行游戏，则必须假设新计数可以随时处理，因此你必须取得对象的所有权 - 通过向其发送 retain 消息 - 以确保它不会被释放。你还必须通过向其发送 release 消息来放弃旧计数对象的所有权。（在 Objective-C 中允许向 nil 发送消息，因此如果尚未设置 _count，则实现仍然有效。）你必须在[newCount retain]之后发送此消息，以防两者是同一个对象 - 你不想无意中导致它被释放。

```objc
- (void)setCount:(NSNumber *)newCount {
    [newCount retain];
    [_count release];
    // Make the new assignment.
    _count = newCount;
}
```

#### 使用访问器方法设置属性值

假设你要实现重置计数器的方法。你有几个选择。第一个实现使用 alloc 创建 NSNumber 实例，因此你可以通过 release 将其进行平衡。

```objc
- (void)reset {
    NSNumber *zero = [[NSNumber alloc] initWithInteger:0];
    [self setCount:zero];
    [zero release];
}
```

第二个使用便捷构造方法来创建新的 NSNumber 对象。因此，不需要 retain 或 release 消息

```objc
- (void)reset {
    NSNumber *zero = [NSNumber numberWithInteger:0];
    [self setCount:zero];
}
```

请注意，两者都使用 set 访问器方法。

对于简单的情况，以下几乎肯定会正常工作，但是尽可能避免使用访问器方法，这样做几乎肯定会在某个阶段导致错误（例如，当你忘记 retain 或 release 时，或者如果实例变量的内存管理语义更改）。

```objc
- (void)reset {
    NSNumber *zero = [[NSNumber alloc] initWithInteger:0];
    [_count release];
    _count = zero;
}
```

另请注意，如果使用 key-value observing，则以这种方式更改变量不会触发 KVO。

#### 不要在初始化方法和 dealloc 中使用访问器方法

你不应该使用访问器方法来设置实例变量的唯一地方是初始化方法和 dealloc。要使用表示零的数字对象初始化计数器对象，可以按如下方式实现 init 方法：

```objc
- init {
    self = [super init];
    if (self) {
        _count = [[NSNumber alloc] initWithInteger:0];
    }
    return self;
}
```

要允许使用非零计数初始化计数器，你可以实现 initWithCount: 方法，如下所示：

```objc
- initWithCount:(NSNumber *)startingCount {
    self = [super init];
    if (self) {
        _count = [startingCount copy];
    }
    return self;
}
```

由于 Counter 类具有对象实例变量，因此还必须实现 dealloc 方法。它应该通过向它们发送 release 消息来放弃任何实例变量的所有权，并最终应该调用 super 的实现：

```objc
- (void)dealloc {
    [_count release];
    [super dealloc];
}
```

### 使用弱引用来避免循环 Retain

Retaining 对象会创建对该对象的强引用。在 released 所有强引用之前，不能释放对象。因此，如果两个对象可能具有循环引用，则会出现一个被称为循环 retain 的问题 - 也就是说，它们彼此之间具有强引用（直接或通过一系列其他对象，每个对象都强引用下一个对象直到回到第一个）。

图1中所示的对象关系说明了潜在的循环 retain。Document 对象具有文档中每个页面的 Page 对象。每个 Page 对象都有一个属性，用于跟踪它所在的文档。如果 Document 对象具有对 Page 对象的强引用，并且 Page 对象具有对 Document 对象的强引用，则任何对象都不能被释放。在释放 Page 对象之前，Document 的引用计数不能为零，并且在取消分配 Document 对象之前不会释放 Page 对象。

图 1 循环引用的图示

![](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Art/retaincycles_2x.png)

循环 retain 问题的解决方案是使用弱引用。弱引用是非拥有关系，其中源对象不保留它具有引用的对象。

但是，为了保持对象图完好无损，必须在某处有强引用（如果只有弱引用，则页面和段落可能没有任何所有者，因此将被释放）。因此，Cocoa 建立了一个约定，即父对象应该对其孩子保持强引用，并且孩子们应该有对父对象的弱引用。

因此，在图1中，文档对象具有对（retains）其页面对象的强引用，但页面对象具有对（not retain）文档对象的弱引用。

Cocoa 中弱引用的示例包括但不限于 table data sources，outline view items，notification observers 以及其他 targets 和 delegates。

你需要注意将消息发送到仅包含弱引用的对象。如果在销毁对象后向对象发送消息，则应用程序将崩溃。对象有效时，你必须具有明确定义的条件。在大多数情况下，弱引用对象知道另一个对象对它的弱引用，就像循环引用的情况一样，并且负责在销毁时时通知另一个对象。例如，当你向通知中心注册对象时，通知中心会存储对该对象的弱引用，并在发布相应的通知时向其发送消息。销毁对象后，你需要将其注销到通知中心，以防止通知中心向该对象发送任何不再存在的消息。同样，当销毁 delegate 对象时，你需要通过向另一个对象发送带有 nil 参数的 setDelegate: 消息来删除 delegate 引用。这些消息通常从对象的 dealloc 方法发送。

### 避免导致你正在使用的对象被销毁

Cocoa 的所有权策略指定接收的对象通常应该在调用方法的整个范围内保持有效。还应该可以从当前范围返回接收到的对象而不用担心它被释放。对于你的应用程序而言，对象的 getter 方法返回缓存的实例变量或计算值应该无关紧要。重要的是该对象在你需要的时间内仍然有效。

此规则偶尔会有例外情况，主要分为两类。

1. 从一个基本集合类中删除对象时。

```objc
heisenObject = [array objectAtIndex:n];
[array removeObjectAtIndex:n];
// heisenObject could now be invalid.
```

从其中一个基本集合类中删除对象时，会向其发送一个 release（而不是 autorelease）消息。如果集合是已删除对象的唯一所有者，则会立即释放已删除的对象（示例中为 heisenObject）。

2. 当“父对象”被释放时。

```objc
id parent = <#create a parent object#>;
// ...
heisenObject = [parent child] ;
[parent release]; // Or, for example: self.parent = nil;
// heisenObject could now be invalid.
```

在某些情况下，你从另一个对象检索对象，然后直接或间接释放父对象。如果释放父对象导致它被释放，并且父对象是子对象的唯一所有者，则子对象（示例中的 heisenObject）将同时被释放（假设它被发送一个 release 而不是一个 autorelease 消息在父对象的 dealloc 方法中）。

为了防止这些情况，你在收到 heisenObject 后会保留它，并在完成后将其释放。例如：

```objc
heisenObject = [[array objectAtIndex:n] retain];
[array removeObjectAtIndex:n];
// Use heisenObject...
[heisenObject release];
```

### 不要使用 dealloc 来管理稀缺资源

你通常不应该在 dealloc 方法中管理稀缺资源，例如文件描述符，网络连接以及缓冲区或缓存。特别是，你不应该设计类，以便在你认为将调用 dealloc 时调用 dealloc。由于错误或应用程序拆除(tear-down)，dealloc 的调用可能会被延迟或回避。

相反，如果你有一个实例管理稀缺资源的类，你应该设计你的应用程序，以便你知道何时不再需要资源，然后可以告诉实例在那时“清理”。你通常会释放该实例，dealloc 会跟随，但如果没有，你将不会遇到其他问题。

如果你尝试在 dealloc 之上捎带资源管理，则可能会出现问题。例如：

1. 对象图拆除的顺序依赖性。

   对象图拆除机制本质上是无序的。虽然你通常会期望并获得特定顺序，但你会引入脆弱性。如果对象意外地自动释放而不是正常释放，则拆卸顺序可能会改变，这可能会导致意外结果。

2. 不回收稀缺资源。

   内存泄漏是应该修复的错误，但它们通常不会立即致命。但是，如果在你希望释放资源时没有释放稀缺资源，则可能会遇到更严重的问题。例如，如果你的应用程序用完了文件描述符，则用户可能无法保存数据。

3. 在错误的线程上执行清理逻辑。

   如果一个对象在意外的时间自动释放，它将在它碰巧进入的任何线程的自动释放池块中被释放。对于只能从一个线程触及的资源来说，这很容易致命。

### 集合拥有它们包含的对象

将对象添加到集合（例如数组，字典或集合）时，集合将获得对象的所有权。当从集合中移除对象或集合本身被释放时，集合将放弃所有权。因此，例如，如果要创建数字数组，可以执行以下任一操作：

```objc
NSMutableArray *array = <#Get a mutable array#>;
NSUInteger i;
// ...
for (i = 0; i < 10; i++) {
    NSNumber *convenienceNumber = [NSNumber numberWithInteger:i];
    [array addObject:convenienceNumber];
}
```

在这种情况下，你没有调用 alloc，因此无需调用 release。不需要保留新数字（convenienceNumber），因为数组会这样做。

```objc
NSMutableArray *array = <#Get a mutable array#>;
NSUInteger i;
// ...
for (i = 0; i < 10; i++) {
    NSNumber *allocedNumber = [[NSNumber alloc] initWithInteger:i];
    [array addObject:allocedNumber];
    [allocedNumber release];
}
```

在这种情况下，你需要在 for 循环的范围内发送 allocedNumber 释放消息以平衡 alloc。由于数组在 addObject: 添加时 retained 了数字，因此在数组中它不会被释放。

要理解这一点，请将自己置于实现集合类的人的位置。你希望确保没有给予你照顾的对象从你的下方消失，因此你在传入时向他们发送 retain 消息。如果它们被删除，你必须发送 release 消息以保持平衡，在你自己的 dealloc 方法中，应该向任何剩余的对象发送一条 release 消息。

### 使用 Retain Counts 实现所有权政策

所有权策略通过引用计数实现 - 通常在 retain 方法之后称为“retain count”。每个对象都有一个 retain count。

- 创建对象时，其 retain count 为1。
- 向对象发送 retain 消息时，其 retain count 将增加1。
- 向对象发送 release 消息时，其 retain count 减1。

  当你向对象发送 autorelease 消息时，其 retain count 在当前自动释放池的末尾递减1。

- 如果对象的 retain count 减少到零，则将其销毁。

重要提示：应该没有理由明确询问对象的 retain count 是什么。结果通常会产生误导，因为你可能不知道哪些框架对象 retained 了你感兴趣的对象。在调试内存管理问题时，你应该只关心确保代码遵守所有权规则。

## 使用 Autorelease Pool Blocks

自动释放池块提供了一种机制，你可以放弃对象的所有权，但避免立即释放它（例如从方法返回对象时）。通常，你不需要创建自己的自动释放池块，但在某些情况下，你必须或者这样做是有益的。

### 关于 Autorelease Pool Blocks

使用 @autoreleasepool 标记 autorelease pool block，如以下示例所示：

```objc
@autoreleasepool {
    // Code that creates autoreleased objects.
}
```

在 autorelease pool block 的末尾，在块中接收到 autorelease 消息的对象被发送 release 消息 - 对象在每次在块内发送 autorelease 消息时接收 release 消息。

与任何其他代码块一样，autorelease pool blocks 可以嵌套：

```objc
@autoreleasepool {
    // . . .
    @autoreleasepool {
        // . . .
    }
    . . .
}
```

（你通常不会完全按照上面的方式看到代码;通常，一个源文件中的 autorelease pool block 中的代码将调用另一个源文件中的包含在 autorelease pool block 中的代码。）对于给定的 autorelease 消息，相应的 release 消息在 autorelease pool block 的末尾向发送过 autorelease 消息的对象发送。

Cocoa 总是希望代码在 autorelease pool block 中执行，否则自动释放的对象不会被释放而应用程序会泄漏内存。（如果你在 autorelease pool block 之外发送 autorelease 消息，Cocoa 会记录一个合适的错误消息。）AppKit 和 UIKit 框架处理 autorelease pool block 中的每个事件循环迭代（例如鼠标按下事件或敲击）。因此，你通常不必自己创建 autorelease pool block，甚至不必查看用于创建池的代码。但是，有三种情况可能会使用你自己的自动释放池块：

- 如果你正在编写不基于 UI 框架的程序，例如命令行工具。
- 如果编写一个创建许多临时对象的循环。

  你可以在循环内使用 autorelease pool block 在下一次迭代之前处理这些对象。在循环中使用 autorelease pool block 有助于减少应用程序的最大内存占用量。

- 如果你产生一个辅助线程。

  一旦线程开始执行，你必须创建自己的 autorelease pool block; 否则，你的应用程序将泄漏对象。

### 使用 Local Autorelease Pool Blocks 来减少峰值内存占用量

许多程序创建自动释放的临时对象。这些对象会添加到程序的内存占用空间，直到块结束。在许多情况下，允许临时对象累积直到当前事件循环迭代结束时不会导致过多的开销; 但是，在某些情况下，你可能会创建大量临时对象，这些对象会大大增加内存占用，并且你希望更快地处置。在后面这些情况下，你可以创建自己的 autorelease pool block。在块结束时，临时对象被释放，这通常导致它们的释放，从而减少程序的内存占用。

以下示例显示了如何在 for 循环中使用 local autorelease pool block。

```objc
NSArray *urls = <# An array of file URLs #>;
for (NSURL *url in urls) {
 
    @autoreleasepool {
        NSError *error;
        NSString *fileContents = [NSString stringWithContentsOfURL:url
                                         encoding:NSUTF8StringEncoding error:&error];
        /* Process the string, creating and autoreleasing more objects. */
    }
}
```

for 循环一次处理一个文件。在 autorelease pool block 内发送 autorelease 消息的任何对象（例如 fileContents）在块结束时释放。

在 autorelease pool block 之后，你应该将块中自动释放的任何对象视为“已处置”。不要向该对象发送消息或将其返回给你的方法的调用者。如果必须使用 autorelease pool block 之外的临时对象，则可以通过向块内的对象发送保留消息，然后在块之后将其自动释放发送，如此示例所示：

```objc
– (id)findMatchingObject:(id)anObject {
 
    id match;
    while (match == nil) {
        @autoreleasepool {
 
            /* Do a search that creates a lot of temporary objects. */
            match = [self expensiveSearchForObject:anObject];
 
            if (match != nil) {
                [match retain]; /* Keep match around. */
            }
        }
    }
 
    return [match autorelease];   /* Let match go and return it. */
}
```

发送 retain 以在自 autorelease pool block 中匹配并在 autorelease pool block 延长匹配的生命周期后向其发送自动释放，并允许它在循环外接收消息并返回到 findMatchingObject: 的调用者。

### Autorelease Pool Blocks 和线程

Cocoa 应用程序中的每个线程都维护自己的 autorelease pool blocks 栈。如果你正在编写仅基于 Foundation 的程序或者分离线程，则需要创建自己的 autorelease pool block。

如果你的应用程序或线程长寿并且可能生成大量自动释放的对象，则应使用 autorelease pool blocks（如主线程上的 AppKit 和 UIKit）; 否则，自动释放的对象会累积，并且你的内存占用会增加。如果你的分离线程没有进行 Cocoa 调用，则不需要使用 autorelease pool block。