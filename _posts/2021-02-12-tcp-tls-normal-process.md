---
layout: post
title: TCP - 基于 TLSv1.2 的正常交互流程
date: 2021-02-12
categories:
- 技术
tags: [TCP, Wireshark]
status: publish
type: post
published: true
---

# 一个基于 TLSv1.2 的正常交互的抓包示例

![正常交互.png](/images/blog_img/20210212/正常交互.png)

图中 124.64.x.x 是客户端 IP，172.16.x.x 是服务端 IP。

按编号来依次大致说明一下几个数据包：

3697 ~ 3699，握手建立连接；

3700 ~ 3708，建立 TLS；

3709 ~ 3723，正常的基于 TLS 的加密交互；

3727，客户端告知服务端加密通讯停止，SSL_shutdown；

3728 ~ 3730，挥手关闭连接。

---

# 说明

一次常规交互，抓个包为了方便与其他不正常流程做对比，同时也为了学习。

关于建立 TLS，参考[《如何用 wireshark 抓包 TLS 封包》](https://segmentfault.com/a/1190000018746027)中的“TLS 加密流程”章节，这里就不做摘抄了。

关于 3727 包中的 Encrypted Alert，参考 [TLSv1 Record Layer: Encrypted Alert](https://osqa-ask.wireshark.org/questions/38050/tlsv1-record-layer-encrypted-alert/38094)。
