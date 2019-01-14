# HTTPURLResponse

与 HTTP 协议 URL 加载请求的响应关联的元数据。

## 声明

```swift
class HTTPURLResponse : URLResponse
```

## 概览

HTTPURLResponse 类是 URLResponse 的子类，它提供访问特定于 HTTP 协议响应的信息的方法。每当你发出 HTTP URL 加载请求时，从 URLSession，NSURLConnection 或 NSURLDownload 类返回的任何响应对象都是 HTTPURLResponse 类的实例。