# 派遣队列

Grand Central Dispatch（GCD）调度队列是执行任务的强大工具。 调度队列允许你相对于调用者异步或同步地执行任意代码块。 你可以使用调度队列执行几乎所有在单独线程上执行的任务。 调度队列的优点是它们比相应的线程代码更易于使用，并且执行这些任务的效率更高。

本章介绍了调度队列，以及有关如何使用它们在应用程序中执行常规任务的信息。 如果要使用调度队列替换现有的线程代码，可以在[迁移线程](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW1)中找到一些有关如何执行此操作的其他提示。

## 关于调度队列

调度队列是在应用程序中异步和并发执行任务的简便方法。 任务只是你的应用程序需要执行的一些工作。 例如，你可以定义任务以执行某些计算，创建或修改数据结构，处理从文件读取的一些数据或任何数量的事物。 你可以通过将相应的代码放在函数或块对象中并将其添加到调度队列来定义任务。

调度队列是一种类似于对象的结构，用于管理你提交给它的任务。 所有调度队列都是先进先出的数据结构。 因此，添加到队列的任务始终按照添加的顺序启动。 GCD 会自动为你提供一些调度队列，但你可以为特定目的创建其他调度队列。 表3-1列出了应用程序可用的调度队列类型以及如何使用它们。

表 3-1 调度队列的类型

| 类型 | 描述 |
| - | - |
| Serial | 串行队列（也称为专用调度队列）按照将它们添加到队列的顺序一次执行一个任务。 当前正在执行的任务在由调度队列管理的不同线程（可能因任务而异）上运行。 串行队列通常用于同步对特定资源的访问。 你可以根据需要创建任意数量的串行队列，并且每个队列与所有其他队列同时运行。 换句话说，如果创建四个串行队列，则每个队列一次只执行一个任务，但最多可以同时执行四个任务，每个队列一个。 有关如何创建串行队列的信息，请参阅[创建串行调度队列](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW6)。 |
| Concurrent | 并发队列（也称为一种全局调度队列）同时执行一个或多个任务，但任务仍按其添加到队列的顺序启动。 当前正在执行的任务在由调度队列管理的不同线程上运行。 在任何给定点执行的确切任务数量是可变的，取决于系统条件。 在 iOS 5 及更高版本中，你可以通过将 DISPATCH_QUEUE_CONCURRENT 指定为队列类型来自行创建并发调度队列。 此外，还有四个预定义的全局并发队列供你的应用程序使用。 有关如何获取全局并发队列的更多信息，请参阅[获取全局并发调度队列](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW5)。 |
| Main dispatch queue | 主调度队列是一个全局可用的串行队列，它在应用程序的主线程上执行任务。 此队列与应用程序的运行循环（如果存在）一起工作，以将排队任务的执行与附加到运行循环的其他事件源的执行交错。 因为它在应用程序的主线程上运行，所以主队列通常用作应用程序的关键同步点。 虽然你不需要创建主调度队列，但你确实需要确保应用程序正确地消耗它。 有关如何管理此队列的更多信息，请参阅[在主线程上执行任务](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW15)。 |

在为应用程序添加并发性时，调度队列提供了多个优于线程的优势。最直接的优点是工作队列编程模型的简单性。使用线程，你必须为要执行的工作以及线程本身的创建和管理编写代码。调度队列使你可以专注于你实际想要执行的工作，而无需担心线程的创建和管理。相反，系统会为你处理所有线程创建和管理。优点是系统能够比任何单个应用程序更有效地管理线程。系统可以根据可用资源和当前系统条件动态扩展线程数。此外，系统通常能够比你自己创建线程时更快地开始运行任务。

虽然你可能认为重写代码队列的代码很困难，但为代码队列编写代码通常比编写线程代码更容易。编写代码的关键是设计自包含且能够异步运行的任务。 （这对于线程和调度队列都是如此。）但是，调度队列具有优势的可预测性。如果你有两个访问相同共享资源但在不同线程上运行的任务，则任一线程都可以先修改资源，你需要使用锁来确保两个任务不会同时修改该资源。使用调度队列，你可以将两个任务添加到串行调度队列，以确保在任何给定时间只有一个任务修改了资源。这种基于队列的同步比锁更有效，因为锁在争用和无争议的情况下总是需要昂贵的内核陷阱，而调度队列主要在应用程序的进程空间中工作，并且只在绝对必要时才调用内核。

虽然你指出在串行队列中运行的两个任务不会同时运行是正确的，但你必须记住，如果两个线程同时进行锁定，则线程提供的任何并发都将丢失或显着减少。更重要的是，线程模型需要创建两个线程，这两个线程占用内核和用户空间内存。调度队列不会为其线程支付相同的内存损失，并且它们使用的线程保持忙碌而不会被阻塞。

关于调度队列的一些其他关键要点包括：

- 调度队列与其他调度队列同时执行其任务。任务的序列化仅限于单个调度队列中的任务。
- 系统确定任何时间执行的任务总数。因此，在100个不同队列中具有100个任务的应用程序可能不会同时执行所有这些任务（除非它具有100个或更多有效核心）。
- 在选择要启动的新任务时，系统会考虑队列优先级。有关如何设置串行队列优先级的信息，请参阅[为队列提供清理功能](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW7)。
- 队列中的任务必须准备好在添加到队列时执行。 （如果你之前使用过 Cocoa 操作对象，请注意此行为与模型操作使用的不同。）
- 专用调度队列是引用计数对象。除了在自己的代码中保留队列之外，请注意，调度源也可以附加到队列并增加其保留计数。因此，你必须确保取消所有调度源，并使用适当的释放调用平衡所有保留调用。有关保留和释放队列的详细信息，请参阅[调度队列的内存管理](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW11)。有关调度源的更多信息，请参阅[关于调度源](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW12)。

有关用于操作调度队列的接口的更多信息，请参阅 Grand Central Dispatch（GCD）参考。

## 队列相关技术

除了调度队列之外，Grand Central Dispatch 还提供了一些使用队列来帮助管理代码的技术。 表3-2列出了这些技术，并提供了指向你可以在其中找到有关它们的更多信息的链接。

表 3-2 使用调度队列的技术

| 技术 | 描述 |
| - | - |
| Dispatch groups | 调度组是一种监视一组块对象以完成的方法。 （你可以根据需要同步或异步监视块。）组为代码提供了有用的同步机制，这取决于其他任务的完成情况。 有关使用组的详细信息，请参阅[等待排队任务组](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW25)。 |
| Dispatch semaphores | 调度信号量类似于传统信号量，但通常更有效。 仅当因为信号量不可用而需要阻塞调用线程时，Dispatch 信号量才会调用内核。 如果信号量可用，则不进行内核调用。 有关如何使用调度信号量的示例，请参阅[使用调度信号量来调节有限资源的使用](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW24)。 |
| Dispatch sources | 调度源生成通知以响应特定类型的系统事件。 你可以使用调度源来监视诸如进程通知，信号和描述符事件等事件。 发生事件时，调度源将你的任务代码异步提交到指定的调度队列以进行处理。 有关创建和使用调度源的更多信息，请参阅[调度源](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1)。 |

## 使用块实现任务

块对象是基于 C 的语言特性，可以在 C，Objective-C 和 C++ 代码中使用。通过块，可以轻松定义独立的工作单元。尽管它们看起来类似于函数指针，但实际上块是由类似于对象的底层数据结构表示的，并由编译器为你创建和管理。编译器打包你提供的代码（以及任何相关数据），并将其封装在可以存放在堆中并在应用程序中传递的表单中。

块的一个关键优势是它们能够在自己的词法范围之外使用变量。在函数或方法中定义块时，块在某些方面将充当传统的代码块。例如，块可以读取父作用域中定义的变量的值。由块访问的变量被复制到堆上的块数据结构中，以便块稍后可以访问它们。将块添加到调度队列时，通常必须将这些值保留为只读格式。但是，同步执行的块也可以使用前面带有 __block 关键字的变量将数据返回给父调用范围。

使用与函数指针使用的语法类似的语法声明与代码内联的块。块和函数指针之间的主要区别在于块名称前面有插入符号（^）而不是星号（*）。像函数指针一样，你可以将参数传递给块并从中接收返回值。清单3-1展示了如何在代码中同步声明和执行块。变量 aBlock 被声明为一个块，它接受一个整数参数并且不返回任何值。然后将匹配该原型的实际块分配给 aBlock 并以内联方式声明。最后一行立即执行块，将指定的整数打印到标准输出。

清单 3-1 一个简单的块示例

```
int x = 123;
int y = 456;
 
// Block declaration and assignment
void (^aBlock)(int) = ^(int z) {
    printf("%d %d %d\n", x, y, z);
};
 
// Execute the block
aBlock(789);   // prints: 123 456 789
```

以下是设计块时应考虑的一些关键指南的摘要：

- 对于计划使用调度队列异步执行的块，可以安全地从父函数或方法捕获标量变量，并在块中使用它们。但是，你不应尝试捕获由调用上下文分配和删除的大型结构或其他基于指针的变量。执行块时，该指针引用的内存可能会消失。当然，自己分配内存（或对象）并明确地将该内存的所有权移交给块是安全的。
- Dispatch 队列复制添加到它们的块，并在它们完成执行时释放块。换句话说，在将块添加到队列之前，不需要显式复制块。
- 尽管队列在执行小任务时比原始线程更有效，但仍然存在创建块并在队列上执行它们的开销。如果块的工作量太少，则内联执行它可能比将其分配到队列更便宜。判断块是否工作量太少的方法是使用性能工具收集每个路径的度量标准并进行比较。
- 不要缓存相对于底层线程的数据，并期望可以从不同的块访问数据。如果同一队列中的任务需要共享数据，请使用调度队列的上下文指针来存储数据。有关如何访问调度队列的上下文数据的更多信息，请参阅[使用队列存储自定义上下文信息](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW13)。
- 如果你的块创建了多个 Objective-C 对象，你可能希望将块的代码部分包含在 @autorelease 块中，以处理这些对象的内存管理。尽管 GCD 调度队列具有自己的自动释放池，但它们无法保证这些池何时耗尽。如果你的应用程序受内存限制，则创建自己的自动释放池可以让你以更加规律的间隔释放自动释放对象的内存。

有关块的更多信息，包括如何声明和使用它们，请参阅[块编程主题](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40007502)。 有关如何将块添加到调度队列的信息，请参阅[向队列添加任务](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW20)。

## 创建和管理调度队列

在将任务添加到队列之前，必须确定要使用的队列类型以及打算如何使用它。 调度队列可以串行或同时执行任务。 此外，如果你特意使用队列，则可以相应地配置队列属性。 以下部分介绍如何创建调度队列并配置它们以供使用。

### 获取全局并发调度队列

当你有多个可以并行运行的任务时，并发调度队列非常有用。并发队列仍然是一个队列，因为它以先进先出的顺序将任务队列化;但是，并发队列可能会在任何先前任务完成之前将其他任务出列。并发队列在任何给定时刻执行的实际任务数都是可变的，并且可以随着应用程序中的条件的变化而动态变化。许多因素会影响并发队列执行的任务数，包括可用核心数，其他进程正在完成的工作量以及其他串行调度队列中任务的数量和优先级。

系统为每个应用程序提供四个并发调度队列。这些队列对于应用程序是全局的，并且仅通过其优先级来区分。因为它们是全局的，所以不要显式创建它们。相反，你使用 dispatch_get_global_queue 函数请求其中一个队列，如以下示例所示：

```
dispatch_queue_t aQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```

除了获取默认并发队列之外，你还可以通过将 DISPATCH_QUEUE_PRIORITY_HIGH 和 DISPATCH_QUEUE_PRIORITY_LOW 常量传递给函数来获取具有高优先级和低优先级的队列，或者通过传递 DISPATCH_QUEUE_PRIORITY_BACKGROUND 常量来获取后台队列。 正如你所料，高优先级并发队列中的任务在默认队列和低优先级队列中的任务之前执行。 同样，默认队列中的任务在低优先级队列中的任务之前执行。

> 注意：dispatch_get_global_queue 函数的第二个参数保留用于将来的扩展。 现在，你应该始终为此参数传递0。

虽然调度队列是引用计数对象，但你不需要保留和释放全局并发队列。 因为它们对你的应用程序是全局的，所以忽略对这些队列的保留和释放调用。 因此，你不需要存储对这些队列的引用。 只要需要引用其中一个函数，就可以调用 dispatch_get_global_queue 函数。 

### 创建串行调度队列

当你希望任务按特定顺序执行时，串行队列非常有用。串行队列一次只执行一个任务，并始终从队列的头部提取任务。你可以使用串行队列而不是锁来保护共享资源或可变数据结构。与锁不同，串行队列确保以可预测的顺序执行任务。只要你将任务异步提交到串行队列，队列就永远不会死锁。

与为你创建的并发队列不同，你必须显式创建和管理要使用的任何串行队列。你可以为应用程序创建任意数量的串行队列，但应避免创建大量串行队列，以此作为同时执行任意数量任务的方法。如果要同时执行大量任务，请将它们提交到其中一个全局并发队列。创建串行队列时，请尝试确定每个队列的用途，例如保护资源或同步应用程序的某些关键行为。

清单3-2显示了创建自定义串行队列所需的步骤。 [dispatch_queue_create](https://developer.apple.com/documentation/dispatch/1453030-dispatch_queue_create) 函数有两个参数：队列名称和一组队列属性。调试器和性能工具显示队列名称，以帮助你跟踪任务的执行方式。队列属性保留供将来使用，应为 NULL。

清单 3-2 创建一个新的串行队列

```
dispatch_queue_t queue;
queue = dispatch_queue_create("com.example.MyQueue", NULL);
```

除了你创建的任何自定义队列之外，系统还会自动创建一个串行队列并将其绑定到应用程序的主线程。 有关获取主线程队列的更多信息，请参阅[在运行时获取常见队列](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW3)。

### 在运行时获取常见队列

Grand Central Dispatch 提供的功能允许你从应用程序访问多个常见的调度队列：

- 使用 [dispatch_get_current_queue](https://developer.apple.com/documentation/dispatch/1493248-dispatch_get_current_queue) 函数进行调试或测试当前队列的标识。 从块对象内部调用此函数将返回块已提交到的队列（现在可能正在运行该队列）。 从块外部调用此函数将返回应用程序的默认并发队列。
- 使用 dispatch_get_main_queue 函数获取与应用程序主线程关联的串行调度队列。 此队列是为 Cocoa 应用程序以及在主线程上调用 [dispatch_main](https://developer.apple.com/documentation/dispatch/1452860-dispatchmain) 函数或配置运行循环（使用 [CFRunLoopRef](https://developer.apple.com/documentation/corefoundation/cfrunloopref) 类型或 [NSRunLoop](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Classes/NSRunLoop/Description.html#//apple_ref/occ/cl/NSRunLoop) 对象）的应用程序自动创建的。
- 使用 [dispatch_get_global_queue](https://developer.apple.com/documentation/dispatch/1452927-dispatch_get_global_queue) 函数获取任何共享并发队列。 有关更多信息，请参阅[获取全局并发调度队列](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW5)。

### 调度队列的内存管理

调度队列和其他调度对象是引用计数数据类型。 创建串行调度队列时，它的初始引用计数为1.你可以使用 dispatch_retain 和 dispatch_release 函数根据需要递增和递减该引用计数。 当队列的引用计数达到零时，系统会异步释放队列。

保留和释放调度对象（例如队列）非常重要，以确保它们在使用时保留在内存中。 与内存管理的 Cocoa 对象一样，一般规则是如果你计划使用传递给代码的队列，则应在使用之前保留队列，并在不再需要时释放它。 这种基本模式可确保队列在你使用时保留在内存中。

> 注意：你无需保留或释放任何全局调度队列，包括并发调度队列或主调度队列。 任何保留或释放队列的尝试都将被忽略。

即使你实现了垃圾收集的应用程序，你仍必须保留并释放调度队列和其他调度对象。 Grand Central Dispatch 不支持回收内存的垃圾收集模型。

### 使用队列存储自定义上下文信息

所有调度对象（包括调度队列）都允许你将自定义上下文数据与对象相关联。 要在给定对象上设置和获取此数据，请使用 [dispatch_set_context](https://developer.apple.com/documentation/dispatch/1452807-dispatch_set_context) 和 [dispatch_get_context](https://developer.apple.com/documentation/dispatch/1453005-dispatch_get_context) 函数。 系统不以任何方式使用你的自定义数据，你可以在适当的时间分配和取消分配数据。

对于队列，你可以使用上下文数据存储指向 Objective-C 对象或其他数据结构的指针，以帮助识别队列或其对代码的预期用途。 你可以使用队列的终结器函数在取消分配之前从队列中释放（或取消关联）你的上下文数据。 清单3-3显示了如何编写清除队列上下文数据的终结器函数的示例。

### 为队列提供清理功能

创建串行调度队列后，可以附加终结器函数以在取消分配队列时执行任何自定义清理。调度队列是引用计数对象，你可以使用 [dispatch_set_finalizer_f](https://developer.apple.com/documentation/dispatch/1453100-dispatch_set_finalizer_f) 函数指定当队列的引用计数达到零时要执行的函数。你可以使用此函数清除与队列关联的上下文数据，并且仅在上下文指针不为 NULL 时才调用该函数。

清单3-3显示了一个自定义终结器函数和一个创建队列并安装该终结器的函数。队列使用终结器函数来释放存储在队列的上下文指针中的数据。 （从代码引用的 myInitializeDataContextFunction 和 myCleanUpDataContextFunction 函数是用于初始化和清理数据结构本身内容的自定义函数。）传递给终结函数的上下文指针包含与队列关联的数据对象。

清单 3-3 安装队列清理功能

```
void myFinalizerFunction(void *context)
{
    MyDataContext* theData = (MyDataContext*)context;
 
    // Clean up the contents of the structure
    myCleanUpDataContextFunction(theData);
 
    // Now release the structure itself.
    free(theData);
}
 
dispatch_queue_t createMyQueue()
{
    MyDataContext*  data = (MyDataContext*) malloc(sizeof(MyDataContext));
    myInitializeDataContextFunction(data);
 
    // Create the queue and set the context data.
    dispatch_queue_t serialQueue = dispatch_queue_create("com.example.CriticalTaskQueue", NULL);
    dispatch_set_context(serialQueue, data);
    dispatch_set_finalizer_f(serialQueue, &myFinalizerFunction);
 
    return serialQueue;
}
```

## 将任务添加到队列

要执行任务，必须将其分派到适当的调度队列。 你可以同步或异步分派任务，也可以单独或成组分派任务。 一旦进入队列，队列将负责尽快执行任务，因为它具有约束和队列中已有的任务。 本节向你展示了将任务分派到队列的一些技术，并描述了每种技术的优点。

### 将单个任务添加到队列

有两种方法可以将任务添加到队列中：异步或同步。如果可能，使用 dispatch_async 和 dispatch_async_f 函数的异步执行优先于同步备选。将块对象或函数添加到队列时，无法知道该代码何时执行。因此，异步添加块或函数可以让你安排代码的执行，并继续从调用线程执行其他工作。如果你从应用程序的主线程调度任务（可能是为了响应某些用户事件），这一点尤为重要。

虽然你应尽可能异步添加任务，但有时可能需要同步添加任务以防止竞争条件或其他同步错误。在这些实例中，你可以使用 dispatch_sync 和 dispatch_sync_f 函数将任务添加到队列中。这些函数会阻止当前执行的线程，直到指定的任务完成执行。

> 要点：永远不要从正在计划传递给函数的同一队列中执行的任务调用 dispatch_sync 或 dispatch_sync_f 函数。 这对于保证死锁的串行队列尤其重要，但对于并发队列也应该避免。

以下示例显示如何使用基于块的变体异步和同步分派任务：

```
dispatch_queue_t myCustomQueue;
myCustomQueue = dispatch_queue_create("com.example.MyCustomQueue", NULL);
 
dispatch_async(myCustomQueue, ^{
    printf("Do some work here.\n");
});
 
printf("The first block may or may not have run.\n");
 
dispatch_sync(myCustomQueue, ^{
    printf("Do some more work here.\n");
});
printf("Both blocks have completed.\n");
```

### 任务完成时执行完成阻止

就其性质而言，调度到队列的任务独立于创建它们的代码而运行。但是，当任务完成后，你的应用程序可能仍希望收到有关该事实的通知，以便它可以合并结果。使用传统的异步编程，你可以使用回调机制执行此操作，但使用调度队列可以使用完成块。

完成块只是你在原始任务结束时分派到队列的另一段代码。调用代码通常在完成任务时将完成块作为参数提供。所有任务代码必须做的是在完成其工作时将指定的块或函数提交到指定的队列。

清单3-4显示了使用块实现的平均功能。平均函数的最后两个参数允许调用者指定报告结果时要使用的队列和块。在 averaging 函数计算其值后，它会将结果传递给指定的块并将其分派到队列中。为防止队列过早释放，最初保留该队列并在分派完成块后释放该队列至关重要。

清单 3-4 在任务之后执行完成回调

```
void average_async(int *data, size_t len,
   dispatch_queue_t queue, void (^block)(int))
{
   // Retain the queue provided by the user to make
   // sure it does not disappear before the completion
   // block can be called.
   dispatch_retain(queue);
 
   // Do the work on the default concurrent queue and then
   // call the user-provided block with the results.
   dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
      int avg = average(data, len);
      dispatch_async(queue, ^{ block(avg);});
 
      // Release the user-provided queue when done
      dispatch_release(queue);
   });
}
```

### 同时执行循环迭代

并发调度队列可能会提高性能的地方是你有一个循环执行固定迭代次数的地方。 例如，假设你有一个for循环，它通过每个循环迭代执行一些工作：

```
for (i = 0; i < count; i++) {
   printf("%u\n",i);
}
```

如果在每次迭代期间执行的工作与在所有其他迭代期间执行的工作不同，并且每个连续循环完成的顺序不重要，则可以通过调用 dispatch_apply 或 dispatch_apply_f 函数来替换循环。 对于每个循环迭代，这些函数将指定的块或函数提交给队列一次。 当分派到并发队列时，因此可以同时执行多个循环迭代。

调用 [dispatch_apply](https://developer.apple.com/documentation/dispatch/1453050-dispatch_apply) 或 [dispatch_apply_f](https://developer.apple.com/documentation/dispatch/1452846-dispatch_apply_f) 时，可以指定串行队列或并发队列。 传入并发队列允许你同时执行多个循环迭代，这是使用这些函数的最常用方法。 虽然允许使用串行队列并为你的代码做正确的事情，但使用这样的队列与将循环留在原位相比没有真正的性能优势。

> 要点：与常规 for 循环一样，[dispatch_apply](https://developer.apple.com/documentation/dispatch/1453050-dispatch_apply) 和 dispatch_apply_f 函数在所有循环迭代完成之前不会返回。 因此，从已经从队列上下文执行的代码中调用它们时应该小心。 如果作为参数传递给函数的队列是串行队列，并且与执行当前代码的队列相同，则调用这些函数将使队列死锁。

> 因为它们有效地阻塞了当前线程，所以在从主线程调用这些函数时也要小心，它们可能会阻止事件处理循环及时响应事件。 如果循环代码需要大量的处理时间，则可能需要从不同的线程调用这些函数。

清单3-5显示了如何使用 dispatch_apply 语法替换前面的 for 循环。 传递给 dispatch_apply 函数的块必须包含一个标识当前循环迭代的参数。 执行该块时，该参数的值对于第一次迭代为0，对于第二次迭代为1，依此类推。 最后一次迭代的参数值为 count-1，其中 count 是迭代的总次数。

清单 3-5 同时执行 for 循环的迭代

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
 
dispatch_apply(count, queue, ^(size_t i) {
   printf("%u\n",i);
});
```

你应该确保你的任务代码在每次迭代中都能完成合理的工作量。 与调度到队列的任何块或函数一样，调度该代码以执行也会产生开销。 如果循环的每次迭代只执行少量工作，则调度代码的开销可能会超过从将其分派到队列时可能获得的性能优势。 如果在测试期间发现这是真的，则可以使用跨步来增加每次循环迭代期间执行的工作量。 通过跨步，你可以将原始循环的多次迭代组合到一个块中，并按比例减少迭代次数。 例如，如果你最初执行100次迭代但决定使用4的步幅，则现在从每个块执行4次循环迭代，并且迭代计数为25.有关如何实现跨步的示例，请参阅[改进循环代码](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW2)。

### 在主线程上执行任务

Grand Central Dispatch 提供了一个特殊的调度队列，你可以使用该队列在应用程序的主线程上执行任务。 此队列是为所有应用程序自动提供的，并由在其主线程上设置运行循环（由 [CFRunLoopRef](https://developer.apple.com/documentation/corefoundation/cfrunloopref) 类型或 [NSRunLoop](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Classes/NSRunLoop/Description.html#//apple_ref/occ/cl/NSRunLoop) 对象管理）的任何应用程序自动排空。 如果你没有创建 Cocoa 应用程序并且不想显式设置运行循环，则必须调用 [dispatch_main](https://developer.apple.com/documentation/dispatch/1452860-dispatchmain) 函数以显式排空主调度队列。 你仍然可以向队列添加任务，但如果你不调用此函数，则永远不会执行这些任务。

你可以通过调用 dispatch_get_main_queue 函数来获取应用程序主线程的调度队列。 添加到此队列的任务在主线程本身上串行执行。 因此，你可以将此队列用作在应用程序的其他部分中完成工作的同步点。

### 在任务中使用 Objective-C 对象

GCD 提供对 Cocoa 内存管理技术的内置支持，因此你可以在提交的块中自由使用 Objective-C 对象来调度队列。 每个调度队列都维护自己的自动释放池，以确保在某个时刻释放自动释放的对象; 队列无法保证何时实际释放这些对象。

如果你的应用程序受内存限制且你的块创建了多个自动释放的对象，则创建自己的自动释放池是确保及时释放对象的唯一方法。 如果你的块创建了数百个对象，你可能希望创建多个自动释放池或定期排空池。

有关自动释放池和 Objective-C 内存管理的详细信息，请参阅[高级内存管理编程指南](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)。

## 暂停和恢复队列

你可以通过挂起来阻止队列临时执行块对象。 使用 [dispatch_suspend](https://developer.apple.com/documentation/dispatch/dispatchobject/1452801-suspend) 函数挂起调度队列，并使用 [dispatch_resume](https://developer.apple.com/documentation/dispatch/dispatchobject/1452929-resume) 函数恢复它。 调用 dispatch_suspend 会增加队列的挂起引用计数，并调用 dispatch_resume 会减少引用计数。 当引用计数大于零时，队列保持挂起状态。 因此，你必须使用匹配的恢复调用来平衡所有挂起调用，以便恢复处理块。

> 重要说明：挂起和恢复调用是异步的，仅在执行块之间生效。 挂起队列不会导致已执行的块停止。

## 使用 Dispatch 信号量来规范有限资源的使用

如果你要提交给调度队列的任务访问某些有限资源，你可能希望使用调度信号量来规范同时访问该资源的任务数。调度信号量的作用类似于常规信号量，但有一个例外。当资源可用时，获取调度信号量所需的时间少于获取传统系统信号量所需的时间。这是因为 Grand Central Dispatch 不针对这种特殊情况调用内核。它调用内核的唯一时间是资源不可用时，系统需要停止线程，直到发出信号量信号。

使用调度信号量的语义如下：

1、创建信号量时（使用 dispatch_semaphore_create 函数），可以指定一个正整数，指示可用资源的数量。

2、在每个任务中，调用 dispatch_semaphore_wait 来等待信号量。

3、等待调用返回时，获取资源并完成工作。

4、完成资源后，通过调用 dispatch_semaphore_signal 函数释放它并发信号通知信号量。

有关这些步骤如何工作的示例，请考虑在系统上使用文件描述符。每个应用程序都有一定数量的文件描述符供使用。如果你有一个处理大量文件的任务，你不希望一次打开那么多文件描述符。相反，你可以使用信号量来限制文件处理代码在任何时候使用的文件描述符的数量。你将要包含在任务中的基本代码如下：

```
// Create the semaphore, specifying the initial pool size
dispatch_semaphore_t fd_sema = dispatch_semaphore_create(getdtablesize() / 2);
 
// Wait for a free file descriptor
dispatch_semaphore_wait(fd_sema, DISPATCH_TIME_FOREVER);
fd = open("/etc/services", O_RDONLY);
 
// Release the file descriptor when done
close(fd);
dispatch_semaphore_signal(fd_sema);
```

创建信号量时，指定可用资源的数量。 该值成为信号量的初始计数变量。 每次等待信号量时，dispatch_semaphore_wait 函数会将 count 计数变量减1.如果结果值为负，则函数会告诉内核阻塞你的线程。 另一方面，dispatch_semaphore_signal 函数将 count 变量递增1以指示资源已被释放。 如果有任务被阻止并等待资源，则其中一个随后被解除阻止并被允许执行其工作。

## 等待排队任务组

调度组是一种阻止线程直到一个或多个任务完成执行的方法。你可以在完成所有指定任务之前无法取得进展的位置使用此行为。例如，在分派几个任务来计算某些数据之后，你可以使用组等待这些任务，然后在完成后处理结果。使用调度组的另一种方法是作为线程连接的替代方法。你可以将相应的任务添加到调度组并等待整个组，而不是启动多个子线程然后再加入每个子线程。

清单3-6显示了设置组，向其分派任务以及等待结果的基本过程。不是使用 dispatch_async 函数将任务分派到队列，而是使用 dispatch_group_async 函数。此函数将任务与组关联，并将其排队以执行。要等待一组任务完成，然后使用 dispatch_group_wait 函数，传入适当的组。

清单 3-6 等待异步任务
```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
 
// Add a task to the group
dispatch_group_async(group, queue, ^{
   // Some asynchronous work
});
 
// Do some other work while the tasks execute.
 
// When you cannot make any more forward progress,
// wait on the group to block the current thread.
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
 
// Release the group when it is no longer needed.
dispatch_release(group);
```

## 调度队列和线程安全

在调度队列的上下文中谈论线程安全可能看起来很奇怪，但线程安全仍然是一个相关的主题。每当你在应用程序中实现并发时，您应该知道以下几点：

- 调度队列本身是线程安全的。换句话说，你可以从系统上的任何线程向调度队列提交任务，而无需先锁定或同步对队列的访问。
- 不要从传递给函数调用的同一队列上执行的任务调用 [dispatch_sync](https://developer.apple.com/documentation/dispatch/1452870-dispatch_sync) 函数。这样做会使队列死锁。如果需要调度到当前队列，请使用 [dispatch_async](https://developer.apple.com/documentation/dispatch/1453057-dispatch_async) 函数异步执行此操作。
- 避免从你提交给调度队列的任务中获取锁定。尽管使用任务中的锁是安全的，但是当你获得锁时，如果该锁不可用，则可能会完全阻塞串行队列。同样，对于并​​发队列，等待锁定可能会阻止其他任务执行。如果需要同步部分代码，请使用串行调度队列而不是锁。
- 虽然你可以获取有关运行任务的基础线程的信息，但最好避免这样做。有关调度队列与线程兼容性的更多信息，请参阅[与 POSIX 线程的兼容性](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW18)。

有关如何更改现有线程代码以使用调度队列的其他提示，请参阅[从线程迁移](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW1)。