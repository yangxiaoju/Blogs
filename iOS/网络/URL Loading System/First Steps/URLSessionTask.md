# URLSessionTask

在 URL 会话中执行的任务，如下载特定资源。

## 声明

```swift
class URLSessionTask : NSObject
```

## 概览

URLSessionTask 类是 URL 会话中任务的基类。任务始终是会话的一部分; 通过在 URLSession 实例上调用任务创建方法之一来创建任务。你调用的方法决定了任务的类型。

- 使用 URLSession 的 dataTask(with:) 和相关方法来创建 URLSessionDataTask 实例。数据任务请求资源，将服务器的响应作为一个或多个 NSData 对象返回到内存中。它们在默认，短暂和共享会话中受支持，但在后台会话中不受支持。
- 使用 URLSession 的 uploadTask(with:from:) 和相关方法创建 URLSessionUploadTask 实例。上传任务就像数据任务，除了它们使提供请求体更容易，因此你可以在检索服务器响应之前上传数据。此外，后台会话支持上传任务。
- 使用 URLSession 的 downloadTask(with:) 和相关方法创建 URLSessionDownloadTask 实例。下载任务将资源直接下载到磁盘上的文件。任何类型的会话都支持下载任务。
- 使用 URLSession 的 streamTask(withHostName:port:) 或 streamTask(with:) 创建 URLSessionStreamTask 实例。流任务从主机名和端口或网络服务对象建立 TCP/IP 连接。

创建任务后，通过调用 resume() 方法启动它。然后会话保持对任务的强引用，直到请求完成或失败; 你不需要维护对任务的引用，除非它对你的应用程序的内部簿记有用。

> 注意：所有任务属性都支持键值观察。