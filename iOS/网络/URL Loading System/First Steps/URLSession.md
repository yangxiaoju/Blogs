# URLSession

协调一组相关网络数据传输任务的对象。

## 声明

```swift
class URLSession : NSObject
```

## 概览

URLSession 类和相关类提供了一个 API，用于从 URL 指示的端点下载数据并将数据上载到端点。该 API 还可让你的应用在你的应用未运行时执行后台下载，或者在 iOS 中，在你的应用暂停时执行后台下载。一组丰富的 delegate 方法支持身份验证，并允许你的应用程序收到重定向等事件的通知。

> 重要：URLSession API 涉及许多不同的类，它们以相当复杂的方式协同工作，如果你自己阅读参考文档，则可能并不明显。在使用 API 之前，请阅读 URL Loading System 主题中的概述。First Steps，Uploading 和 Downloading 部分中的文章提供了使用 URLSession 执行常见任务的示例。

使用 URLSession API，你的应用程序会创建一个或多个会话，每个会话都会协调一组相关的数据传输任务。例如，如果你正在创建 Web 浏览器，则你的应用可能会为每个选项卡或窗口创建一个会话，或者一个会话用于交互式使用，另一个会话用于后台下载。在每个会话中，你的应用程序会添加一系列任务，每个任务都代表对特定 URL 的请求（如果需要，可以在 HTTP 重定向之后）。

## URL 会话的类型

给定 URL 会话中的任务共享公共会话配置对象，该对象定义连接行为，例如要对单个主机进行的最大同时连接数，是否允许通过蜂窝网络进行连接等。

URLSession 具有基本请求的单例共享会话（没有配置对象）。它不像你创建的会话那样可自定义，但如果你的要求非常有限，它可以作为一个很好的起点。你可以通过调用共享类方法来访问此会话。对于其他类型的会话，你可以使用以下三种配置之一实例化 URLSession：

- 默认会话的行为与共享会话非常相似，但允许更多配置，并允许你使用 delegate 逐步获取数据。
- 临时会话类似于共享会话，但不会将缓存，cookie 或凭据写入磁盘。
- 通过后台会话，你可以在应用未运行时在后台执行内容上传和下载。

有关创建每种配置类型的详细信息，请参阅 URLSessionConfiguration 类中的 Creating a Session Configuration Object。

## URL 会话任务的类型

在会话中，你可以创建任务，可选择将数据上载到服务器，然后从服务器检索数据，作为磁盘上的文件或内存中的一个或多个 NSData 对象。URLSession API 提供三种类型的任务：

- 数据任务使用 NSData 对象发送和接收数据。数据任务旨在用于对服务器的简短且通常是交互式的请求。
- 上传任务类似于数据任务，但它们也会发送数据（通常以文件的形式），并在应用程序未运行时支持后台上传。
- 下载任务以文件的形式检索数据，并在应用程序未运行时支持后台下载和上传。

## 使用会话代理

会话中的任务还共享一个公共 delegate，允许你在发生各种事件时提供和获取信息 - 身份验证失败，数据从服务器到达，数据何时准备缓存等等。如果你不需要 delegate 提供的任何功能，则可以使用此 API，而无需在创建会话时传递 nil。

重要：会话对象保持对 delegate 的强引用，直到你的应用退出或显式使会话无效。如果你没有使会话无效，那么你的应用程序会在内存泄漏之前泄漏内存。

## 异步性和 URL 会话

与大多数网络 API 一样，URLSession API 也是高度异步的。它会以两种方式之一将数据返回给你的应用，具体取决于你调用的方法：

- 通过在传输成功完成或出现错误时调用 completion handler block。
- 通过在接收数据时以及传输完成时调用会话 delegate 上的方法。

除了向 delegates 提供此信息之外，URLSession API 还提供状态和进度属性，如果你需要根据任务的当前状态做出编程决策（可以随时告知其状态可能会发生变化），你可以查询这些属性。

URL 会话还支持取消，重新启动，恢复和暂停任务，并提供在中断时恢复暂停，取消或失败的下载的功能。

## 协议支持

URLSession 类本身支持数据，文件，ftp，http 和 https URL 方案，并且在用户的系统首选项中配置了对代理服务器和 SOCKS 网关的透明支持。

URLSession 支持 HTTP/1.1，SPDY 和 HTTP/2 协议。RFC 7540 描述了 HTTP/2 支持，并且需要服务器支持应用层协商协商（ALPN）或下一协议协商（NPN）。

你还可以通过继承 URLProtocol 来添加对你自己的自定义网络协议和 URL 方案的支持（供你的应用程序专用）。

## 应用运输安全（ATS）

从 iOS 9.0 和 OS X 10.11 开始，默认情况下为 URLSession 创建的所有 HTTP 连接启用一项名为 App Transport Security（ATS）的新安全功能。ATS 要求 HTTP 连接使用 HTTPS（RFC 2818）。

有关更多信息，请参阅 Information Property List Key Reference 中的 NSAppTransportSecurity。

## NSCopying 行为

会话和任务对象符合 NSCopying 协议，如下所示：

- 当你的应用程序复制会话或任务对象时，你将获得相同的对象。
- 当你的应用程序复制配置对象时，你将获得一个可以单独修改的新副本。

## 线程安全

URL 会话 API 本身是完全线程安全的。你可以在任何线程上下文中自由创建会话和任务。当你的 delegate 方法调用提供的 completion handlers 时，工作将自动安排在正确的 delegate 队列中。

> 警告：系统可以在辅助线程上调用 urlSessionDidFinishEvents(forBackgroundURLSession:)  会话 delegate 方法。但是，在 iOS 中，你的该方法的实现可能需要调用在你的应用程序中提供给你的完成处理程序 application(_:handleEventsForBackgroundURLSession:completionHandler:)  app delegate方法。你必须在主线程上调用该 completion handler。

