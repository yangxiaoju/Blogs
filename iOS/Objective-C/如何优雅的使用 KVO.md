# 如何优雅的使用 KVO

`KVO` 是苹果为我们提供的一套强大的机制，用于观察属性值的变化，但是大家在日常开发中想必多少也感受到了使用上的一些不便利，比如：

- 添加观察者和移除观察者的次数需要一一对应，否则会 `Crash`。
- 添加观察者和接受到属性变更通知的位置是分开的，不利于判断上下文。
- 多次对同一个属性值进行观察，会触发多次回调，影响业务逻辑。

为了解决上述三个问题，业界提出了一些方便开发者的开源方案，我们一起来看一下。

## KVOController

`KVOController` 建立在 `Cocoa` 久经考验的 `KVO` 实现之上。它提供了一个简单、现代的 `API`，也是线程安全的。好处包括：

- 使用 `blocks`、`custom actions`或 `NSKeyValueObserving` 回调。
- 观察者移除没有异常。
- 控制器 `dealloc` 时隐式移除观察者。
- 具有防止观察者复活的特殊保护的线程安全。

其使用方式也很简单：

```objc
// create KVO controller with observer
FBKVOController *KVOController = [FBKVOController controllerWithObserver:self];
self.KVOController = KVOController;

// observe clock date property
[self.KVOController observe:clock keyPath:@"date" options:NSKeyValueObservingOptionInitial|NSKeyValueObservingOptionNew block:^(ClockView *clockView, Clock *clock, NSDictionary *change) {

  // update clock view with new value
  clockView.date = change[NSKeyValueChangeNewKey];
}];
```

同时，`KVOController` 还提供了分类，通过关联引用自动帮你创建了 `KVOController` 框架，方便我们使用：

```objc
[self.KVOController observe:clock keyPath:@"date" options:NSKeyValueObservingOptionInitial|NSKeyValueObservingOptionNew action:@selector(updateClockWithDateChange:)];
```

我们来简单看一下 `KVOController` 是怎么做的：

```objc
- (instancetype)initWithObserver:(nullable id)observer retainObserved:(BOOL)retainObserved
{
  self = [super init];
  if (nil != self) {
    _observer = observer;
    NSPointerFunctionsOptions keyOptions = retainObserved ? NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPointerPersonality : NSPointerFunctionsWeakMemory|NSPointerFunctionsObjectPointerPersonality;
    _objectInfosMap = [[NSMapTable alloc] initWithKeyOptions:keyOptions valueOptions:NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPersonality capacity:0];
    pthread_mutex_init(&_lock, NULL);
  }
  return self;
}
```

`KVOController` 分为两种：强引用和弱引用，其中强引用会在使用时持有被观察的对象，反之弱引用则不会。所以在初始化的时候，会创建一个 `objectInfosMap`，这个是 `NSMapTable`，支持弱引用容器。同时会创建一个锁。

注册观察者的时候的代码如下：

```objc
- (void)observe:(nullable id)object keyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options block:(FBKVONotificationBlock)block
{
  NSAssert(0 != keyPath.length && NULL != block, @"missing required parameters observe:%@ keyPath:%@ block:%p", object, keyPath, block);
  if (nil == object || 0 == keyPath.length || NULL == block) {
    return;
  }

  // create info
  _FBKVOInfo *info = [[_FBKVOInfo alloc] initWithController:self keyPath:keyPath options:options block:block];

  // observe object with info
  [self _observe:object info:info];
}
```

通过创建 `_FBKVOInfo` 对象，来实现对观察者信息的封装，算是一个模型类，这个内部类的初始化方法如下：

```objc
- (instancetype)initWithController:(FBKVOController *)controller keyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options block:(FBKVONotificationBlock)block
{
  return [self initWithController:controller keyPath:keyPath options:options block:block action:NULL context:NULL];
}

- (instancetype)initWithController:(FBKVOController *)controller
                           keyPath:(NSString *)keyPath
                           options:(NSKeyValueObservingOptions)options
                             block:(nullable FBKVONotificationBlock)block
                            action:(nullable SEL)action
                           context:(nullable void *)context
{
  self = [super init];
  if (nil != self) {
    _controller = controller;
    _block = [block copy];
    _keyPath = [keyPath copy];
    _options = options;
    _action = action;
    _context = context;
  }
  return self;
}
```

接下来会将观察者的信息存储到 `KVOController` 创建时初始化的 `NSMapTable` 中：

```objc
- (void)_observe:(id)object info:(_FBKVOInfo *)info
{
  // lock
  pthread_mutex_lock(&_lock);

  NSMutableSet *infos = [_objectInfosMap objectForKey:object];

  // check for info existence
  _FBKVOInfo *existingInfo = [infos member:info];
  if (nil != existingInfo) {
    // observation info already exists; do not observe it again

    // unlock and return
    pthread_mutex_unlock(&_lock);
    return;
  }

  // lazilly create set of infos
  if (nil == infos) {
    infos = [NSMutableSet set];
    [_objectInfosMap setObject:infos forKey:object];
  }

  // add info and oberve
  [infos addObject:info];

  // unlock prior to callout
  pthread_mutex_unlock(&_lock);

  [[_FBKVOSharedController sharedController] observe:object info:info];
}
```

`objectInfosMap` 是一个 `NSMapTable` 对象，使用被观察的对象 `object` 作为 `key`, `NSMutableSet` 作为 `value`，如果已经有 `info` 存在了，不会进行二次观察。集合存储自定义对象需要判断其 `hash` 值，`_FBKVOInfo` 的 `hash` 方法实现如下：

```objc
- (NSUInteger)hash
{
  return [_keyPath hash];
}

- (BOOL)isEqual:(id)object
{
  if (nil == object) {
    return NO;
  }
  if (self == object) {
    return YES;
  }
  if (![object isKindOfClass:[self class]]) {
    return NO;
  }
  return [_keyPath isEqualToString:((_FBKVOInfo *)object)->_keyPath];
}
```

也就是说，观察者、被观察者和 `keyPath` 构成了观察的唯一性。

接下来来看 `_FBKVOSharedController` 如何进行的观察：

```objc
- (void)observe:(id)object info:(nullable _FBKVOInfo *)info
{
  if (nil == info) {
    return;
  }

  // register info
  pthread_mutex_lock(&_mutex);
  [_infos addObject:info];
  pthread_mutex_unlock(&_mutex);

  // add observer
  [object addObserver:self forKeyPath:info->_keyPath options:info->_options context:(void *)info];

  if (info->_state == _FBKVOInfoStateInitial) {
    info->_state = _FBKVOInfoStateObserving;
  } else if (info->_state == _FBKVOInfoStateNotObserving) {
    // this could happen when `NSKeyValueObservingOptionInitial` is one of the NSKeyValueObservingOptions,
    // and the observer is unregistered within the callback block.
    // at this time the object has been registered as an observer (in Foundation KVO),
    // so we can safely unobserve it.
    [object removeObserver:self forKeyPath:info->_keyPath context:(void *)info];
  }
}
```

`_FBKVOSharedController` 会将 `_FBKVOInfo` 存储到一个 `NSHashTable` 对象中，并对其进行 `KVO`。

在接受到回调时的处理如下所示：

```objc
- (void)observeValueForKeyPath:(nullable NSString *)keyPath
                      ofObject:(nullable id)object
                        change:(nullable NSDictionary<NSString *, id> *)change
                       context:(nullable void *)context
{
  NSAssert(context, @"missing context keyPath:%@ object:%@ change:%@", keyPath, object, change);

  _FBKVOInfo *info;

  {
    // lookup context in registered infos, taking out a strong reference only if it exists
    pthread_mutex_lock(&_mutex);
    info = [_infos member:(__bridge id)context];
    pthread_mutex_unlock(&_mutex);
  }

  if (nil != info) {

    // take strong reference to controller
    FBKVOController *controller = info->_controller;
    if (nil != controller) {

      // take strong reference to observer
      id observer = controller.observer;
      if (nil != observer) {

        // dispatch custom block or action, fall back to default action
        if (info->_block) {
          NSDictionary<NSString *, id> *changeWithKeyPath = change;
          // add the keyPath to the change dictionary for clarity when mulitple keyPaths are being observed
          if (keyPath) {
            NSMutableDictionary<NSString *, id> *mChange = [NSMutableDictionary dictionaryWithObject:keyPath forKey:FBKVONotificationKeyPathKey];
            [mChange addEntriesFromDictionary:change];
            changeWithKeyPath = [mChange copy];
          }
          info->_block(observer, object, changeWithKeyPath);
        } else if (info->_action) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
          [observer performSelector:info->_action withObject:change withObject:object];
#pragma clang diagnostic pop
        } else {
          [observer observeValueForKeyPath:keyPath ofObject:object change:change context:info->_context];
        }
      }
    }
  }
}
```

就是根据在 `_FBKVOInfo` 中存储的信息，进行相应的回调。

在持有 `KVOController` 的对象被销毁的时候，`KVOController` 也会相应的取消对所有观察对象的 `KVO` 防止出现 `Crash`：

```objc
- (void)dealloc
{
  [self unobserveAll];
  pthread_mutex_destroy(&_lock);
}

- (void)unobserveAll
{
  [self _unobserveAll];
}

- (void)_unobserveAll
{
  // lock
  pthread_mutex_lock(&_lock);

  NSMapTable *objectInfoMaps = [_objectInfosMap copy];

  // clear table and map
  [_objectInfosMap removeAllObjects];

  // unlock
  pthread_mutex_unlock(&_lock);

  _FBKVOSharedController *shareController = [_FBKVOSharedController sharedController];

  for (id object in objectInfoMaps) {
    // unobserve each registered object and infos
    NSSet *infos = [objectInfoMaps objectForKey:object];
    [shareController unobserve:object infos:infos];
  }
}
```

需要注意的是，使用 `KVOController` 观察自身属性的时候，会出现内存泄露的情况，这种情况下请记得使用 `KVOControllerNonRetaining` 来进行观察，同时在观察者 dealloc 的时候，调用 `unobserveAll` 方法。

## YYCategories

很多时候是否引入一个第三方库不是我们业务开发能决定的，而你又想在开发时安全方便的使用 `KVO`，你可以参考 `YYCategories` 里提供的方案来做，使用方法如下：

```
[self.person addObserverBlockForKeyPath:@"age" block:^(id  _Nonnull obj, id  _Nonnull oldVal, id  _Nonnull newVal) {
    NSLog(@"oldVal: %@, newVal: %@", oldVal, newVal);
}];
```

其实现原理也很简单，通过关联对象设置一个 `NSMutableDictionary`，这个字典以 `keyPath` 为 `key`，与这个 `key` 有关的所有 `block` 组成的可变数组为 `value`。 

```objc
// 添加 `KVO`
- (void)addObserverBlockForKeyPath:(NSString *)keyPath block:(void (^)(__weak id obj, id oldVal, id newVal))block {
    if (!keyPath || !block) return;
    _YYNSObjectKVOBlockTarget *target = [[_YYNSObjectKVOBlockTarget alloc] initWithBlock:block];
    NSMutableDictionary *dic = [self _yy_allNSObjectObserverBlocks];
    NSMutableArray *arr = dic[keyPath];
    if (!arr) {
        arr = [NSMutableArray new];
        dic[keyPath] = arr;
    }
    [arr addObject:target];
    [self addObserver:target forKeyPath:keyPath options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:NULL];
}
// 根据 `keyPath` 移除 `KVO`
- (void)removeObserverBlocksForKeyPath:(NSString *)keyPath {
    if (!keyPath) return;
    NSMutableDictionary *dic = [self _yy_allNSObjectObserverBlocks];
    NSMutableArray *arr = dic[keyPath];
    [arr enumerateObjectsUsingBlock: ^(id obj, NSUInteger idx, BOOL *stop) {
        [self removeObserver:obj forKeyPath:keyPath];
    }];
    
    [dic removeObjectForKey:keyPath];
}
// 移除 `KVO`
- (void)removeObserverBlocks {
    NSMutableDictionary *dic = [self _yy_allNSObjectObserverBlocks];
    [dic enumerateKeysAndObjectsUsingBlock: ^(NSString *key, NSArray *arr, BOOL *stop) {
        [arr enumerateObjectsUsingBlock: ^(id obj, NSUInteger idx, BOOL *stop) {
            [self removeObserver:obj forKeyPath:key];
        }];
    }];
    
    [dic removeAllObjects];
}
// 获取当前注册的所有 `KVO` `Block`
- (NSMutableDictionary *)_yy_allNSObjectObserverBlocks {
    NSMutableDictionary *targets = objc_getAssociatedObject(self, &block_key);
    if (!targets) {
        targets = [NSMutableDictionary new];
        objc_setAssociatedObject(self, &block_key, targets, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return targets;
}
```

而通知的回调则是放在 `_YYNSObjectKVOBlockTarget` 中的：

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    if (!self.block) return;
    
    BOOL isPrior = [[change objectForKey:NSKeyValueChangeNotificationIsPriorKey] boolValue];
    if (isPrior) return;
    
    NSKeyValueChange changeKind = [[change objectForKey:NSKeyValueChangeKindKey] integerValue];
    if (changeKind != NSKeyValueChangeSetting) return;
    
    id oldVal = [change objectForKey:NSKeyValueChangeOldKey];
    if (oldVal == [NSNull null]) oldVal = nil;
    
    id newVal = [change objectForKey:NSKeyValueChangeNewKey];
    if (newVal == [NSNull null]) newVal = nil;
    
    self.block(object, oldVal, newVal);
}
```

不过从源码上看，还是需要自己在 `dealloc` 的时候移除观察者的，不过这种方案的好处是可以多次监听同一个 `keyPath`，实现真正的一对多（虽然好像没啥荷包蛋用）。