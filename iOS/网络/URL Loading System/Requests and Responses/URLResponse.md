# URLResponse

与 URL 加载请求的响应关联的元数据，与协议和 URL 方案无关。

## 声明

```swift
class URLResponse : NSObject
```

## 概览

相关的 HTTPURLResponse 类是 URLResponse 的常用子类，其对象表示对 HTTP URL 加载请求的响应，并存储其他特定于协议的信息，例如响应头。每当你发出 HTTP 请求时，你获得的 URLResponse 对象实际上是 HTTPURLResponse 类的实例。

> 注意：URLResponse 对象不包含表示 URL 内容的实际字节。相反，数据通过 delegate 调用一次返回一个片段，或者在请求完成时完整返回，具体取决于用于启动请求的方法和类。阅读 Fetching Website Data into Memory，了解从 URL 加载接收内容数据的各种方法。