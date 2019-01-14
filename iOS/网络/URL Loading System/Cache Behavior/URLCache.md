# URLCache

将 URL 请求映射到缓存的响应对象的对象。

## 声明

```swift
class URLCache : NSObject
```

## 概览

URLCache 类通过将 NSURLRequest 对象映射到 CachedURLResponse 对象来实现对 URL 加载请求的响应的缓存。它提供复合内存和磁盘缓存，并允许你操作内存和磁盘部分的大小。你还可以控制持久存储缓存数据的路径。

> 注意：在 iOS 中，当系统磁盘空间不足时，可能会清除磁盘缓存，但仅当你的应用程序未运行时才会清除磁盘缓存。

## 线程安全

在 iOS 8 及更高版本以及 macOS 10.10 及更高版本中，URLCache 是线程安全的。

尽管可以安全地从多个执行上下文中同时调用 URLCache 实例方法，但请注意，在尝试读取或写入相同请求的响应时，cachedResponse(for:) 和 storeCachedResponse(_:for:) 等方法具有不可避免的竞争条件。

URLCache 的子类必须以这种线程安全的方式实现重写方法。

## 子类注释

URLCache 类旨在按原样使用，但如果你有特定需求，则可以将其子类化。例如，你可能希望筛选缓存的响应，或出于安全性或其他原因重新实现存储机制。

覆盖此类的方法时，请注意系统首选采用任务参数的方法，而不是那些方法。因此，你应该在子类化时覆盖基于任务的方法，如下所示：

- 在缓存中存储响应 - 覆盖基于任务的 storeCachedResponse(_:for:)，而不是基于请求的 storeCachedResponse(_:for:)。
- 从缓存中获取响应 - 覆盖 getCachedResponse(for:completionHandler:)，而不是 cachedResponse(for:)。
- 删除缓存的响应 - 覆盖基于任务的 removeCachedResponse(for:)，而不是基于请求的 removeCachedResponse(for:)。