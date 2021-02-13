---
layout: post
title: TCP - TLS 加密通讯中的丢包/乱序重传
date: 2021-02-13
categories:
- 技术
tags: [TCP, Wireshark]
status: publish
type: post
published: true
---

# 一个乱序重传的抓包示例

![丢包乱序.png](/images/blog_img/20210213/丢包乱序.png)

图中 106.122.x.x 是客户端 IP，172.16.x.x 是服务端 IP。

按编号来依次大致说明一下几个数据包：

14 ~ 16，握手建立连接；

17 ~ 22，建立 TLS；

23 ~ 26，正常的基于 TLS 的加密交互；

27，客户端发来[FIN, ACK]包，发起关闭连接，但 Seq=1199，上一包 26 的 Seq 为 1168，明显中间少了一个包，所以 Wireshark 标出 `TCP Previous segment not captured`；

28，服务端重复响应 Ack=1168，告知客户端有丢包；

29，客户端重传 Seq=1168，这里 Wireshark 把它标成了乱序，因为上一包的 Seq 是 1199；

30，服务端回复27[ACK]，从 Ack=1200 能看出，这里是收到客户端重传的包之后才回复的；

31，服务端正式响应客户端的关闭连接请求，回复[FIN, ACK]；

32，客户端发送 Encrypted Alert，告知服务端加密通讯停止，在详情中可以看到这个包的 Seq=1168，所以它是理应在编号 27 的那个包，因为种种原因，延迟到了现在才到，加上刚才编号 29 中客户端已经补发过一次 Seq=1168 的包，服务端也已经回过 ACK，Wireshark 标记了该包为虚假重传；

33，服务端再次重复响应 Ack=1200，告知客户端有丢包；

34，客户端回复[ACK]，完成四次挥手，连接关闭。

---

# 说明

正常的加密通讯结束后，在断开连接前，客户端一个 Encrypted Alert 包“迟到”，一个[ACK]包“丢失”，服务端两次[Dup ACK]提醒。

如有理解错误，欢迎评论一起讨论。
