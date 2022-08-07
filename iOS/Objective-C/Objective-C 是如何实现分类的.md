# Objective-C 是如何实现分类的

在写 `Objective-C` 代码的时候，如果想给没法获得源码的类增加一些方法，`Category` 即分类是一种很好的方法，本文将带你了解分类是如何实现为类添加方法的。

先说结论，分类中的方法会在编译时变成 `category_t` 结构体的变量，在运行时合并进主类，分类中的方法会放在主类中方法的前面，主类中原有的方法不会被覆盖。同时，同名的分类方法，后编译的分类方法会“覆盖”先编译的分类方法。

## 编译时

在编译时，所有我们写的分类，都会转化为 `category_t` 结构体的变量，`category_t` 的源码如下：

```objc
struct category_t {
    const char *name; // 分类名
    classref_t cls; // 主类
    WrappedPtr<method_list_t, PtrauthStrip> instanceMethods; // 实例方法
    WrappedPtr<method_list_t, PtrauthStrip> classMethods; // 类方法
    struct protocol_list_t *protocols; // 协议
    struct property_list_t *instanceProperties; // 属性
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties; // 类属性

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
    
    protocol_list_t *protocolsForMeta(bool isMeta) {
        if (isMeta) return nullptr;
        else return protocols;
    }
};
```

这个结构体主要是用来存储分类中可表现的信息，同时也从侧面说明了分类是不能创建实例变量的。

## 运行时

`map_images_nolock` 是运行时的开始，同时也决定了编译顺序对分类方法之间优先级的影响，后编译的分类方法会放在先编译的前面：

```
void 
map_images_nolock(unsigned mhCount, const char * const mhPaths[],
                  const struct mach_header * const mhdrs[])
{
    ...
    {
        uint32_t i = mhCount;
        while (i--) { // 读取 header_info 的顺序，决定了后编译的分类方法会放在先编译的前面
            const headerType *mhdr = (const headerType *)mhdrs[i];

            auto hi = addHeader(mhdr, mhPaths[i], totalClasses, unoptimizedTotalClasses);
    ...
```

在运行时，加载分类的起始方法是 `loadAllCategories`，可以看到，该方法从 `FirstHeader` 开始，遍历所有的 `header_info`，并依次调用 `load_categories_nolock` 方法，实现如下：

```
static void loadAllCategories() {
    mutex_locker_t lock(runtimeLock);

    for (auto *hi = FirstHeader; hi != NULL; hi = hi->getNext()) {
        load_categories_nolock(hi);
    }
}
```

在 `load_categories_nolock` 方法中，会判断类是不是 `stubClass` 切是否初始化完成，来决定分类到底附着在哪里，其实现如下：

```objc
static void load_categories_nolock(header_info *hi) {
    // 是否具有类属性
    bool hasClassProperties = hi->info()->hasCategoryClassProperties();

    size_t count;
    auto processCatlist = [&](category_t * const *catlist) { // 获取需要处理的分类列表
        for (unsigned i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls); // 获取分类对应的主类
            locstamped_category_t lc{cat, hi};

            if (!cls) { // 获取不到主类（可能因为弱链接），跳过本次循环
                // Category's target class is missing (probably weak-linked).
                // Ignore the category.
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class",
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category.
            if (cls->isStubClass()) { // 如果时 stubClass，当时无法确定元类对象是哪个，所以先附着在 stubClass 本身上
                // Stub classes are never realized. Stub classes
                // don't know their metaclass until they're
                // initialized, so we have to add categories with
                // class methods or properties to the stub itself.
                // methodizeClass() will find them and add them to
                // the metaclass as appropriate.
                if (cat->instanceMethods ||
                    cat->protocols ||
                    cat->instanceProperties ||
                    cat->classMethods ||
                    cat->protocols ||
                    (hasClassProperties && cat->_classProperties))
                {
                    objc::unattachedCategories.addForClass(lc, cls);
                }
            } else {
                // First, register the category with its target class.
                // Then, rebuild the class's method lists (etc) if
                // the class is realized.
                if (cat->instanceMethods ||  cat->protocols
                    ||  cat->instanceProperties)
                {
                    if (cls->isRealized()) { // 表示类对象已经初始化完毕，会进入合并方法。
                        attachCategories(cls, &lc, 1, ATTACH_EXISTING);
                    } else {
                        objc::unattachedCategories.addForClass(lc, cls);
                    }
                }

                if (cat->classMethods  ||  cat->protocols
                    ||  (hasClassProperties && cat->_classProperties))
                {
                    if (cls->ISA()->isRealized()) { // 表示元类对象已经初始化完毕，会进入合并方法。
                        attachCategories(cls->ISA(), &lc, 1, ATTACH_EXISTING | ATTACH_METACLASS);
                    } else {
                        objc::unattachedCategories.addForClass(lc, cls->ISA());
                    }
                }
            }
        }
    };

    processCatlist(hi->catlist(&count));
    processCatlist(hi->catlist2(&count));
}
```

合并分类的方法是通过 `attachCategories` 方法进行的，对方法、属性和协议分别进行附着。需要注意的是，在新版的运行时方法中不是将方法放到 `rw` 中，而是新创建了一个叫做 `rwe` 的属性，目的是为了节约内存，方法的实现如下：

```objc
// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order, 
// oldest categories first.
static void
attachCategories(Class cls, const locstamped_category_t *cats_list, uint32_t cats_count,
                 int flags)
{
    if (slowpath(PrintReplacedMethods)) {
        printReplacements(cls, cats_list, cats_count);
    }
    if (slowpath(PrintConnecting)) {
        _objc_inform("CLASS: attaching %d categories to%s class '%s'%s",
                     cats_count, (flags & ATTACH_EXISTING) ? " existing" : "",
                     cls->nameForLogging(), (flags & ATTACH_METACLASS) ? " (meta)" : "");
    }

    /*
     * Only a few classes have more than 64 categories during launch.
     * This uses a little stack, and avoids malloc.
     *
     * Categories must be added in the proper order, which is back
     * to front. To do that with the chunking, we iterate cats_list
     * from front to back, build up the local buffers backwards,
     * and call attachLists on the chunks. attachLists prepends the
     * lists, so the final result is in the expected order.
     */
    constexpr uint32_t ATTACH_BUFSIZ = 64;
    method_list_t   *mlists[ATTACH_BUFSIZ];
    property_list_t *proplists[ATTACH_BUFSIZ];
    protocol_list_t *protolists[ATTACH_BUFSIZ];

    uint32_t mcount = 0;
    uint32_t propcount = 0;
    uint32_t protocount = 0;
    bool fromBundle = NO;
    bool isMeta = (flags & ATTACH_METACLASS); // 是否是元类对象
    auto rwe = cls->data()->extAllocIfNeeded(); // 为 rwe 生成分配存储空间

    for (uint32_t i = 0; i < cats_count; i++) { // 遍历分类列表
        auto& entry = cats_list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta); // 获取实例方法或类方法列表
        if (mlist) {
            if (mcount == ATTACH_BUFSIZ) { // 达到容器的容量上限时
                prepareMethodLists(cls, mlists, mcount, NO, fromBundle, __func__); // 准备方法列表
                rwe->methods.attachLists(mlists, mcount); // 附着方法到主类中
                mcount = 0;
            }
            mlists[ATTACH_BUFSIZ - ++mcount] = mlist; // 将分类的方法列表放入准备好的容器中
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist =
            entry.cat->propertiesForMeta(isMeta, entry.hi); // 获取对象属性或类属性列表
        if (proplist) {
            if (propcount == ATTACH_BUFSIZ) { // 达到容器的容量上限时进行附着
                rwe->properties.attachLists(proplists, propcount); // 附着属性到类或元类中
                propcount = 0;
            }
            proplists[ATTACH_BUFSIZ - ++propcount] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocolsForMeta(isMeta); // 获取协议列表
        if (protolist) {
            if (protocount == ATTACH_BUFSIZ) { // 达到容器的容量上限时进行附着
                rwe->protocols.attachLists(protolists, protocount); // 附着遵守的协议到类或元类中
                protocount = 0;
            }
            protolists[ATTACH_BUFSIZ - ++protocount] = protolist;
        }
    }
    // 将剩余的方法、属性和协议进行附着
    if (mcount > 0) {
        prepareMethodLists(cls, mlists + ATTACH_BUFSIZ - mcount, mcount,
                           NO, fromBundle, __func__);
        rwe->methods.attachLists(mlists + ATTACH_BUFSIZ - mcount, mcount);
        if (flags & ATTACH_EXISTING) {
            flushCaches(cls, __func__, [](Class c){
                // constant caches have been dealt with in prepareMethodLists
                // if the class still is constant here, it's fine to keep
                return !c->cache.isConstantOptimizedCache();
            });
        }
    }

    rwe->properties.attachLists(proplists + ATTACH_BUFSIZ - propcount, propcount);

    rwe->protocols.attachLists(protolists + ATTACH_BUFSIZ - protocount, protocount);
}
```

而真正进行方法附着的 `attachLists` 方法，其作用是将分类的方法放置到类对象或元类对象中，且放在类和元类对象原有方法的前面，这也是为什么分类和类中如果出现同名的方法，会优先调用分类的，也从侧面说明了，原有的类中的方法其实并没有被覆盖：

```objc
void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return; // 数量为 0 直接返回

        if (hasArray()) {
            // many lists -> many lists
            uint32_t oldCount = array()->count; // 原有的方法列表的个数
            uint32_t newCount = oldCount + addedCount; // 合并后的方法列表的个数
            array_t *newArray = (array_t *)malloc(array_t::byteSize(newCount)); // 创建新的数组
            newArray->count = newCount;
            array()->count = newCount;

            for (int i = oldCount - 1; i >= 0; i--)
                newArray->lists[i + addedCount] = array()->lists[i]; // 将原有的方法，放到新创建的数组的最后面
            for (unsigned i = 0; i < addedCount; i++)
                newArray->lists[i] = addedLists[i]; // 将分类中的方法，放到数组的前面
            free(array()); // 释放原有数组的内存空间
            setArray(newArray); // 将合并后的数组作为新的方法数组
            validate();
        }
        else if (!list  &&  addedCount == 1) { // 如果原本不存在方法列表，直接替换
            // 0 lists -> 1 list
            list = addedLists[0];
            validate();
        } 
        else { // 如果原来只有一个列表，变为多个，走这个逻辑
            // 1 list -> many lists
            Ptr<List> oldList = list;
            uint32_t oldCount = oldList ? 1 : 0;
            uint32_t newCount = oldCount + addedCount; // 计算所有方法列表的个数
            setArray((array_t *)malloc(array_t::byteSize(newCount))); // 分配新的内存空间并赋值
            array()->count = newCount;
            if (oldList) array()->lists[addedCount] = oldList; // 将原有的方法，放到新创建的数组的最后面
            for (unsigned i = 0; i < addedCount; i++) // 将分类中的方法，放到数组的前面
                array()->lists[i] = addedLists[i]; 
            validate();
        }
    }
```