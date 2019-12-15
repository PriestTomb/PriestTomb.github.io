---
layout: post
title: 验证Nginx的长连接(keepalive)配置
date: 2018-11-01
categories: [Web服务器]
tags: [Nginx, keepalive, wireshark]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

## 写在前面

**注：文章最后更新于 2019-12-15**

最近都在折腾 Nginx 服务器的学习和测试，前几天稍微温习了一下计算机网络方面的知识（一方面是兴趣，一方面是这次学习过程中因为这些计算机基础遗忘，有很多细节问题让人很懵逼），也在 Linux 上试了一下 `tcpdump` 命令，通过抓包来验证自己之前的各种猜想（因为不懂所以瞎猜），分析数据包依旧是用的 Wireshark

之前一直以为只要使用 Http1.1 协议就可以复用连接，节省反复握手挥手的时间消耗，但抓包后才发现，"简简单单"的长连接使用，还真是涉及了 Nginx 、 JMeter 、 Tomcat 里的很多配置啊

* JMeter 虽然在 Http 请求的配置中默认勾选了使用 KeepAlive，但是实际使用中并没有生效

* Nginx 涉及到面向客户端的配置： `keepalive_timeout` 和 `keepalive_requests`，面向后端服务器的配置： `keepalive`（1.15.3之后，upstream 模块也新增了 `keepalive_timeout` 和 `keepalive_requests`，本篇暂不涉及）、 `proxy_http_version` 和 `proxy_set_header`

* Tomcat 默认也有一个 `keepAliveTimeout` 配置

知道了这些相关配置后，一方面想实战下 `tcpdump` 和 Wireshark 的使用，一方面也想用数据验证下 Nginx 的这三个配置，所以就有了接下来的内容

---

## 环境说明

* Nginx 1.14.0

* Linux 2.6.32

* JMeter 5.0

* Wireshark 2.6.4

* JMeter 所在主机 IP 172.16.40.199

* Tomcat 所在主机 IP 172.16.40.201

* Nginx 所在主机 IP 172.16.40.224

---

## 0. 和客户端的长连接超时时间

#### 指令说明

> 语法:	 keepalive_timeout *timeout [header_timeout]*;
>
> 默认值:	keepalive_timeout 75s;
>
> 上下文:	http, server, location
>
> 第一个参数设置客户端的长连接在服务器端保持的最长时间（在此时间客户端未发起新请求，则长连接关闭）。 第二个参数为可选项，设置“Keep-Alive: timeout=time”响应头的值。 可以为这两个参数设置不同的值。
>
> “Keep-Alive: timeout=time”响应头可以被 Mozilla 和 Konqueror 浏览器识别和处理。 MSIE 浏览器在大约60秒后会关闭长连接。

#### 测试计划

keepalive_timeout = 1

客户端持续发起 http 请求，且不主动关闭连接，观察连接的释放时间

#### 测试过程

用 JMeter 测试时发现不能维持长连接，在请求结束时都由 JMeter 自己发起连接关闭，按官方 wiki 说明修改 jmeter.properties 和 user.properties 中的相关配置，测试一下没看到生效，最后还是用 Apache 的 [HttpClient](https://hc.apache.org/httpcomponents-client-ga/) 自己写了点代码才做了这个测试

代码中循环启动多个线程，每个线程启动后不间断发起10次请求，且执行后没有主动关闭 client 和 response

#### 测试结果

![验证keepalive_timeout](/images/blog_img/20181101/验证keepalive_timeout.png)

#### 结果说明

如上所述，测试客户端发起数次请求，请求结束时没有关闭连接（这里有个小问题，连接没有复用，可能是需要配置 client 连接池，我暂时没去折腾）

1秒(超时)后，由 Nginx 服务器发起关闭请求

---

## 1. 和客户端的长连接请求次数

#### 指令说明

> 语法:	keepalive_requests *number*
>
> 默认值:	keepalive_requests 100
>
> 上下文:	http, server, location
>
> 这个指令出现在版本 0.8.0
>
> 设置通过一个长连接可以处理的最大请求数。 请求数超过此值，长连接将关闭。

#### 测试计划

keepalive_requests = 5

使用 `curl` 命令，发送8次(多于 keepalive_requests 个数的)请求，观察请求5次后连接是否断开

#### 测试过程

最初使用 for 循环执行 `curl` 命令，发现命令执行完就立刻释放了连接，在网上查了下，可以在一个命令里访问多次，例如：

```
curl http://172.16.40.224:6066/NginxTest/hi http://172.16.40.224:6066/NginxTest/hi http://172.16.40.224:6066/NginxTest/hi http://172.16.40.224:6066/NginxTest/hi http://172.16.40.224:6066/NginxTest/hi http://172.16.40.224:6066/NginxTest/hi http://172.16.40.224:6066/NginxTest/hi http://172.16.40.224:6066/NginxTest/hi
```

这样的话，就会等8次请求都结束，才关闭连接

#### 测试结果

![验证keepalive_requests](/images/blog_img/20181101/验证keepalive_requests.png)

#### 结果说明

可以看出第一个 TCP 连接共发起了5次 Http 请求，紧接着由 Nginx 发起了关闭

第二个 TCP 连接发起了剩余的3次 Http 请求，紧接着由客户端发起了关闭

---

## 2. 和后端服务器的长连接个数

#### 指令说明

> 语法:	keepalive *connections*
>
> 默认值:	—
>
> 上下文:	upstream
>
> 这个指令出现在版本 1.1.4
>
> connections 参数设置每个 worker 进程与后端服务器保持连接的最大数量。这些保持的连接会被放入缓存。 如果连接数大于这个值时，最久未使用的连接会被关闭。

#### 测试计划

keepalive = 10

用 JMeter 启动多于 keepalive 个数的测试进程，观察测试结束后的连接关闭情况

#### 测试过程

使用 JMeter 的 GUI 客户端编辑一个测试脚本开启50个进程，循环测试120秒

#### 测试结果

![验证keepalive1](/images/blog_img/20181101/验证keepalive1.png)

#### 结果说明

可以看出截至118秒(JMeter 逐渐停止发起测试请求)，都在逐个关闭 TCP 连接，截图部分大都是后端服务器响应的 FIN 包，稍微往前一点能看到都是由 Nginx 主动发起的关闭请求

从138秒开始（后端 Tomcat 服务器的长连接超时时间默认20秒），剩余的10个长连接被后端服务器发起了关闭请求

**---新补充截图及说明---**

#### 测试结果2

![验证keepalive2](/images/blog_img/20181101/验证keepalive2.png)

#### 结果说明2

设置了 keepalive = 10，查看其中一个活跃的端口48443，可以看到该端口有被重用，处理了数次请求，同样，在最后一次使用后20秒，从后端服务器发起了关闭 TCP 连接的请求

#### 测试结果3

![验证keepalive3](/images/blog_img/20181101/验证keepalive3.png)

#### 结果说明3

不设置 keepalive，即 Nginx 不保持和后端的长连接，可以看到即使使用了 HTTP/1.1 协议，每个端口在建立连接->发送请求后，立刻由 Nginx 发起了关闭请求

---

## 3. 和后端服务器之间的 HTTP 协议

#### 指令说明

> 语法：proxy_http_version 1.1;
>      proxy_set_header Connection "";
>
> 上下文：location

#### 测试结果及说明

在 Nginx 1.14 中，Nginx 和后端服务器之间的协议默认还是 HTTP/1.0，所以想要使用长连接，不仅要配置长连接的个数，最关键的是要使用 HTTP/1.1 协议，官方提供的配置就是指定版本为1.1，并将 Connection 头设置为空

---

## 最后

测试过程涉及到的 `tcpdump` 命令是

```
sudo tcpdump -i eth0 host 172.16.40.xxx -w dumplog.pcap

-i 指定网卡
host 指定抓取本机和哪个 IP 之间的数据包
-w 将抓取结果保存到 pcap 文件，用于使用 Wireshark 查看
```

通过上面的几个测试，理解和验证了 Nginx 分别在面向客户端和面向后端服务器时的长连接使用情况
