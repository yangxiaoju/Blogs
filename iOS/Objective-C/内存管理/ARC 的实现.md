# ARC 的实现

> 本文为 《Objective-C 高级编程》1.4 中的内容

ARC 由以下工具、库来实现：

- clang（LLVM 编译器）3.0 以上
- objc4 Objective-C 运行时库 493.9 以上

通过 clang 汇编输出和 objc4 库（主要是 runtime/objc-arr.mm）的源代码进行说明。

## __strong 修饰符

赋值给附有 __strong 修饰符的变量在实际的程序中到底是怎样运行的呢？

```objc
{
    id __strong obj = [[NSObject alloc] init];
}
```

在编译器选项 “-S” 的同时运行 clang，可取得程序汇编输出。看看汇编输出和 objc4 库的源代码就能够知道程序是如何工作的。该源代码实际上可转换为调用以下的函数。为了便于理解，以后的源代码有时也使用模拟源代码。

```objc
/* 编译器的模拟代码*/
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));
objc_release(obj);
```

如原源代码所示，2次调用 objc_msgSend 方法（alloc 方法和 init 方法），变量作用域结束时通过 objc_release 释放对象。虽然 ARC 有效时不能使用 release 方法，但由此可知编译器自动插入了 release。下面我们来看看使用 alloc/new/copy/mutableCopy 以外的方法会是什么情况。

```objc
{
    id __strong obj = [NSMutableArray array];
}
```

虽然调用了我们熟知的 NSMutableArray 类的 array 类方法，但得到的结果却与之前稍有不同。

```objc
/* 编译器的模拟代码*/
id obj = objc_msgSend(NSMutableArray, @selector(array));
objc_retainAutoreleasedReturnValue(obj);
objc_release(obj);
```

虽然最开始的 array 方法的调用以及最后变量作用域结束时的 release 与之前相同，但中间的 objc_retainAutoreleasedReturnValue 函数是什么呢？

objc_retainAutoreleasedReturnValue 函数主要用于最优化程序运行。顾名思义，它是用于自己持有（retain）对象的函数，但它持有的对象应为返回注册在 autoreleasepool 中对象的方法，或是函数的返回值。像该源代码这样，在调用 alloc/new/copy/mutableCopy 以外的方法，即 NSMutableArray 类的 array 类方法等调用之后，由编译器插入该函数。

这种 objc_retainAutoreleasedReturnValue 函数是成对的，与之相对的函数是 objc_autoreleaseReturnValue。它用于 alloc/new/copy/mutableCopy 方法以外的 NSMutableArray 类的 array 类方法等返回对象的实现上。下面我们看看 NSMutableArray 类的 array 类通过编译器会进行怎样的转换。

```objc
+(id)array {
    return [[NSMutableArray alloc] init];
}
```

以下为该源代码的转换，转换后的源代码使用了 objc_autoreleaseReturnValue 函数。

```objc
+(id)array
{
    id obj = objc_msgSend(NSMutableArray, @selector(alloc));
    objc_msgSend(obj, @selector(init));
    return objc_autoreleaseReturnValue(obj);
}
```

像该源代码这样，返回注册到 autoreleasepool 中对象的方法使用了 objc_autoreleaseReturnValue 函数返回注册到 autoreleasepool 中的对象。但是 objc_autoreleaseReturnValue 函数同 objc_autorelease 函数不同，一般不仅限于注册对象到 autoreleasepool 中。

objc_autoreleaseReturnValue 函数会检查使用该函数的方法或函数调用方的执行列表，如果方法或函数的调用方在调用了方法或函数后紧接着调用 objc_retainAutoreleaseReturnValue() 函数，那么就不将返回的对象注册到 autorelease 中，而是直接传递到方法或函数的调用方。objc_retainAutoreleasedReturnValue 函数与 objc_retain 函数不同，它即便不注册到 autoreleasepool 中而返回对象，也能够正确地获取对象。通过 objc_autoreleaseReturnValue 函数和 objc_retainAutoreleasedReturnValue 函数的写作，可以不将对象注册到 autorelease 中而直接传递，这一过程达到了最优化。

### objc_autoreleaseReturnValue

objc_autoreleaseReturnValue 实现如下：

```cpp
// Prepare a value at +1 for return through a +0 autoreleasing convention.
id 
objc_autoreleaseReturnValue(id obj)
{
    if (prepareOptimizedReturn(ReturnAtPlus1)) return obj;

    return objc_autorelease(obj);
}
```

prepareOptimizedReturn 是优化返回的关键，其实现如下：

```cpp
// Try to prepare for optimized return with the given disposition (+0 or +1).
// Returns true if the optimized path is successful.
// Otherwise the return value must be retained and/or autoreleased as usual.
static ALWAYS_INLINE bool 
prepareOptimizedReturn(ReturnDisposition disposition)
{
    assert(getReturnDisposition() == ReturnAtPlus0);

    if (callerAcceptsOptimizedReturn(__builtin_return_address(0))) {
        if (disposition) setReturnDisposition(disposition);
        return true;
    }

    return false;
}
```

#### __builtin_return_address

gcc 的编译特性使用 __builtin_return_address(level) 打印出一个函数的堆栈地址。其中 level 代表是堆栈中第几层调用地址，__builtin_return_address(0) 表示第一层调用地址，即当前函数，__builtin_return_address(1) 表示第二层。

#### callerAcceptsOptimizedReturn

callerAcceptsOptimizedReturn 在 arm 架构上的实现如下：

```cpp
static ALWAYS_INLINE bool 
callerAcceptsOptimizedReturn(const void *ra)
{
    // if the low bit is set, we're returning to thumb mode
    if ((uintptr_t)ra & 1) {
        // 3f 46          mov r7, r7
        // we mask off the low bit via subtraction
        // 16-bit instructions are well-aligned
        if (*(uint16_t *)((uint8_t *)ra - 1) == 0x463f) {
            return true;
        }
    } else {
        // 07 70 a0 e1    mov r7, r7
        // 32-bit instructions may be only 16-bit aligned
        if (*(unaligned_uint32_t *)ra == 0xe1a07007) {
            return true;
        }
    }
    return false;
}
```

关于这段代码的含义请看这篇文章，总结的比我好：[How does objc_retainAutoreleasedReturnValue work?](https://www.galloway.me.uk/2012/02/how-does-objc_retainautoreleasedreturnvalue-work/)

### objc_retainAutoreleasedReturnValue

objc_retainAutoreleasedReturnValue 实现如下，基本就是 objc_autoreleaseReturnValue 的逆操作。

```cpp
// Accept a value returned through a +0 autoreleasing convention for use at +1.
id
objc_retainAutoreleasedReturnValue(id obj)
{
    if (acceptOptimizedReturn() == ReturnAtPlus1) return obj;

    return objc_retain(obj);
}
```

## __weak 修饰符

就像前面我们看到的一样，__weak 修饰符提供的功能如同魔法一般。

- 若附有 __weak 修饰符的变量所引用的对象被废弃，则将 nil 赋值给该变量。
- 使用附有 __weak 修饰符的变量，即是使用注册到 autoreleasepool 中的对象。

这些功能像魔法一样，到底发生了什么，我们一无所知。所以下面我们来看看它们的实现。

```objc
{
    id __weak obj1 = obj;
}
```

假设变量 obj 附加 __strong 修饰符且对象被赋值。

```objc
/* 编译器的模拟代码 */
id obj1;
objc_initWeak(&obj1, obj);
objc_destoryWeak(&obj1);
```

通过 objc_initWeak 函数初始化附有 __weak 修饰符的变量，在变量作用域结束时通过 objc_destroyWeak 函数释放该变量。

如以下源代码所示，objc_initWeak 函数将附有 __weak 修饰符的变量初始化为0后，会将赋值的对象作为参数调用 objc_storeWeak 函数。

```objc
obj1 = 0;
objc_storeWeak(&obj1, obj);
```

objc_destroyWeak 函数将0作为参数调用 objc_storeWeak 函数。

```objc
objc_storeWeak(&obj1, 0);
```

即前面的源代码与下列源代码相同。

```objc
id obj1;
obj1 = 0;
objc_storeWeak(&obj1, obj);
objc_storeWeak(&obj1, 0);
```

objc_storeWeak 函数把第二参数的赋值对象的地址作为键值，将第一参数的附有 __weak 修饰符的变量的地址注册到 weak 表中。如果第二参数为 0，则把变量的地址从 weak 表中删除。

weak 表与引用计数表相同，作为散列表被实现。如果使用 weak 表，将废弃对象的地址作为键值进行检索，就能高速地获取对应的附有 __weak 修饰符的变量的地址。另外，由于一个对象可同时赋值给多个附有 __weak 修饰符的变量中，所以对于一个键值，可注册多个变量的地址。

释放对象时，废弃谁都不持有的对象的同时，程序的动作是怎样的呢？下面我们来跟踪观察。对象将通过 objc_release 函数释放。

1. objc_release
2. 因为引用计数为 0 所以执行 dealloc
3. _objc_rootDealloc
4. object_dispose
5. objc_destructInstance
6. objc_clear_deallocating

对象被废弃时最后调用的 objc_clear_deallocating 函数的动作如下：

1. 将 weak 表中获取废弃对象的地址为键值的记录。
2. 将包含在记录中的所有附有 __weak 修饰符变量的地址，赋值为 nil。
3. 从 weak 表中删除改记录。
4. 从引用计数表中删除废弃对象的地址为键值的记录。

根据以上步骤，前面说的如果附有 __weak 修饰符的变量所引用的对象被废弃，则将 nil 赋值给该变量这一功能即被实现。由此可以，如果大量使用附有 __weak 修饰符的变量，则会消耗相应的 CPU 资源，良策是只在需要避免循环引用时使用 __weak 修饰符。

使用 __weak 修饰符时，以下源代码会引起编译器警告。

```objc
id __weak obj = [[NSObject alloc] init];
```

因为该源代码将自己生成并持有的对象赋值给附有 __weak 修饰符的变量中，所以自己不能持有该对象，这时会被释放并废弃，因此会引起编译器警告。

编译器会如何处理改源代码呢？

```objc
id obj;
id tmp = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(tmp, @selector(init))l
objc_initWeak(tmp);
objc_release(tmp);
objc_destroyWeak(&object);
```

虽然自己生成并持有对象通过 objc_initWeak 函数被赋值给附有 __weak 修饰符的变量中，但编译器判断其没有持有者，故该对象立即通过 objc_release 函数被释放和废弃。

这样一来，nil 就会被赋值给引用废弃对象的附有 __weak 修饰符的变量中。下面我们通过 NSLog 函数来验证一下。

```objc
id __weak obj = [[NSObject alloc] init];
NSLog(@"obj=%@", obj);
```

以下为该源代码的输出结果，其中用 %@ 输出 nil。

```objc
obj=(null)
```

这次我们再用附有 __weak 修饰符的变量来确认另一功能：使用附有 __weak 修饰符的变量，即是使用注册到 autoreleasepool 中的对象。

```objc
{
    id __weak obj1 = obj;
    NSLog(@"%@", obj1);
}
```

该源代码可转换为如下形式：

```objc
/* 编译器的模拟代码 */
id objl
objc_initWeak(&obj1, obj);
id tmp = objc_loadWeakRetained(&obj1);
objc_autorelease(tmp);
NSLog(@"%@", tmp);
objc_destroyWeak(&obj1);
```

与被赋值时相比，在使用附有 __weak 修饰符变量的情形下，增加了对 objc_loadWeakRetained 函数和 objc_autorelease 函数的调用。这些函数的动作如下：

1. objc_loadWeakRetained 函数取出附有 __weak 修饰符变量所引用的对象并 retain。
2. objc_autorelease 函数将对象注册到 autoreleasepool 中。

由此可知，因为附有 __weak 修饰符变量缩阴用的对象像这样被注册到 autoreleasepool 中，所以在 @autoreleasepool 块结束之前都可以放心使用。但是，如果大量地使用附有 __weak 修饰符的变量，注册到 autorelease 的对象也会大量地增加，因此在使用附有 __weak 修饰符的变量时，最好先暂时赋值给附有 __weak 修饰符的变量后再使用。

```objc
{
    id __weak o = obj;
    NSLog(@"1 %@", o);
    NSLog(@"2 %@", o);
    NSLog(@"3 %@", o);
    NSLog(@"4 %@", o);
    NSLog(@"5 %@", o);
}
```

相应地，变量 o 所赋值的对象也就注册到 autoreleasepool 中 5 次。

将附有 __weak 修饰符的变量 o 赋值给附有 __strong 修饰符的变量后再使用可以避免此类问题。

```objc
{
    id __weak o = obj;
    id tmp = o;
    NSLog(@"1 %@", tmp);
    NSLog(@"2 %@", tmp);
    NSLog(@"3 %@", tmp);
    NSLog(@"4 %@", tmp);
    NSLog(@"5 %@", tmp);
}
```

在 "tmp = 0;" 时对象仅登录到 autoreleasepool 中1次。

在 iOS 4 和 OS X Snow Leopard 中是不能使用 __weak 修饰符的，而有时在其他环境下也不能使用。实际上存在着不支持 __weak 修饰符的类。

例如 NSMachPort 类就是不支持 __weak 修饰符的类。这些类重写了 retain/release 并实现该类肚子的引用计数机制。但是赋值以及使用附有 __weak 修饰符的变量都必须恰当地使用 objc4 运行时库中的函数，因此独自实现引用计数机制的类大多不支持 __weak 修饰符。

不支持 __weak 修饰符的类，在其类声明中附加了 "__attribute ((objc_arc_weak_reference_unavailable))" 这一属性，同时定义了 NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE。如果将不支持 __weak 声明类的对象赋值给附有 __weak 修饰符的变量，那么一旦编译器检验出来就会报告编译错误。而且在 Cocoa 框架类中，不支持 __weak 修饰符的类极为罕见，因此没有必要太过担心。

### objc_initWeak

objc_initWeak 方法的实现如下：

```objc
/** 
 * Initialize a fresh weak pointer to some object location. 
 * It would be used for code like: 
 *
 * (The nil case) 
 * __weak id weakPtr;
 * (The non-nil case) 
 * NSObject *o = ...;
 * __weak id weakPtr = o;
 * 
 * This function IS NOT thread-safe with respect to concurrent 
 * modifications to the weak variable. (Concurrent weak clear is safe.)
 *
 * @param location Address of __weak ptr. 
 * @param newObj Object ptr. 
 */
id
objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
```

### objc_destroyWeak

objc_destroyWeak 实现如下：

```objc
/** 
 * Destroys the relationship between a weak pointer
 * and the object it is referencing in the internal weak
 * table. If the weak pointer is not referencing anything, 
 * there is no need to edit the weak table. 
 *
 * This function IS NOT thread-safe with respect to concurrent 
 * modifications to the weak variable. (Concurrent weak clear is safe.)
 * 
 * @param location The weak pointer address. 
 */
void
objc_destroyWeak(id *location)
{
    (void)storeWeak<DoHaveOld, DontHaveNew, DontCrashIfDeallocating>
        (location, nil);
}
```

### objc_storeWeak

```objc
/** 
 * This function stores a new value into a __weak variable. It would
 * be used anywhere a __weak variable is the target of an assignment.
 * 
 * @param location The address of the weak pointer itself
 * @param newObj The new object this weak ptr should now point to
 * 
 * @return \e newObj
 */
id
objc_storeWeak(id *location, id newObj)
{
    return storeWeak<DoHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object *)newObj);
}
```

#### storeWeak 方法实现如下

看了上述代码我们可以知道，objc_initWeak、objc_destroyWeak 和 objc_storeWeak 三个方法本质上都调用了 storeWeak 方法，其实现如下

```cpp
template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
    assert(haveOld  ||  haveNew);
    if (!haveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    if (haveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread 
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // Clean up old value, if any.
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any.
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```

## __unsafe_unretained

如前所述，以下源代码会引起编译警告。

```objc
id __weak obj = [[NSObject alloc] init];
```

这是由于编译器判断生成并持有的对象不能继续持有。附有 __unsafe_unretained 修饰符的变量又如何呢？

```objc
id __unsafe_unretained obj = [[NSObject alloc] init];
```

与 __weak 修饰符完全相同，编译器判断生成并持有的对象不能继续持有，从而发出警告。

该源代码通过编译器转换为以下形式。

```objc
/* 编译器的模拟代码 */
id obj = obj_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));
objc_release(obj);
```

objc_release 函数立即释放了生成并持有的对象，这样该对象的悬垂指针被赋值给变量 obj 中。

那么如果最初不赋值变量又会如何呢？下面的源代码在 ARC 无效时必定会发生内存泄漏。

```objc
[[NSObject alloc] init];
```

由于源代码不使用返回值的对象，所以编译器发出警告。

可像下面这样通过向 void 型转换来避免发生警告。

```objc
(void)[[NSObject alloc] init];
```

不管是否转换为 void，该源代码都会转换为以下形式。

```objc
/* 编译器的模拟代码 */
id tmp = obj_msgSend(NSObject, @selector(alloc));
objc_msgSend(tmp, @selector(init));
objc_release(tmp);
```

虽然没有指定赋值变量，但与赋值给附有 __unsafe_unretained 修饰符变量的源代码完全相同。由于不能继续持有生成并持有的对象，所以编译器生成了立即调用 objc_release 函数的源代码。而由于 ARC 的处理，这样的源代码也不会造成内存泄露。

另外，能调用被立即释放的对象的示例方法吗？

```objc
(void)[[[NSObject alloc] init] hash];
```

该源代码可变为如下形式：

```
/* 编译器的模拟代码 */
id tmp = obj_msgSend(NSObject, @selector(alloc));
objc_msgSend(tmp, @selector(init));
objc_msgSend(tmp, @selector(hash));
objc_release(tmp);
```

在调用了生成并持有对象的实例方法后，该对象被释放。看来“由编译器进行内存管理”这句话应该是正确的。

### allowsWeakReference/retainWeakReference 方法

实际上还有一种情况也不能使用 __weak 修饰符。

就是当 allowsWeakReference/retainWeakReference 实例方法（没有写入 NSObject 接口说明文档中）返回 NO 的情况。这些方法的声明如下：

```objc
- (BOOL)allowsWeakReference;
- (BOOL)retainWeakReference;
```

在赋值给 __weak 修饰符的变量时，如果赋值对象的 allowsWeakReference 方法返回 NO，程序将异常终止。

即对于所有 allowsWeakReference 方法返回 NO 的类都绝对不能使用 __weak 修饰符。这样的类必定在其参考说明中有所记述。

另外，在使用 __weak 修饰符的变量时，当被赋值对象的 retainWeakReference 方法返回 NO 的情况下，该变量将使用“nil”。如以下的源代码：

```objc
{
    id __strong obj = [[NSObject alloc] init];
    id __weak o = obj;
    NSLog(@“1 %@”, o);
    NSLog(@“2 %@”, o);
    NSLog(@“3 %@”, o);
    NSLog(@“4 %@”, o);
    NSLog(@“5 %@”, o);
}
```

由于最开始生成并持有的对象为附有 __strong 修饰符变量 obj 所持有的强引用，所以在该变量作用域结束之前都始终存在。因此如下所示，在变量作用域结束之前，没可以持续使用附有 __weak 修饰符的变量 o 所引用的对象。

下面对 retainWeakReference 方法进行实验。我们做一个 MyObject 类，让其继承 NSObject 类并实现 retainWeakReference 方法。

```objc
@interface MyObject : NSObject
{
    NSUInteger count;
}
@end

@implementation MyObject

- (id)init
{
    self = [super init];
    return self;
}

- (BOOL)retainWeakReference
{
    if (++count > 3)
        return NO;
    return [super retainWeakReference];
}
@end
```

该例中，当 retainWeakReference 方法被调用4次或4次以上时返回 NO。在之前的源代码中，将从 NSObject 类生成并持有对象的部分更改为 MyObject 类。

```objc
{
    id __strong obj = [[MyObject alloc] init];
    id __weak o = obj;
    NSLog(@"1 %@", o);
    NSLog(@"2 %@", o);
    NSLog(@"3 %@", o);
    NSLog(@"4 %@", o);
    NSLog(@"5 %@", o);
}
```

以下为执行结果。

从第4次起，使用附有 __weak 修饰符的变量 o 时，由于所饮用对象的 retainWeakReference 方法返回 NO，所以无法获取对象。像这样的类也必定在其参考说明中有所记述。

另外，运行时库为了操作 __weak 修饰符在执行过程中调用 allowsWeakReference/retainWeakReference 方法，因此从该方法中再次操作运行时库时，其操作内容会永久等待。原本这些方法并没有计入文档，因此应用程序编程人员不可能实现该方法群，但如果因某些原因而不得不实现，那么还是在全部理解的基础上实现比较好。

## __autoreleasing 修饰符

将对象赋值给附有 __autoreleasing 修饰符的变量等同于 ARC 无效时调用对象的 autorelease 方法。我们通过以下源代码来看一下。

```objc
@autoreleasepool {
    id __autoreleasing obj = [[NSObject alloc] init];
}
```

该源代码主要将 NSObject 类对象注册到 autoreleasepool 中，可作如下变换：

```objc
/* 编译器的模拟代码 */
id pool = objc_autoreleasePoolPush();
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));
objc_autorelease(obj);
objc_autoreleasePoolPop(pool);
```

这与苹果的 autorelease 实现中的说明完全相同。虽然 ARC 有效和无效时，其在源代码上的表现有所不同，但 autorelease 的功能完全一样。

在 alloc/new/copy/mutableCopy 方法群之外的方法中使用注册到 autoreleasepool 中的对象会如何呢？下面我们来看看 NSMutableArray 类的 array 类方法。

```objc
@autoreleasepool {
    id __autoreleasing obj = [NSMutableArray array];
}
```

这与前面的源代码有何不同呢？

```objc
/* 编译器的模拟代码 */
id pool = objc_autoreleasePoolPush();
id obj = objc_msgSend(NSMutableArray, @selector(array));
objc_retainAutoreleasedReturnValue(obj);
objc_autorelease(obj);
objc_autoreleasePoolPop(pool);
```

虽然持有对象的方法从 alloc 方法变为 objc_retainAutoreleasedReturnValue 函数，但注册 autoreleasepool 的方法没有改变，仍是 objc_autorelease 函数。