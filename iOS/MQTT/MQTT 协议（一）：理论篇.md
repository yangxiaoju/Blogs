# MQTT 协议（一）：理论篇

MQTT（Message Queuing Telemetry Transport，消息队列遥测传输）是 IBM 开发的一个即时通讯协议，有可能成为物联网的重要组成部分。该协议支持所有平台，几乎可以把所有联网物品和外部连接起来，被用来当做传感器和致动器（比如通过 Twitter 让房屋联网）的通信协议。

# MQTT 简介

早在1999年，IBM 的 Andy Stanford-Clark 博士以及 Arcom 公司 ArlenNipper 博士发明了 MQTT 技术 。

MQTT 的话题是他们在谈论开源物联网平台 Pachube 时提到的。Stanford-Clark 认为 Pachube 很酷，其不足之处是不具备真正的推送功能。你需要不断的进行轮询才能得到即时数据。这正是 MQTT 能够实现的，他提到了使用推送通信系统的石油管道检测系统。

# MQTT 应用

IBM 和 St. Jude 医疗中心通过 MQTT 开发了一套 Merlin 系统，该系统使用了用于家庭保健的传感器。St. Jude 医疗中心设计了一个叫做 Merlin@home 的心脏装置，这种无线发射器可以用来监控那些已经植入复律-除颤器和起搏器（两者都是基本的传感器）的心脏病人。

该产品利用 MQTT 把病人的即时更新信息传给医生/医院，然后医院进行保存。这样的话，病人就不用亲自去医院检查心脏仪器了，医生可以随时查看病人的数据，给出建议，病人在家里就可以自行检查。

IBM 称该发射器包括一个大型触摸屏，一个嵌入式键盘平台，以及一个 Linux 操作系统。

在未来几年，MQTT 的应用会越来越广，值得关注。

通过 MQTT 协议，目前已经扩展出了数十个 MQTT 服务器端程序，可以通过 PHP，JAVA，Python，C，C# 等系统语言来向 MQTT 发送相关消息。

此外，国内很多企业都广泛使用 MQTT 作为 Android 手机客户端与服务器端推送消息的协议。其中 Sohu，Cmstop 手机客户端中均有使用到 MQTT 作为消息推送协议。据 Cmstop 主要负责消息推送的高级研发工程师李文凯称，随着移动互联网的发展，MQTT 由于开放源代码，耗电量小等特点，将会在移动消息推送领域有更多的贡献，在物联网领域，传感器与服务器的通信，信息的收集，MQTT 都可以作为考虑的方案之一。在未来 MQTT 会进入到我们生活的各各方面。

如果需要下载 MQTT 服务器端，可以直接去 MQTT 官方网站点击 software 进行下载 MQTT 协议衍生出来的各个不同版本。

# MQTT 特点

MQTT 协议是为大量计算能力有限，且工作在低带宽、不可靠的网络的远程传感器和控制设备通讯而设计的协议，它具有以下主要的几项特性：

1.使用发布/订阅消息模式，提供一对多的消息发布，解除应用程序耦合。

这一点很类似于 XMPP，但是 MQTT 的信息冗余远小于 XMPP（因为 XMPP 使用的是 XML 这种格式来传递数据，你懂的）。

2.对负载内容屏蔽的消息传输。

3.使用 TCP/IP 提供网络连接。

主流的 MQTT 是基于 TCP 连接进行数据推送的，但是同样有基于 UDP 的版本，叫做 MQTT-SN 。这两种版本由于基于不同的连接方式，优缺点自然也就各有不同了。

4.有三种消息发布服务质量：

“至多一次”，消息发布完全依赖底层 TCP/IP 网络。会发生消息丢失或重复。这一级别可用于如下情况，环境传感器数据，丢失一次读记录无所谓，因为不久后还会有第二次发送。

这一种方式主要普通 APP 的推送，倘若你的智能设备在消息推送时未联网，推送过去没收到，再次联网也就收不到了。

“至少一次”，确保消息到达，但消息重复可能会发生。

这一种方式比较鸡肋，在我的想象中没能想到这种质量的发送在常规的 APP 开发中有什么用处。

“只有一次”，确保消息到达一次。这一级别可用于如下情况，在计费系统中，消息重复或丢失会导致不正确的结果。

这种最高质量的消息发布服务还可以用于即时通讯类的 APP 的推送，确保用户收到且只会收到一次。

5.小型传输，开销很小（固定长度的头部是2字节），协议交换最小化，以降低网络流量。

这就是为什么在介绍里说它非常适合“在物联网领域，传感器与服务器的通信，信息的收集”，要知道嵌入式设备的运算能力和带宽都相对薄弱，使用这种协议来传递消息再适合不过了。

6、使用 Last Will 和 Testament 特性通知有关各方客户端异常中断的机制。

Last Will：即遗言机制，用于通知同一主题下的其他设备发送遗言的设备已经断开了连接。

Testament：遗嘱机制，功能类似于 Last Will 。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~