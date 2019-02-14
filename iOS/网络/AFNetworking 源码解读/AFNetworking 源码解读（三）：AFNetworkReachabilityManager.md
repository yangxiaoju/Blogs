# AFNetworking 源码解读（三）：AFNetworkReachabilityManager

在开发应用中，很多时候都需要了解当前的网络状态是什么。AFNetworkReachabilityManager 就是提供这样功能的一个类，它提供了单例和创建对象两种方式供我们使用。

## 初始化方法

先来看指定初始化方法：

```objc
- (instancetype)initWithReachability:(SCNetworkReachabilityRef)reachability {
    self = [super init];
    if (!self) {
        return nil;
    }

    _networkReachability = CFRetain(reachability);
    self.networkReachabilityStatus = AFNetworkReachabilityStatusUnknown;

    return self;
}
```

指定初始化方法接受一个 SCNetworkReachabilityRef 参数，由于这是 Core Foundation 框架的类型，所以赋值时需要自己 retain（CFRetain(reachability)）。同时初始化 networkReachabilityStatus 为 AFNetworkReachabilityStatusUnknown。

AFNetworkReachabilityManager 同时还提供了可以传入 domain 或 address 参数的初始化方法，用于监控对某个域名的可连接性。

```objc
+ (instancetype)managerForDomain:(NSString *)domain {
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithName(kCFAllocatorDefault, [domain UTF8String]);

    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];
    
    CFRelease(reachability);

    return manager;
}

+ (instancetype)managerForAddress:(const void *)address {
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithAddress(kCFAllocatorDefault, (const struct sockaddr *)address);
    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];

    CFRelease(reachability);
    
    return manager;
}
```

AFNetworkReachabilityManager 还提供了无需自己输入 address 的初始化方法 manager：

```objc
+ (instancetype)manager
{
#if (defined(__IPHONE_OS_VERSION_MIN_REQUIRED) && __IPHONE_OS_VERSION_MIN_REQUIRED >= 90000) || (defined(__MAC_OS_X_VERSION_MIN_REQUIRED) && __MAC_OS_X_VERSION_MIN_REQUIRED >= 101100)
    struct sockaddr_in6 address;
    bzero(&address, sizeof(address));
    address.sin6_len = sizeof(address);
    address.sin6_family = AF_INET6;
#else
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_len = sizeof(address);
    address.sin_family = AF_INET;
#endif
    return [self managerForAddress:&address];
}
```

还有单例方法：

```objc
+ (instancetype)sharedManager {
    static AFNetworkReachabilityManager *_sharedManager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedManager = [self manager];
    });

    return _sharedManager;
}
```

### startMonitoring

startMonitoring 方法会启动对网络状态的监控。

```objc
- (void)startMonitoring {
    [self stopMonitoring];

    if (!self.networkReachability) {
        return;
    }

    __weak __typeof(self)weakSelf = self;
    AFNetworkReachabilityStatusCallback callback = ^(AFNetworkReachabilityStatus status) {
        __strong __typeof(weakSelf)strongSelf = weakSelf;

        strongSelf.networkReachabilityStatus = status;
        if (strongSelf.networkReachabilityStatusBlock) {
            strongSelf.networkReachabilityStatusBlock(status);
        }
        
        return strongSelf;
    };

    SCNetworkReachabilityContext context = {0, (__bridge void *)callback, AFNetworkReachabilityRetainCallback, AFNetworkReachabilityReleaseCallback, NULL};
    SCNetworkReachabilitySetCallback(self.networkReachability, AFNetworkReachabilityCallback, &context);
    SCNetworkReachabilityScheduleWithRunLoop(self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),^{
        SCNetworkReachabilityFlags flags;
        if (SCNetworkReachabilityGetFlags(self.networkReachability, &flags)) {
            AFPostReachabilityStatusChange(flags, callback);
        }
    });
}
```

startMonitoring 做了如下几件事情：

1. 创建一个 AFNetworkReachabilityStatusCallback 类型的回调，用于更改现有的网络状态和调用 networkReachabilityStatusBlock。
2. 创建 SCNetworkReachabilityContext 上下文，是一个结构体。结构体声明如下：

```objc
typedef struct {
     CFIndex		version; // 结构体版本号
     void *		__nullable info; // 回调 block
     const void	* __nonnull (* __nullable retain)(const void *info); // 用于管理内存的 retain 回调
     void		(* __nullable release)(const void *info); // 用于管理内存的 release 回调
     CFStringRef	__nonnull (* __nullable copyDescription)(const void *info); // 描述信息
 } SCNetworkReachabilityContext;
 ```

3. 设置网络情况改变时的回调函数：

```
static void AFNetworkReachabilityCallback(SCNetworkReachabilityRef __unused target, SCNetworkReachabilityFlags flags, void *info) {
    AFPostReachabilityStatusChange(flags, (__bridge AFNetworkReachabilityStatusCallback)info);
}
```

4. 在 mainRunloop 的 kCFRunLoopCommonModes 中开始监控。
5. 获取当前的网络状态，并通过 AFPostReachabilityStatusChange 发出通知：

```objc
static void AFPostReachabilityStatusChange(SCNetworkReachabilityFlags flags, AFNetworkReachabilityStatusCallback block) {
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusForFlags(flags);
    dispatch_async(dispatch_get_main_queue(), ^{
        AFNetworkReachabilityManager *manager = nil;
        if (block) {
            manager = block(status);
        }
        NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
        NSDictionary *userInfo = @{ AFNetworkingReachabilityNotificationStatusItem: @(status) };
        [notificationCenter postNotificationName:AFNetworkingReachabilityDidChangeNotification object:manager userInfo:userInfo];
    });
}
```

### stopMonitoring

stopMonitoring 相对来说简单一些，如下所示：

```objc
- (void)stopMonitoring {
    if (!self.networkReachability) {
        return;
    }

    SCNetworkReachabilityUnscheduleFromRunLoop(self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);
}
```