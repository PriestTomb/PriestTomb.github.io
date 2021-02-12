---
layout: post
title: TCP - 三次握手失败，反复超时重传
date: 2021-02-12
categories:
- 技术
tags: [TCP, Wireshark]
status: publish
type: post
published: true
---

# 一个三次握手失败的抓包示例

![握手失败重传.png](/images/blog_img/20210212/握手失败重传.png)

图中 106.122.x.x 是客户端 IP，172.16.x.x 是服务端 IP。

按编号来依次说明一下几个数据包：

1.客户端发送[SYN]包，服务端正常收到；

2.服务端回复1[SYN, ACK]；

3.服务端超时重传[SYN, ACK]；

4.客户端重传[SYN]；

5.服务端收到4，回复4[SYN, ACK]；

6.客户端重传[SYN]；

7.服务端收到6，回复6[SYN, ACK]；

8.服务端超时重传[SYN, ACK]；

9.客户端重传[SYN]；

10.服务端收到9，回复9[SYN, ACK]；

11.客户端重传[SYN]；

12.服务端收到11，回复11[SYN, ACK]；

13.服务端超时重传[SYN, ACK]；

39.服务端超时重传[SYN, ACK]；

63.服务端超时重传[SYN, ACK]。

---

# 说明

可以看出是服务端返[SYN, ACK]失败，客户端收不到，导致的握手失败，以及多次的超时重传。此时，服务端的连接状态为 SYN_RECV。
