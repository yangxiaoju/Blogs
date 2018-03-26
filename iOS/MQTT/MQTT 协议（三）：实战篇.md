# MQTT 协议（三）：实战篇

在进行了两篇博客的理论覆盖后，我们来写一个 MQTT 的 Demo，看看如何在 iOS 开发中使用这项技术。

## 寻找框架

在面向对象的开发中，框架是快速开发的利器。封装良好的框架可以有效地帮助我们避免直接接触协议底层的一些东西。

于是我打开了 Github，搜索 MQTT，找到了 Stars 最多的一个用 Objective-C 封装的 MQTT 框架：MQTT-Client-Framework。接下来让我们一起来学习如何使用这个框架~

## 导入框架

如果你比较喜欢用 Cocoapods（在工作中大家应该都会使用这个的，对吧？），可以把以下的语句写入 Podfile：

```
pod 'MQTTClient'
```

然后 `pod update`，完成之后我们就可以开心的使用这个框架啦~

## 使用框架

第一步，自然是导入 MQTT-Client-Framework 框架的主头文件：

```
#import <MQTTClient/MQTTClient.h>
```

第二步，在 AppDelegate 的类扩展里声明一个 MQTTSession 类的属性：

```
@property (nonatomic, strong) MQTTSession *mySession;
```

第三步，在 AppDelegate 中初始化 mySession：

首先，初始化一个 MQTTCFSocketTransport 对象，这个对象是用来记录 MQTT 协议中的一些属性，例如：host（服务器），port（端口）等。

```
MQTTCFSocketTransport *transport = [[MQTTCFSocketTransport alloc] init]; // 初始化对象

transport.host = @"localhost"; // 设置MQTT服务器的地址

transport.port = 1883; // 设置MQTT服务器的端口（默认是1883，当然，你也可以和你的后台好基友协商~）

self.mySession = [[MQTTSession alloc] init]; // 初始化MQTTSession对象

self.mySession.transport = transport; // 给mySession对象设置基本信息

self.mySession.delegate = self; // 设置mySession的代理为APPDelegate，同时不要忘记遵守协议~

[self.mySession connectAndWaitTimeout:30]; // 设定超时时长，如果超时则认为是连接失败，如果设为0则是一直连接。
```

## 订阅主题

正如我们在第一篇博文所说的，MQTT协议是一个发布/订阅式的协议，所以在创建和连接完成后，我们就开始订阅主题。

```
[self.mySession subscribeToTopic:@"example/#" atLevel:2 subscribeHandler:^(NSError *error, NSArray*gQoss) {//Topic则表示要订阅的主题，Level（qosLevel）表示消息等级。
    if (error) {
        NSLog(@"Subscription failed %@", error.localizedDescription);
    } else {
        NSLog(@"Subscription sucessfull! Granted Qos: %@", gQoss);
    }
}];
```

## 收到消息

```
- (void)newMessage:(MQTTSession *)session data:(NSData*)data onTopic:(NSString*)topic qos:(MQTTQosLevel)qos retained:(BOOL)retained mid:(unsignedint)mid {
    // 这个是代理回调方法，接收到的数据可以在这里进行处理。
}
```

## 发布消息

```
[self.mySession publishAndWaitData:data
                            onTopic:@"topic"
                             retain:NO
                                qos:MQTTQosLevelAtLeastOnce];
```

其中，data 表示你要发送的数据，topic 表示你向哪个主题发送数据，retain 如果是 YES, 数据会被存储在服务器，直到下一条 retain 也为YES的数据传入就会复写，qos 则是三种消息的等级，这些在第一篇博客中已经讲过了。

# 总结

大体上的用法就是这些，通过这几步就可以基本实现 MQTT 的基础功能了，至于保持心跳和使用 MQTT 框架搭建一个轻量级的即时通讯系统的问题，就靠同学们去框架里研究啦~

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~
