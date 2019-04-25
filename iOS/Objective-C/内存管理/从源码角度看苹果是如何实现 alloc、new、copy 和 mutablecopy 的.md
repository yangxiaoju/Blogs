# 从源码角度看苹果是如何实现 alloc、new、copy 和 mutablecopy 的

在了解了如何在 MRC 和 ARC 两种不同的环境下管理我们的内存之后，接下来让我们从源码角度看一下，苹果是如何实现 MRC 环境中的内存管理的相关方法的。

源码下载地址：[https://opensource.apple.com/tarballs/objc4/](https://opensource.apple.com/tarballs/objc4/)

## alloc

先来看 alloc 方法在 NSObject 类中的实现：

```objc
+ (id)alloc {
    return _objc_rootAlloc(self);
}
```

简单的调用了 _objc_rootAlloc 函数，实现如下：

```cpp
// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
```

_objc_rootAlloc 则调用了 callAlloc 函数，callAlloc 实现如下：

```cpp
// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (slowpath(checkNil && !cls)) return nil; // 如果 checkNil 为 ture，且 cls 为 nil，直接返回 nil。

#if __OBJC2__
    if (fastpath(!cls->ISA()->hasCustomAWZ())) { // 如果 cls 没有实现自定义 allocWithZone 方法
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        if (fastpath(cls->canAllocFast())) { // 如果 cls 支持快速 Alloc
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor(); // 获取 cls 是否有自己的析构函数。
            id obj = (id)calloc(1, cls->bits.fastInstanceSize()); // 使用 calloc 根据 fastInstanceSize 的大小申请内存空间
            if (slowpath(!obj)) return callBadAllocHandler(cls); // 如果内存空间申请失败，调用 callBadAllocHandler
            obj->initInstanceIsa(cls, dtor); // 如果成功，初始化 isa
            return obj; // 返回对象
        }
        else { // 如果 cls 不支持快速 Alloc
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);  // 通过 class_createInstance 方法直接创建对象
            if (slowpath(!obj)) return callBadAllocHandler(cls); // 如果对象创建失败，调用 callBadAllocHandler
            return obj; // 返回对象
        }
    }
#endif

    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];  // 如果 allocWithZone 为 true，调用 allocWithZone 创建对象
    return [cls alloc]; // 否则调用 alloc 方法创建对象
}
```

整个方法的实现和代码解读已经写在注释里了，这里面有很多知识点值得讲一讲：

### ALWAYS_INLINE

inline 是一种降低函数调用成本的方法，其本质是在调用声明为 inline 的函数时，会直接把函数的实现替换过去，这样减少了调用函数的成本。当然 inline 是一种以空间换时间的做法，滥用 inline 会导致应用程序的体积增大，所以有的时候编译器未必会真的按照你声明 inline 的方式去用函数的实现替换函数的调用。

ALWAYS_INLINE 宏如其名，会强制开启 inline，其实现如下：

```cpp
#define ALWAYS_INLINE inline __attribute__((always_inline))
```

### slowpath & fastpath

这两个宏的实现如下：

```cpp
#define slowpath(x) (__builtin_expect(bool(x), 0))
#define fastpath(x) (__builtin_expect(bool(x), 1))
```
这两个宏都使用了一个叫做 __builtin_expect 的函数：

```cpp
long __builtin_expect (long EXP, long C)  
```

它的返回值就是整个函数的返回值，参数 C 代表预计的值，表示程序员知道 EXP 的值很可能就是 C。

因此，在苹果定义的两个宏中，fastpath(x) 依然返回 x，只是告诉编译器 x 的值一般不为 0，从而编译器可以进行优化。同理，slowpath(x) 表示 x 的值很可能为 0，希望编译器进行优化。

因为计算机每次不会只读取一条语句，所以上述两个宏在 if 条件判断中很有用，有助于编译器优化代码，以减少指令重读。

### __OBJC2__

用于判断当前使用的语言是否是 Objective-C 2.0

### 如何获取类对象的 isa

在之前的方法中，cls 的 isa 是通过 cls->ISA() 获取的，在解读方法的实现之前，先来学习一下 SUPPORT_NONPOINTER_ISA 宏的含义。

```cpp
// Define SUPPORT_NONPOINTER_ISA=1 on any platform that may store something
// in the isa field that is not a raw pointer.
#if !SUPPORT_INDEXED_ISA  &&  !SUPPORT_PACKED_ISA
#   define SUPPORT_NONPOINTER_ISA 0
#else
#   define SUPPORT_NONPOINTER_ISA 1
#endif
```

SUPPORT_NONPOINTER_ISA 用于标记是否支持优化的 isa 指针，其字面含义意思是 isa 的内容不再是类的指针了，而是包含了更多信息，比如引用计数，析构状态，被其他 weak 变量引用情况。而 SUPPORT_NONPOINTER_ISA 宏的定义又涉及到了其他两个宏：SUPPORT_INDEXED_ISA 和 SUPPORT_PACKED_ISA：

```cpp
// Define SUPPORT_INDEXED_ISA=1 on platforms that store the class in the isa 
// field as an index into a class table.
// Note, keep this in sync with any .s files which also define it.
// Be sure to edit objc-abi.h as well.
#if __ARM_ARCH_7K__ >= 2  ||  (__arm64__ && !__LP64__)
#   define SUPPORT_INDEXED_ISA 1
#else
#   define SUPPORT_INDEXED_ISA 0
#endif

// Define SUPPORT_PACKED_ISA=1 on platforms that store the class in the isa 
// field as a maskable pointer with other data around it.
#if (!__LP64__  ||  TARGET_OS_WIN32  ||  \
     (TARGET_OS_SIMULATOR && !TARGET_OS_IOSMAC))
#   define SUPPORT_PACKED_ISA 0
#else
#   define SUPPORT_PACKED_ISA 1
#endif
```

从注释上来看，SUPPORT_INDEXED_ISA 的含义是，如果为 1，则会在 isa field 中存储 class 在 class table 的索引。SUPPORT_PACKED_ISA 为 1 时则是在 isa field 中通过 mask 的方式存储 class 的指针信息。

接下来让我们来看一下 ISA 方法的实现：

```cpp
inline Class 
objc_object::ISA() 
{
    assert(!isTaggedPointer());  // 要确保该类不是 tagged pointer
#if SUPPORT_INDEXED_ISA
    if (isa.nonpointer) { // 如果 isa 是非指针型 isa
        uintptr_t slot = isa.indexcls; // 获取 class 索引
        return classForIndex((unsigned)slot); // 从 class table 中查找并返回 isa 信息
    }
    return (Class)isa.bits; // 直接返回 isa.bits
#else
    return (Class)(isa.bits & ISA_MASK); // 通过掩码获取 bits 中对应的信息并返回
#endif
}
```

### 如何判断类是否实现了 allocWithZone 方法

hasCustomAWZ 方法的实现如下

```cpp
bool hasCustomAWZ() {
    return ! bits.hasDefaultAWZ();
}
```

hasDefaultAWZ 方法的实现如下：

```cpp
bool hasDefaultAWZ() {
    return data()->flags & RW_HAS_DEFAULT_AWZ;
}
```

而 data 方法的实现是：

```cpp
class_rw_t* data() {
    return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```

总而言之就是从 bits 中根据位掩码 FAST_DATA_MASK 获取 data，再根据位掩码 RW_HAS_DEFAULT_AWZ 获取是否是默认的 allocWithZone 方法。

### 如何判断类是否支持快速 Alloc

canAllocFast 使用来获取类是否支持快速 alloc 的方法，其实现如下：

```cpp
bool canAllocFast() {
    assert(!isFuture());
    return bits.canAllocFast();
}
```

我们先来看 isFuture 的实现：

```objc
// Returns true if this is an unrealized future class.
// Locking: To prevent concurrent realization, hold runtimeLock.
bool isFuture() { 
    return data()->flags & RW_FUTURE;
}
```

bits 的 canAllocFast 方法实现如下，同样是通过位掩码 FAST_ALLOC 从 bits 中获取信息：

```cpp
// summary bit for fast alloc path: !hasCxxCtor and 
//   !instancesRequireRawIsa and instanceSize fits into shiftedSize
#define FAST_ALLOC              (1UL<<2)

#if FAST_ALLOC
    bool canAllocFast() {
        return bits & FAST_ALLOC;
    }
#else
    bool canAllocFast() {
        return false;
    }
#endif
```

### hasCxxDtor

hasCxxDtor 方法的作用是获取当前 Class 是否有自己的析构函数。hasCxxDtor 方法用来获取类及父类是否有自己的析构函数，与对象是否有实例变量有关，会记录在对象的 isa 内。实现如下：

```cpp
bool hasCxxDtor() {
    // addSubclass() propagates this flag from the superclass.
    assert(isRealized());
    return bits.hasCxxDtor();
}
```

isRealized 类似与 isFuture，实现如下：

```objc
// Locking: To prevent concurrent realization, hold runtimeLock.
bool isRealized() {
    return data()->flags & RW_REALIZED;
}
```

bits 的 hasCxxDtor 方法实现如下：

```cpp
// class or superclass has .cxx_destruct implementation
#define FAST_HAS_CXX_DTOR       (1UL<<51)

#if FAST_HAS_CXX_DTOR
bool hasCxxDtor() {
    return getBit(FAST_HAS_CXX_DTOR);
}
#else
bool hasCxxDtor() {
    return data()->flags & RW_HAS_CXX_DTOR;
}
#endif
```

### 为对象申请内存空间

接着，通过 calloc 函数，用来动态地分配内存空间并初始化为 0。calloc 函数原型如下：

```cpp
void *calloc(size_t __count, size_t __size)
```

calloc 在内存中动态地分配 num 个长度为 size 的连续空间，并将每一个字节都初始化为 0。所以它的结果是分配了 num * size 个字节长度的内存空间，并且每个字节的值都是0。分配成功返回指向该内存的地址，失败则返回 NULL。

cls->bits.fastInstanceSize() 是用来获取实例占用存储空间的方法，实现如下：

```cpp
size_t fastInstanceSize() 
{
    assert(bits & FAST_ALLOC);
    return (bits >> FAST_SHIFTED_SIZE_SHIFT) * 16;
}
```

接着会判断对象是否创建成功，如果没能创建成功会调用 callBadAllocHandler(cls)，

```objc
static id defaultBadAllocHandler(Class cls)
{
    _objc_fatal("attempt to allocate object of class '%s' failed", 
                cls->nameForLogging());
}

static id(*badAllocHandler)(Class) = &defaultBadAllocHandler;

static id callBadAllocHandler(Class cls)
{
    // fixme add re-entrancy protection in case allocation fails inside handler
    return (*badAllocHandler)(cls);
}
```

### 初始化 isa 

如果对象创建成功，会使用 initInstanceIsa 为其初始化 isa，代码如下：

```cpp
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    assert(!cls->instancesRequireRawIsa());
    assert(hasCxxDtor == cls->hasCxxDtor());

    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    assert(!isTaggedPointer()); 
    
    if (!nonpointer) {
        isa.cls = cls; // 如果 nonpointer 为 false，则直接把 cls 赋值给 cls
    } else {
        assert(!DisableNonpointerIsa);
        assert(!cls->instancesRequireRawIsa());

        isa_t newisa(0); // 初始化新的 isa

#if SUPPORT_INDEXED_ISA
        assert(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE; // 对 isa 中的一些值进行初始化
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
#endif

        // This write must be performed in a single store in some cases
        // (for example when realizing a class because other threads
        // may simultaneously try to use the class).
        // fixme use atomics here to guarantee single-store and to
        // guarantee memory order w.r.t. the class index table
        // ...but not too atomic because we don't want to hurt instantiation
        isa = newisa; // 将 newisa 赋值给 isa
    }
}
```

### class_createInstance

如果类不支持 AllocFast，则需要通过 class_createInstance 方法进行对象的创建，方法的实现如下：

```cpp
id 
class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}

/***********************************************************************
* class_createInstance
* fixme
* Locking: none
**********************************************************************/

static __attribute__((always_inline)) 
id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, 
                              bool cxxConstruct = true, 
                              size_t *outAllocatedSize = nil)
{
    if (!cls) return nil;

    assert(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cls->hasCxxCtor(); // 获取 cls 及其父类是否有构造函数
    bool hasCxxDtor = cls->hasCxxDtor(); // 获取 cls 及其父类是否有析构函数
    bool fast = cls->canAllocNonpointer(); // 是对 isa 的类型的区分，如果一个类和它父类的实例不能使用 isa_t 类型的 isa 的话，返回值为 false，但是在 Objective-C 2.0 中，大部分类都是支持的

    size_t size = cls->instanceSize(extraBytes); // 获取需要申请的空间大小
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (!zone  &&  fast) { // 如果 zone 参数为空，且支持 fast，通过 calloc 申请空间并初始化 isa
        obj = (id)calloc(1, size);
        if (!obj) return nil;
        obj->initInstanceIsa(cls, hasCxxDtor);
    } 
    else {
        if (zone) { // 如果 zone 不为空，使用 malloc_zone_calloc 方法申请空间
            obj = (id)malloc_zone_calloc ((malloc_zone_t *)zone, 1, size);
        } else { // 如果 zone 为空，使用 calloc 方法申请空间
            obj = (id)calloc(1, size);
        }
        if (!obj) return nil;

        // Use raw pointer isa on the assumption that they might be 
        // doing something weird with the zone or RR.
        obj->initIsa(cls); // 初始化 isa
    }

    if (cxxConstruct && hasCxxCtor) {
        obj = _objc_constructOrFree(obj, cls); // 构建对象
    }

    return obj;
}
```

### allocWithZone

如果是非 Objective-C 2.0 的代码，且 allocWithZone 为 true，会通过 allocWithZone 方法初始化对象，其实现如下所示：

```objc
// Replaced by ObjectAlloc
+ (id)allocWithZone:(struct _NSZone *)zone {
    return _objc_rootAllocWithZone(self, (malloc_zone_t *)zone);
}

id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone)
{
    id obj;

#if __OBJC2__
    // allocWithZone under __OBJC2__ ignores the zone parameter
    (void)zone;
    obj = class_createInstance(cls, 0);
#else
    if (!zone) {
        obj = class_createInstance(cls, 0);
    }
    else {
        obj = class_createInstanceFromZone(cls, 0, zone);
    }
#endif

    if (slowpath(!obj)) obj = callBadAllocHandler(cls);
    return obj;
}
```

## new

再讲完 alloc 后，new 的实现就很简单了，如下所示：

```cpp
+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
```

就是嵌套调用了 alloc 和 init。

## copy & mutableCopy

copy 和 mutableCopy 的实现就很简单了，它们只是调用了 NSCopying 协议中的方法：

```cpp
- (id)copy {
    return [(id)self copyWithZone:nil];
}

- (id)mutableCopy {
    return [(id)self mutableCopyWithZone:nil];
}
```

至此，关于 alloc、new、copy 和 mutableCopy 方法的实现就已经分析完了，关于类的一些细节，可以参考扩展阅读里的文章进行更深一步的学习。

扩展阅读：

- [用 isa 承载对象的类信息](https://www.desgard.com/isa/)