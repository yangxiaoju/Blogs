# 从源码角度看苹果是如何实现 autorelease 和 autoreleasepool 的

书接上文，接下来让我们了解一下，苹果是如何实现 autorelease 和 autoreleasepool 的

## autorelease

autorelease 方法的作用是延迟对象的 release，通常用于返回值时使用，如下是它的实现：

```cpp
// Replaced by ObjectAlloc
- (id)autorelease {
    return ((id)self)->rootAutorelease();
}

// Base autorelease implementation, ignoring overrides.
inline id 
objc_object::rootAutorelease()
{
    if (isTaggedPointer()) return (id)this; // 如果是 tagged pointer，直接返回 this
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this; // 如果 prepareOptimizedReturn(ReturnAtPlus1) 返回 true，直接返回 this

    return rootAutorelease2(); // 调用 rootAutorelease2
}
```

### prepareOptimizedReturn

prepareOptimizedReturn 是用于优化返回的，其实现如下

```cpp
// Try to prepare for optimized return with the given disposition (+0 or +1).
// Returns true if the optimized path is successful.
// Otherwise the return value must be retained and/or autoreleased as usual.
static ALWAYS_INLINE bool 
prepareOptimizedReturn(ReturnDisposition disposition)
{
    assert(getReturnDisposition() == ReturnAtPlus0); // 确保当前获取的 Disposition 为 false

    if (callerAcceptsOptimizedReturn(__builtin_return_address(0))) {  // 如果当前允许优化返回值
        if (disposition) setReturnDisposition(disposition); // 设置 ReturnDisposition 为 true
        return true;
    }

    return false;
}
```

#### ReturnDisposition & setReturnDisposition

ReturnDisposition & setReturnDisposition 是用于获取和设置 Disposition 用的函数，实现如下

```cpp
static ALWAYS_INLINE ReturnDisposition 
getReturnDisposition()
{
    return (ReturnDisposition)(uintptr_t)tls_get_direct(RETURN_DISPOSITION_KEY);
}

static ALWAYS_INLINE void 
setReturnDisposition(ReturnDisposition disposition)
{
    tls_set_direct(RETURN_DISPOSITION_KEY, (void*)(uintptr_t)disposition);
}
```

tls，全程为 Thread Local Storage，是在当前线程存储一些数据用的，在这里是通过 RETURN_DISPOSITION_KEY 这个 key 进行存储的，其实现如下：

```cpp
typedef pthread_key_t tls_key_t;

#if defined(__PTK_FRAMEWORK_OBJC_KEY0)
#   define SUPPORT_DIRECT_THREAD_KEYS 1
#   define TLS_DIRECT_KEY        ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY0)
#   define SYNC_DATA_DIRECT_KEY  ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY1)
#   define SYNC_COUNT_DIRECT_KEY ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY2)
#   define AUTORELEASE_POOL_KEY  ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY3)
# if SUPPORT_RETURN_AUTORELEASE
#   define RETURN_DISPOSITION_KEY ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY4)
# endif
#else
#   define SUPPORT_DIRECT_THREAD_KEYS 0
#endif

```

#### __builtin_return_address

gcc 的编译特性使用 __builtin_return_address(level) 打印出一个函数的堆栈地址。其中 level 代表是堆栈中第几层调用地址，__builtin_return_address(0) 表示第一层调用地址，即当前函数，__builtin_return_address(1) 表示第二层。

#### callerAcceptsOptimizedReturn

callerAcceptsOptimizedReturn 是用来判断当前返回是否可以优化的函数，其在不同架构上的实现不同，我暂时没搞懂这段代码到底是什么意思，等我哪天想清楚了再补上这个坑吧，先贴上其在 arm 架构下的实现：

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

### rootAutorelease2

如果之前的一系列优化没能返回 this，最后会调用 rootAutorelease2 这个方法，其实现如下：

```objc
__attribute__((noinline,used))
id 
objc_object::rootAutorelease2()
{
    assert(!isTaggedPointer());
    return AutoreleasePoolPage::autorelease((id)this);
}
```

我们会发现其实现相当简单，接下来就是 autorelease 的重头戏，autoreleasepool 究竟是怎么实现的。

## autoreleasepool

想要彻底了解 autoreleasepool 的实现，不是三言两语可以说清楚的，让我们接着之前的 AutoreleasePoolPage::autorelease((id)this); 来一步步分析 autoreleasepool 是如何实现的：

首先需要了解的一点是，所有 autorelease 相关的方法，最终都是通过 AutoreleasePoolPage 这个类来实现的，其 autorelease 方法实现如下：

```cpp
static inline id autorelease(id obj)
{
    assert(obj); // 对象不可为 NULL
    assert(!obj->isTaggedPointer()); // 对象不可为 tagged pointer
    id *dest __unused = autoreleaseFast(obj); 
    assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj); // 确保 dest 不存在或等于 EMPTY_POOL_PLACEHOLDER 或等于 obj
    return obj;
}
```

### autoreleaseFast

autoreleaseFast 的实现如下：

```cpp
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage(); // 获取 hotPage
    if (page && !page->full()) { // 如果 page 存在且未满
        return page->add(obj); // 将对象添加到 page 中
    } else if (page) { // 如果 page 存在，但已经满了。
        return autoreleaseFullPage(obj, page); // 调用 autoreleaseFullPage 方法
    } else { // 如果 page 为 NULL
        return autoreleaseNoPage(obj); // 调用 autoreleaseNoPage 方法
    }
}
```

到这里我们可以大致了解到，所谓的 autoreleasepool 其实是由一个或多个 AutoreleasePoolPage 对象组成的，这些对象有容量限制，让我们继续拆解这个方法。

#### hotPage

hotPage 方法是用来获取当前可用 page 的，其实现如下：

```cpp
static inline AutoreleasePoolPage *hotPage() 
{
    AutoreleasePoolPage *result = (AutoreleasePoolPage *)
        tls_get_direct(key); // 通过 tls 查询可用 AutoreleasePoolPage 对象
    if ((id *)result == EMPTY_POOL_PLACEHOLDER) return nil; // 如果查询结果为 EMPTY_POOL_PLACEHOLDER，返回 nil
    if (result) result->fastcheck(); // fastcheck
    return result;
}
```

这里的 key 定义如下：

```cpp
// Thread keys reserved by libc for our use.
#if defined(__PTK_FRAMEWORK_OBJC_KEY0)
#   define SUPPORT_DIRECT_THREAD_KEYS 1
#   define TLS_DIRECT_KEY        ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY0)
#   define SYNC_DATA_DIRECT_KEY  ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY1)
#   define SYNC_COUNT_DIRECT_KEY ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY2)
#   define AUTORELEASE_POOL_KEY  ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY3)
# if SUPPORT_RETURN_AUTORELEASE
#   define RETURN_DISPOSITION_KEY ((tls_key_t)__PTK_FRAMEWORK_OBJC_KEY4)
# endif
#else
#   define SUPPORT_DIRECT_THREAD_KEYS 0
#endif

static pthread_key_t const key = AUTORELEASE_POOL_KEY;
```

fastcheck 方法实现如下：

```cpp
void check(bool die = true) 
{
    if (!magic.check() || !pthread_equal(thread, pthread_self())) {
        busted(die);
    }
}

void fastcheck(bool die = true) 
{
#if CHECK_AUTORELEASEPOOL
    check(die);
#else
    if (! magic.fastcheck()) {
        busted(die);
    }
#endif
}
```

一般会做一些线程之类的检查，最终都会调用到 busted 方法，也就是实际做检查的地方，

```cpp
void busted(bool die = true) 
{
    magic_t right;
    (die ? _objc_fatal : _objc_inform)
        ("autorelease pool page %p corrupted\n"
        "  magic     0x%08x 0x%08x 0x%08x 0x%08x\n"
        "  should be 0x%08x 0x%08x 0x%08x 0x%08x\n"
        "  pthread   %p\n"
        "  should be %p\n", 
        this, 
        magic.m[0], magic.m[1], magic.m[2], magic.m[3], 
        right.m[0], right.m[1], right.m[2], right.m[3], 
        this->thread, pthread_self());
}
```

根据 die 参数不同会决定是 _objc_fatal 或 _objc_inform。

#### full

full 方法是用来检查 AutoreleasePoolPage 对象是否已经满了的方法，其实现如下：

```cpp
id * end() {
    return (id *) ((uint8_t *)this+SIZE);
}

bool full() { 
    return next == end();
}
```

看到此处，我们已经可以断定 AutoreleasePoolPage 对象的数据类型是链表类型了。

SIZE 的定义如下：

```cpp
#define I386_PGBYTES            4096
#define PAGE_SIZE               I386_PGBYTES
#define PAGE_MAX_SIZE           PAGE_SIZE

static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
```

#### add

```cpp
id *add(id obj)
{
    assert(!full()); // 确保当前状态不满
    unprotect(); // 设置为读/写状态
    id *ret = next;  // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect(); // 设置为只读状态
    return ret;
}
```

protect 和 unprotect 方法实现如下：

```cpp
inline void protect() {
#if PROTECT_AUTORELEASEPOOL
    mprotect(this, SIZE, PROT_READ);
    check();
#endif
}

inline void unprotect() {
#if PROTECT_AUTORELEASEPOOL
    check();
    mprotect(this, SIZE, PROT_READ | PROT_WRITE);
#endif
}
```

主要是使用了 mprotect 函数，mprotect 函数的原型如下：

```cpp
int mprotect(const void *addr, size_t len, int prot);
```

其中 addr 是待保护的内存首地址，必须按页对齐；len 是待保护内存的大小，必须是页的整数倍，prot 代表模式，可能的取值有 PROT_READ（表示可读）、PROT_WRITE（可写）等。

不同体系结构和操作系统，一页的大小不尽相同。如何获得页大小呢？通过 PAGE_SIZE 宏或者 getpagesize() 系统调用即可。

#### autoreleaseFullPage

```cpp
static __attribute__((noinline))
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
    // The hot page is full. 
    // Step to the next non-full page, adding a new page if necessary.
    // Then add the object to that page.
    assert(page == hotPage());
    assert(page->full()  ||  DebugPoolAllocation);

    do {
        if (page->child) page = page->child; // 如果 page->child 存在，则给 page 赋值 page->child
        else page = new AutoreleasePoolPage(page); // 否则就初始化一个新的 page。
    } while (page->full());

    setHotPage(page); // 设置 page 为当前 hotPage
    return page->add(obj); // 添加 obj 进入 page
}
```

AutoreleasePoolPage 的构造函数如下所示：

```cpp
AutoreleasePoolPage(AutoreleasePoolPage *newParent) 
    : magic(), next(begin()), thread(pthread_self()),
    parent(newParent), child(nil), 
    depth(parent ? 1+parent->depth : 0), 
    hiwat(parent ? parent->hiwat : 0)
{ 
    if (parent) {
        parent->check();
        assert(!parent->child);
        parent->unprotect();
        parent->child = this;
        parent->protect();
    }
    protect();
}
```

通过初始化方法我们可以看到，AutoreleasePoolPage 对象不仅有 child，还有 parent，由此可以断定，其为双向链表。

setHotPage 实现如下：

```cpp
static inline void setHotPage(AutoreleasePoolPage *page) 
{
    if (page) page->fastcheck();
    tls_set_direct(key, (void *)page);
}
```

### autoreleaseNoPage

```cpp
static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{
    // "No page" could mean no pool has been pushed
    // or an empty placeholder pool has been pushed and has no contents yet
    assert(!hotPage());

    bool pushExtraBoundary = false;
    if (haveEmptyPoolPlaceholder()) { // 如果当前有 poolPlaceholder
        // We are pushing a second pool over the empty placeholder pool
        // or pushing the first object into the empty placeholder pool.
        // Before doing that, push a pool boundary on behalf of the pool 
        // that is currently represented by the empty placeholder.
        pushExtraBoundary = true;
    }
    else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
        // We are pushing an object with no pool in place, 
        // and no-pool debugging was requested by environment.
        _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                    "autoreleased with no pool in place - "
                    "just leaking - break on "
                    "objc_autoreleaseNoPool() to debug", 
                    pthread_self(), (void*)obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return nil;
    }
    else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) { 
        // We are pushing a pool with no pool in place,
        // and alloc-per-pool debugging was not requested.
        // Install and return the empty pool placeholder.
        return setEmptyPoolPlaceholder(); // 设置占位符
    }

    // We are pushing an object or a non-placeholder'd pool.

    // Install the first page.
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);
        
    // Push a boundary on behalf of the previously-placeholder'd pool.
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY); // 添加哨兵对象
    }
        
    // Push the requested object or pool.
    return page->add(obj);
}
```

haveEmptyPoolPlaceholder 方法实现如下：

```cpp
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

static inline bool haveEmptyPoolPlaceholder()
{
    id *tls = (id *)tls_get_direct(key);
    return (tls == EMPTY_POOL_PLACEHOLDER);
}
```

setEmptyPoolPlaceholder 方法实现如下：

```cpp
static inline id* setEmptyPoolPlaceholder()
{
    assert(tls_get_direct(key) == nil);
    tls_set_direct(key, (void *)EMPTY_POOL_PLACEHOLDER);
    return EMPTY_POOL_PLACEHOLDER;
}
```