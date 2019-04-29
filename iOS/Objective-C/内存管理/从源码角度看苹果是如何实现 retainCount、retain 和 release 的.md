# 从源码角度看苹果是如何实现 retainCount、retain 和 release 的

在上一篇文章中，我们从源码上来看了下苹果是如何实现一系列方法的，这篇文章将继续分析 retainCount、retain 和 release 方法。

## retainCount

retainCount 是获取当前对象引用计数的方法，其实现如下：

```cpp
- (NSUInteger)retainCount {
    return ((id)self)->rootRetainCount();
}
```

直接调用了 rootRetainCount 方法，实现如下：

```cpp
inline uintptr_t 
objc_object::rootRetainCount()
{
    if (isTaggedPointer()) return (uintptr_t)this; // 如果是 tagged pointer，直接返回 this

    sidetable_lock(); // 为 sidetable 加锁
    isa_t bits = LoadExclusive(&isa.bits); // 通过 LoadExclusive 方法加载 isa 的值，加锁
    ClearExclusive(&isa.bits); // 解锁
    if (bits.nonpointer) { // 如果 isa 不是通过指针方式实现的话
        uintptr_t rc = 1 + bits.extra_rc; // 获取当前对象的引用计数
        if (bits.has_sidetable_rc) { // 如果当前对象有额外的引用计数
            rc += sidetable_getExtraRC_nolock(); // 加上额外的引用计数
        }
        sidetable_unlock(); // 为 sidetable 解锁
        return rc;
    }

    sidetable_unlock(); // 为 sidetable 解锁
    return sidetable_retainCount(); // 返回 sidetable 中存储的 retainCount
}
```

rootRetainCount 方法做的事情已经写到上面了，接下来来看一下其中几个重要函数（方法）的实现：

### LoadExclusive & ClearExclusive

LoadExclusive 的作用是让读取操作原子化，根据 CPU 不同实现不同，比如在 x86-64 上就是单纯的直接返回值，而在 arm64 上则使用了 ldxr（load exclusive register） 指令。如下所示：

```cpp
static ALWAYS_INLINE
uintptr_t 
LoadExclusive(uintptr_t *src)
{
    uintptr_t result;
    asm("ldxr %" p "0, [%x1]" 
        : "=r" (result) 
        : "r" (src), "m" (*src));
    return result;
}
```

同理，ClearExclusive 则是解除锁，使用了 clrex 指令，如下所示：

```cpp
static ALWAYS_INLINE
void 
ClearExclusive(uintptr_t *dst)
{
    // pretend it writes to *dst for instruction ordering purposes
    asm("clrex" : "=m" (*dst));
}
```

### extra_rc

extra_rc 就是用于保存自动引用计数的标志位，是在 isa_t 结构体中，共有8位。它存储的是对象本身之外的引用计数的数量，所以获取总数的时候要加 1。但 extra_rc 可能不足以存储引用计数，这时候 sidetable 就派上用场了。

### has_sidetable_rc

has_sidetable_rc 是用于标志是否通过 sidetable 存储引用计数的标志。

### sidetable_getExtraRC_nolock

sidetable_getExtraRC_nolock 函数是用于从 sidetable 中获取引用计数信息的方法，实现如下：

```cpp
size_t 
objc_object::sidetable_getExtraRC_nolock()
{
    assert(isa.nonpointer);
    SideTable& table = SideTables()[this]; // 根据 this 找到引用计数存在的 table
    RefcountMap::iterator it = table.refcnts.find(this); // 查找引用计数
    if (it == table.refcnts.end()) return 0; // 如果没找到，返回0
    else return it->second >> SIDE_TABLE_RC_SHIFT; // 如果找到了，通过 SIDE_TABLE_RC_SHIFT 位掩码获取对应的引用计数
}
```

### sidetable_retainCount

如果 nonpointer 为 true，表示使用的还是老版本中通过指针表示 isa 的方式，这个时候，所有的引用计数全部存在 sidetable 中，如下所示：

```cpp
uintptr_t
objc_object::sidetable_retainCount()
{
    SideTable& table = SideTables()[this]; // 根据 this 找到引用计数存在的 table

    size_t refcnt_result = 1; // 设置对象本身的引用计数为1
    
    table.lock(); // 加锁
    RefcountMap::iterator it = table.refcnts.find(this); // 查找引用计数
    if (it != table.refcnts.end()) { // 如果找到了
        // this is valid for SIDE_TABLE_RC_PINNED too
        refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;  // 返回 1 + sidetable 中存储的引用计数
    }
    table.unlock(); // 解锁
    return refcnt_result;
}
```

## retain

retain 的主要作用是增加引用计数，实现如下：

```cpp
// Replaced by ObjectAlloc
- (id)retain {
    return ((id)self)->rootRetain();
}

ALWAYS_INLINE id 
objc_object::rootRetain()
{
    return rootRetain(false, false);
}

ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    if (isTaggedPointer()) return (id)this; // 如果是 tagged pointer 直接返回 this

    bool sideTableLocked = false; // 标记 sideTable 是否处于加锁状态
    bool transcribeToSideTable = false; // 是否需要将引用计数存储在 sideTable 中

    isa_t oldisa;
    isa_t newisa;

    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits); // 取出 isa，加锁
        newisa = oldisa;
        if (slowpath(!newisa.nonpointer)) { // 如果 isa 是通过指针方式实现的
            ClearExclusive(&isa.bits); // 解锁
            if (!tryRetain && sideTableLocked) sidetable_unlock(); // 如果 tryRetain 为 false 且给 sideTable 加过锁，则为其解锁
            if (tryRetain) return sidetable_tryRetain() ? (id)this : nil; // 如果 tryRetain 为 ture，则根据调用 sidetable_tryRetain() 结果决定返回 this 或者是 nil。
            else return sidetable_retain(); // 其余情况直接返回 sidetable_retain 的结果
        }
        // don't check newisa.fast_rr; we already called any RR overrides
        if (slowpath(tryRetain && newisa.deallocating)) { // 如果 tryRetain 为 true，且对象正在被销毁，则执行以下代码
            ClearExclusive(&isa.bits); // 解锁
            if (!tryRetain && sideTableLocked) sidetable_unlock(); // 如果 tryRetain 为 false 且给 sideTable 加过锁，则为其解锁
            return nil;
        }
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++

        if (slowpath(carry)) { // 如果 carry 为 true，要处理引用计数溢出的情况
            // newisa.extra_rc++ overflowed
            if (!handleOverflow) { // 如果 handleOverflow 为 false
                ClearExclusive(&isa.bits); // 解锁
                return rootRetain_overflow(tryRetain); // 跳转执行 rootRetain_overflow
            }
            // Leave half of the retain counts inline and 
            // prepare to copy the other half to the side table.
            if (!tryRetain && !sideTableLocked) sidetable_lock(); // 如果 sidetable_lock 未加锁，则为其加锁
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true; // 标记是否在 sidetable 中存有引用计数
        }
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));

    if (slowpath(transcribeToSideTable)) { // 如果需要讲部分的引用计数放到 sidetable 中
        // Copy the other half of the retain counts to the side table.
        sidetable_addExtraRC_nolock(RC_HALF);
    }

    if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock(); // 解锁
    return (id)this;
}
```

关于 retain 方法每一步做了什么都已经写到注释里了，接下来来看几个知识点：

### sidetable_lock & sidetable_unlock

这两个方法是成对儿使用的，作用是给 sidetable 加锁/解锁

```objc
void 
objc_object::sidetable_lock()
{
    SideTable& table = SideTables()[this];
    table.lock();
}

void 
objc_object::sidetable_unlock()
{
    SideTable& table = SideTables()[this];
    table.unlock();
}
```

### sidetable_tryRetain

sidetable_tryRetain 的实现如下：

```objc
bool
objc_object::sidetable_tryRetain()
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this];

    // NO SPINLOCK HERE
    // _objc_rootTryRetain() is called exclusively by _objc_loadWeak(), 
    // which already acquired the lock on our behalf.

    // fixme can't do this efficiently with os_lock_handoff_s
    // if (table.slock == 0) {
    //     _objc_fatal("Do not call -_tryRetain.");
    // }

    bool result = true;
    RefcountMap::iterator it = table.refcnts.find(this); // 通过 this 从 table 中寻找是否有引用计数
    if (it == table.refcnts.end()) { // 如果查找不到
        table.refcnts[this] = SIDE_TABLE_RC_ONE; // 设置表中的引用计数为 SIDE_TABLE_RC_ONE
    } else if (it->second & SIDE_TABLE_DEALLOCATING) { // 如果对象此时处于 deallocating 状态
        result = false; // 结果设为 false
    } else if (! (it->second & SIDE_TABLE_RC_PINNED)) { // 如果查找到了，且未溢出，则将引用计数加1。
        it->second += SIDE_TABLE_RC_ONE;
    }
    
    return result;
}
```

#### 位偏移量

Objecitve-C 定义了几个重要的偏移量，用于取出对应的值，定义如下

```cpp
// The order of these bits is important.
#define SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)
#define SIDE_TABLE_DEALLOCATING      (1UL<<1)  // MSB-ward of weak bit
#define SIDE_TABLE_RC_ONE            (1UL<<2)  // MSB-ward of deallocating bit
#define SIDE_TABLE_RC_PINNED         (1UL<<(WORD_BITS-1))

#define SIDE_TABLE_RC_SHIFT 2
#define SIDE_TABLE_FLAG_MASK (SIDE_TABLE_RC_ONE-1)
```

1. SIDE_TABLE_DEALLOCATING (1UL<<1)（表示对象所在内存的第 2 位），标识该对象是否正在 dealloc（析构）。
2. SIDE_TABLE_RC_ONE (1UL<<2) （表示对象所在内存的第 3 位），存放引用计数数值（其实第三位之后都用来存放引用计数数值）。
3. SIDE_TABLE_RC_PINNED (1UL<<(WORD_BITS-1)) (表示对象所在内存的倒数第一位)，是引用计数的溢出标志位。

### sidetable_retain

实现如下：

```cpp
id
objc_object::sidetable_retain()
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this];
    
    table.lock();
    size_t& refcntStorage = table.refcnts[this];
    if (! (refcntStorage & SIDE_TABLE_RC_PINNED)) { // 如果查找到了，且未溢出，则将引用计数加1。
        refcntStorage += SIDE_TABLE_RC_ONE;
    }
    table.unlock();

    return (id)this;
}
```

### addc

这个函数的作用就是引用计数加一，其实现如下：

```cpp
static ALWAYS_INLINE uintptr_t 
addc(uintptr_t lhs, uintptr_t rhs, uintptr_t carryin, uintptr_t *carryout)
{
    return __builtin_addcl(lhs, rhs, carryin, carryout);
}
```

__builtin_addcl 是 Clang Language Extensions，但是我没找到它的定义，大概是大整数加法吧。

### rootRetain_overflow

当 addc 方法告知我们有溢出情况出现时，我们需要通过 rootRetain_overflow 方法处理溢出的情况，实现如下：

```cpp
NEVER_INLINE id 
objc_object::rootRetain_overflow(bool tryRetain)
{
    return rootRetain(tryRetain, true);
}
```

本质上属于递归调用，又回到了这个方法。

#### NEVER_INLINE

NEVER_INLINE 的定义如下：

```cpp
#define NEVER_INLINE inline __attribute__((noinline))
```

我们在上一篇文章里已经介绍过 inline 了，在此就不赘述了。

### RC_HALF;

当我们确定引用计数已经溢出的时候，通过 newisa.extra_rc = RC_HALF; 将其设置为溢出值。RC_HALF 定义如下：

```cpp
#define RC_HALF (1ULL<<7)
```

extra_rc 总共为 8 位，RC_HALF = 0b10000000。

### StoreExclusive

其实现如下：

```cpp
static ALWAYS_INLINE
bool 
StoreExclusive(uintptr_t *dst, uintptr_t oldvalue, uintptr_t value)
{
    
    return __sync_bool_compare_and_swap((void **)dst, (void *)oldvalue, (void *)value);
}
```

__sync 开头的方法是原子操作，这个函数是用来做比较后操作数的原子操作，如果 dst == oldvalue，就将 value 写入 dst，并在相等并写入的情况下返回 true。

### sidetable_addExtraRC_nolock

sidetable_addExtraRC_nolock 方法用于将溢出的引用计数加到 sidetable 中。实现如下：

```cpp
// Move some retain counts to the side table from the isa field.
// Returns true if the object is now pinned.
bool 
objc_object::sidetable_addExtraRC_nolock(size_t delta_rc)
{
    assert(isa.nonpointer);
    SideTable& table = SideTables()[this];

    size_t& refcntStorage = table.refcnts[this];
    size_t oldRefcnt = refcntStorage;
    // isa-side bits should not be set here
    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    if (oldRefcnt & SIDE_TABLE_RC_PINNED) return true;

    uintptr_t carry;
    size_t newRefcnt = 
        addc(oldRefcnt, delta_rc << SIDE_TABLE_RC_SHIFT, 0, &carry);
    if (carry) {
        refcntStorage =
            SIDE_TABLE_RC_PINNED | (oldRefcnt & SIDE_TABLE_FLAG_MASK);
        return true;
    }
    else {
        refcntStorage = newRefcnt;
        return false;
    }
}
```

## release

在了解完 retain 的实现之后，release 的实现就比较容易解读了，基本就是 retain 的反操作，实现如下：

```cpp
-(void) release
{
    _objc_rootRelease(self);
}

void
_objc_rootRelease(id obj)
{
    assert(obj);

    obj->rootRelease();
}

ALWAYS_INLINE bool 
objc_object::rootRelease()
{
    return rootRelease(true, false);
}

ALWAYS_INLINE bool 
objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    if (isTaggedPointer()) return false;

    bool sideTableLocked = false; // 标识位，用于标记 sideTable 是否加锁

    isa_t oldisa;
    isa_t newisa;

 retry:
    do {
        oldisa = LoadExclusive(&isa.bits); // 为 isa.bits 加锁
        newisa = oldisa;
        if (slowpath(!newisa.nonpointer)) { // 如果 newisa 是通过指针实现的
            ClearExclusive(&isa.bits); // 为 isa.bits 解锁
            if (sideTableLocked) sidetable_unlock(); // 如果 sideTableLocked 处于被加锁状态，为其解锁
            return sidetable_release(performDealloc); // 通过 sidetable_release 为其解锁
        }
        // don't check newisa.fast_rr; we already called any RR overrides
        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (slowpath(carry)) { // 如果发现溢出的情况
            // don't ClearExclusive()
            goto underflow;
        }
    } while (slowpath(!StoreReleaseExclusive(&isa.bits, 
                                             oldisa.bits, newisa.bits)));

    if (slowpath(sideTableLocked)) sidetable_unlock(); // 解锁
    return false;

 underflow:
    // newisa.extra_rc-- underflowed: borrow from side table or deallocate

    // abandon newisa to undo the decrement
    newisa = oldisa;

    if (slowpath(newisa.has_sidetable_rc)) {
        if (!handleUnderflow) {
            ClearExclusive(&isa.bits); // 接触 isa.bits 的锁
            return rootRelease_underflow(performDealloc); // 处理下溢的情况
        }

        // Transfer retain count from side table to inline storage.

        if (!sideTableLocked) {
            ClearExclusive(&isa.bits); // 解除 isa.bits 的锁 
            sidetable_lock(); // 为 sidetable_lock 加锁
            sideTableLocked = true; // 标记 sideTable 被加锁
            // Need to start over to avoid a race against 
            // the nonpointer -> raw pointer transition.
            goto retry;
        }

        // Try to remove some retain counts from the side table.        
        size_t borrowed = sidetable_subExtraRC_nolock(RC_HALF); // 将 sidetable 中的引用计数减一

        // To avoid races, has_sidetable_rc must remain set 
        // even if the side table count is now zero.

        if (borrowed > 0) {
            // Side table retain count decreased.
            // Try to add them to the inline count.
            newisa.extra_rc = borrowed - 1;  // redo the original decrement too
            bool stored = StoreReleaseExclusive(&isa.bits, 
                                                oldisa.bits, newisa.bits); // 存储更改后的 isa.bits 
            if (!stored) { // 如果存储失败，即刻重试一次
                // Inline update failed. 
                // Try it again right now. This prevents livelock on LL/SC 
                // architectures where the side table access itself may have 
                // dropped the reservation.
                isa_t oldisa2 = LoadExclusive(&isa.bits);
                isa_t newisa2 = oldisa2;
                if (newisa2.nonpointer) {
                    uintptr_t overflow;
                    newisa2.bits = 
                        addc(newisa2.bits, RC_ONE * (borrowed-1), 0, &overflow);
                    if (!overflow) {
                        stored = StoreReleaseExclusive(&isa.bits, oldisa2.bits, 
                                                       newisa2.bits);
                    }
                }
            }

            if (!stored) { // 如果还是更新失败，则回滚操作并 retry
                // Inline update failed.
                // Put the retains back in the side table.
                sidetable_addExtraRC_nolock(borrowed);
                goto retry;
            }

            // Decrement successful after borrowing from side table.
            // This decrement cannot be the deallocating decrement - the side 
            // table lock and has_sidetable_rc bit ensure that if everyone 
            // else tried to -release while we worked, the last one would block.
            sidetable_unlock(); // 解锁
            return false;
        }
        else {
            // Side table is empty after all. Fall-through to the dealloc path.
        }
    }

    // Really deallocate.

    if (slowpath(newisa.deallocating)) { // 如果当前 newisa 处于 deallocating 状态
        ClearExclusive(&isa.bits); // 解除 isa.bits 的锁
        if (sideTableLocked) sidetable_unlock(); // 解除 sidetable 的锁
        return overrelease_error(); // 返回 overrelease_error 的结果值
        // does not actually return
    }
    newisa.deallocating = true; // 设置 deallocating 为 true
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry; // 如果存储失败，继续重试

    if (slowpath(sideTableLocked)) sidetable_unlock(); // 解除 sidetable 的锁

    __sync_synchronize();
    if (performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return true;
}
```

### sidetable_release

sidetable_release 是为 sidetable 中存储的引用计数执行减1操作的方法，实现如下：

```cpp
// rdar://20206767
// return uintptr_t instead of bool so that the various raw-isa 
// -release paths all return zero in eax
uintptr_t
objc_object::sidetable_release(bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this]; // 通过对象获取对应的 table

    bool do_dealloc = false; // 标识是否需要执行 dealloc 方法

    table.lock(); // 加锁
    RefcountMap::iterator it = table.refcnts.find(this); // 通过 this 查找引用计数
    if (it == table.refcnts.end()) { // 未找到
        do_dealloc = true; 
        table.refcnts[this] = SIDE_TABLE_DEALLOCATING; // 设为 deallocating 状态
    } else if (it->second < SIDE_TABLE_DEALLOCATING) { // 如果对象处于 deallocating 状态
        // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
        do_dealloc = true;
        it->second |= SIDE_TABLE_DEALLOCATING;
    } else if (! (it->second & SIDE_TABLE_RC_PINNED)) { // 如果查找到了对象的引用计数
        it->second -= SIDE_TABLE_RC_ONE; // 引用计数加一
    }
    table.unlock(); // 解锁
    if (do_dealloc  &&  performDealloc) { // 如果符合调用 dealloc 方法的情况下
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc); // 通过 objc_msgSend 执行 dealloc 方法
    }
    return do_dealloc;
}
```

### subc

就是 addc 的反操作，实现如下：

```cpp
static ALWAYS_INLINE uintptr_t 
subc(uintptr_t lhs, uintptr_t rhs, uintptr_t carryin, uintptr_t *carryout)
{
    return __builtin_subcl(lhs, rhs, carryin, carryout);
}
```

### rootRelease_underflow

用于处理下溢的情况，实现如下

```cpp
NEVER_INLINE bool 
objc_object::rootRelease_underflow(bool performDealloc)
{
    return rootRelease(performDealloc, true);
}
```

### sidetable_subExtraRC_nolock

sidetable_subExtraRC_nolock 用来将 sidetable 中存储的引用计数减一，实现如下：

```cpp
// Move some retain counts from the side table to the isa field.
// Returns the actual count subtracted, which may be less than the request.
size_t 
objc_object::sidetable_subExtraRC_nolock(size_t delta_rc)
{
    assert(isa.nonpointer);
    SideTable& table = SideTables()[this];

    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()  ||  it->second == 0) {
        // Side table retain count is zero. Can't borrow.
        return 0;
    }
    size_t oldRefcnt = it->second;

    // isa-side bits should not be set here
    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0); // 确保对象不是 deallocating 状态
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0); // 确保是不是弱饮用

    size_t newRefcnt = oldRefcnt - (delta_rc << SIDE_TABLE_RC_SHIFT); // 引用计数减一
    assert(oldRefcnt > newRefcnt);  // shouldn't underflow
    it->second = newRefcnt;
    return delta_rc; // 返回引用计数的变化
}
```

### StoreReleaseExclusive

StoreReleaseExclusive 内部直接调用了我们之前提过的 StoreExclusive 方法，实现如下：

```cpp
static ALWAYS_INLINE
bool 
StoreReleaseExclusive(uintptr_t *dst, uintptr_t oldvalue, uintptr_t value)
{
    return StoreExclusive(dst, oldvalue, value);
}
```

### overrelease_error

overrelease_error 是用来在过度调用 release 的时候报错用的，实现如下：

```cpp
NEVER_INLINE
bool 
objc_object::overrelease_error()
{
    _objc_inform_now_and_on_crash("%s object %p overreleased while already deallocating; break on objc_overrelease_during_dealloc_error to debug", object_getClassName((id)this), this);
    objc_overrelease_during_dealloc_error();
    return false;  // allow rootRelease() to tail-call this
}

BREAKPOINT_FUNCTION(
    void objc_overrelease_during_dealloc_error(void)
);
```

### __sync_synchronize

__sync_synchronize 的作用是：

No memory operand will be moved across the operation, either forward or backward. Further, instructions will be issued as necessary to prevent the processor from speculating loads across the operation and from queuing stores after the operation.

简而言之就是，在这个方法调用之后，涉及到内存的运算符都被禁止操作了（因为要进行析构了）。

以上就是关于 retainCount、retain 和 release 的全部源码解读了。