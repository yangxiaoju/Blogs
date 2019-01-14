# CachedURLResponse

对 URL 请求的缓存响应。

## 声明

```swift
class CachedURLResponse : NSObject
```

## 概览

CachedURLResponse 对象以 URLResponse 对象的形式提供服务器的响应元数据，以及包含实际缓存的内容数据的 NSData 对象。其存储策略确定响应是应缓存在磁盘上，内存中还是根本不缓存。

缓存响应还包含用户信息字典，你可以在其中存储有关缓存项目的应用程序特定信息。

URLCache 类存储和检索 CachedURLResponse 的实例。