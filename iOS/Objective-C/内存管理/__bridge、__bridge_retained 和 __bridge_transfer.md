# __bridge、__bridge_retained 和 __bridge_transfer

> 本文为 《Objective-C 高级编程》1.3.4 中的部分内容

在 ARC 无效时，像以下代码这样将 id 变量强制转换 void * 变量并不会出问题。

```objc
/* ARC 无效 */

id obj = [[NSObject alloc] init];

void *p = obj;
```

更进一步，将该 void * 变量赋值给 id 变量中，调用其实例方法，运行时也不会有问题。

```objc
/* ARC 无效 */

id o = p;

[o release];
```

但是在 ARC 有效时这便会引起编译错误。

id 型或对象型变量赋值给 void * 或者逆向赋值时都需要进行特定的转换。如果只想单纯地赋值，则可以使用“__bridge 转换”。

```objc
id obj = [[NSObject alloc] init];

void *p = (__bridge void *)obj;

id o = (__bridge id)p;
```

像这样，通过“__bridge 转换”，id 和 void * 就能够相互转换。

但是转换为 void * 的 __bridge 转换，其安全性与赋值给 __unsafe_unretained 修饰符相近，甚至会更低。如果管理时不注意赋值对象的所有者，就会因悬垂指针而导致程序崩溃。

__bridge 转换中还有另外两种转换，分别是“__bridge_retained 转换”和“__bridge_transfer 转换”

```objc
id obj = [[NSObject alloc] init];

void *p = (__bridge_retained void *)obj;
```

__bridge_retained 转换可使要转换赋值的变量也持有所赋值的对象。下面我们来看 ARC 无效时源代码是如何编写的。

```objc
/* ARC 无效 */

id obj = [[NSObject alloc] init];

void *p = obj;

[(id)p retain];
```

__bridge_retained 转换变为了 retain。变量 obj 和变量 p 同时持有对象。再来看几个其他的例子。

```objc
void *p = 0;

{
    id obj = [[NSObject alloc] init];

    p = (__bridge_retained void *)obj;
}

NSLog(@"class=%@", [(__bridge id)p class]);
```

变量作用域结束时，虽然随着持有强引用的变量 obj 失效，对象随之释放，但由于 __bridge_retained 转换使变量 p 看上去处于持有该对象的状态，因此该对象不会被废弃。下面我们比较下 ARC 无效时的代码是怎样的。

```objc
void *p = 0;

{
    id obj = [[NSObject alloc] init];
    /* [obj retainCount] -> 1 */

    p = [obj retain];
    /* [obj retainCount] -> 2 */

    [obj release];
    /* [obj retainCount] -> 1 */
}

/*
 * [(id)p retainCount] -> 1 即 [obj retainCount] -> 1 对象仍存在
 */

NSLog(@"class=%@", [(__bridge id)p class]);
```

__bridge_transfer 转换提供与此相反的动作，被转换的变量所持有的对象在该变量被赋值给转换目标变量后随之释放。

```objc
id obj = (__bridge_transfer id)p;
```

该源代码在 ARC 无效时又如何表达呢？

```objc
/* ARC 无效 */

id obj = (id)p;
[obj retain];
[(id)p release];
```

同 __bridge_retained 转换与 retain 类似，__bridge_transfer 转换与 release 相似。在给 id obj 赋值时 retain 即相当于 __strong 修饰符的变量。

如果使用以上两种转换，那么不使用 id 型或对象型变量也可以生成、持有以及释放对象。虽然可以这样做，但在 ARC 中不推荐这种方法。使用时还请注意。

当然，也可以使用 CFBridgingRetain 函数来代替 __bridge_retained，使用 CFBridgingRelease 来代替 __bridge_transfer。选用自己更熟悉的方法即可。