# 从源码角度看苹果是如何实现 dealloc 的

大多数与生命周期有关的方法我们都已经讲完了，接下来来看最后一个方法，dealloc。

## dealloc

dealloc 方法的实现如下：

```cpp
// Replaced by NSZombies
- (void)dealloc {
    _objc_rootDealloc(self);
}

void
_objc_rootDealloc(id obj)
{
    assert(obj);

    obj->rootDealloc();
}

inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;  // fixme necessary?

    if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    { // 如果 isa 不是通过指针实现的，不是弱引用，没有关联对象，没有 C++ 析构函数，没有在 sidetable 中存储引用计数，直接 free
        assert(!sidetable_present());
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}
```

object_dispose 的实现如下所示：

```cpp
/***********************************************************************
* object_dispose
* fixme
* Locking: none
**********************************************************************/
id 
object_dispose(id obj)
{
    if (!obj) return nil;

    objc_destructInstance(obj);    
    free(obj);

    return nil;
}

/***********************************************************************
* objc_destructInstance
* Destroys an instance without freeing memory. 
* Calls C++ destructors.
* Calls ARC ivar cleanup.
* Removes associative references.
* Returns `obj`. Does nothing if `obj` is nil.
**********************************************************************/
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor(); // 获取是否有 C++ 析构函数
        bool assoc = obj->hasAssociatedObjects(); // 获取是否有关联对象

        // This order is important.
        if (cxx) object_cxxDestruct(obj); // 如果有析构函数，调用 object_cxxDestruct
        if (assoc) _object_remove_assocations(obj); // 如果有关联对象，调用 _object_remove_assocations
        obj->clearDeallocating();
    }

    return obj;
}
```

### hasCxxDtor & object_cxxDestruct

hasCxxDtor 实现如下：

```cpp
inline bool
objc_object::hasCxxDtor()
{
    assert(!isTaggedPointer());
    if (isa.nonpointer) return isa.has_cxx_dtor;
    else return isa.cls->hasCxxDtor();
}
```

object_cxxDestruct 实现如下：

```cpp
/***********************************************************************
* object_cxxDestruct.
* Call C++ destructors on obj, if any.
* Uses methodListLock and cacheUpdateLock. The caller must hold neither.
**********************************************************************/
void object_cxxDestruct(id obj)
{
    if (!obj) return;
    if (obj->isTaggedPointer()) return;
    object_cxxDestructFromClass(obj, obj->ISA());
}

/***********************************************************************
* object_cxxDestructFromClass.
* Call C++ destructors on obj, starting with cls's 
*   dtor method (if any) followed by superclasses' dtors (if any), 
*   stopping at cls's dtor (if any).
* Uses methodListLock and cacheUpdateLock. The caller must hold neither.
**********************************************************************/
static void object_cxxDestructFromClass(id obj, Class cls)
{
    void (*dtor)(id);

    // Call cls's dtor first, then superclasses's dtors.

    for ( ; cls; cls = cls->superclass) { // 递归调用 obj 及其父类的析构函数
        if (!cls->hasCxxDtor()) return; 
        dtor = (void(*)(id))
            lookupMethodInClassAndLoadCache(cls, SEL_cxx_destruct); // 查找方法
        if (dtor != (void(*)(id))_objc_msgForward_impcache) {
            if (PrintCxxCtors) {
                _objc_inform("CXX: calling C++ destructors for class %s", 
                             cls->nameForLogging());
            }
            (*dtor)(obj);
        }
    }
}
```

### hasAssociatedObjects & _object_remove_assocations

hasAssociatedObjects 实现如下：

```objc
inline bool
objc_object::hasAssociatedObjects()
{
    if (isTaggedPointer()) return true;
    if (isa.nonpointer) return isa.has_assoc;
    return true;
}
```

_object_remove_assocations 实现如下，简而言之就是查找对象关联的所有对象，并依次调用 ReleaseValue。

```cpp
void _object_remove_assocations(id object) {
    vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        if (associations.size() == 0) return;
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            // copy all of the associations that need to be removed.
            ObjectAssociationMap *refs = i->second;
            for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                elements.push_back(j->second);
            }
            // remove the secondary table.
            delete refs;
            associations.erase(i);
        }
    }
    // the calls to releaseValue() happen outside of the lock.
    for_each(elements.begin(), elements.end(), ReleaseValue());
}

struct ReleaseValue {
    void operator() (ObjcAssociation &association) {
        releaseValue(association.value(), association.policy());
    }
};
```

### clearDeallocating

clearDeallocating 方法的实现如下：

```cpp
inline void 
objc_object::clearDeallocating()
{
    if (slowpath(!isa.nonpointer)) {
        // Slow path for raw pointer isa.
        sidetable_clearDeallocating();
    }
    else if (slowpath(isa.weakly_referenced  ||  isa.has_sidetable_rc)) {
        // Slow path for non-pointer isa with weak refs and/or side table data.
        clearDeallocating_slow();
    }

    assert(!sidetable_present());
}
```

#### sidetable_clearDeallocating

sidetable_clearDeallocating 方法会在 isa 是指针的实现中被调用，其实现如下：

```cpp
void 
objc_object::sidetable_clearDeallocating()
{
    SideTable& table = SideTables()[this];

    // clear any weak table items
    // clear extra retain count and deallocating bit
    // (fixme warn or abort if extra retain count == 0 ?)
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        if (it->second & SIDE_TABLE_WEAKLY_REFERENCED) {
            weak_clear_no_lock(&table.weak_table, (id)this);
        }
        table.refcnts.erase(it);
    }
    table.unlock();
}
```

#### clearDeallocating_slow

clearDeallocating_slow 方法会在 isa 是非指针的实现中被调用，其实现如下：

```cpp
// Slow path of clearDeallocating() 
// for objects with nonpointer isa
// that were ever weakly referenced 
// or whose retain count ever overflowed to the side table.
NEVER_INLINE void
objc_object::clearDeallocating_slow()
{
    assert(isa.nonpointer  &&  (isa.weakly_referenced || isa.has_sidetable_rc));

    SideTable& table = SideTables()[this];
    table.lock();
    if (isa.weakly_referenced) {
        weak_clear_no_lock(&table.weak_table, (id)this);
    }
    if (isa.has_sidetable_rc) {
        table.refcnts.erase(this);
    }
    table.unlock();
}
```
