---
layout: post
title: Java dump问题（SIGSEGV、libresolv.so.2+0x7a91、__libc_res_nquery+0x1c1）
date: 2019-03-31
categories:
- 后端技术
tags: [Java, dump]
status: publish
type: post
published: true
---

写了个 Quartz 实现的定时程序，打包到服务器上启动时发现会崩溃，本地测试都没毛病，之前服务器上启动也不会崩，搞不懂是不是服务器最近改了什么配置，反正先看看报错：

```java
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGSEGV (0xb) at pc=0x00007fcc66899a91, pid=6880, tid=0x00007fcc66def700
#
# JRE version: Java(TM) SE Runtime Environment (8.0_171-b11) (build 1.8.0_171-b11)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.171-b11 mixed mode linux-amd64 compressed oops)
# Problematic frame:
# C  [libresolv.so.2+0x7a91]  __libc_res_nquery+0x1c1
#
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# An error report file with more information is saved as:
# /home/xxx/hs_err_pid6880.log
#
```

用 `SIGSEGV`、`libresolv.so.2+0x7a91`、`__libc_res_nquery+0x1c1` 几个关键词搜了一下，没发现有什么直观的答案

打开 error 日志看一下具体的报错模块：

```java
Java frames: (J=compiled Java code, j=interpreted, Vv=VM code)
j  java.net.Inet6AddressImpl.lookupAllHostAddr(Ljava/lang/String;)[Ljava/net/InetAddress;+0
j  java.net.InetAddress$2.lookupAllHostAddr(Ljava/lang/String;)[Ljava/net/InetAddress;+4
j  java.net.InetAddress.getAddressesFromNameService(Ljava/lang/String;Ljava/net/InetAddress;)[Ljava/net/InetAddress;+51
J 2194 C1 java.net.InetAddress.getAllByName0(Ljava/lang/String;Ljava/net/InetAddress;Z)[Ljava/net/InetAddress; (57 bytes) @ 0x00007ff30203527c [0x00007ff302035020+0x25c]
J 2193 C1 java.net.InetAddress.getAllByName(Ljava/lang/String;Ljava/net/InetAddress;)[Ljava/net/InetAddress; (387 bytes) @ 0x00007ff3020390c4 [0x00007ff3020371a0+0x1f24]
J 2387 C1 java.net.InetSocketAddress.<init>(Ljava/lang/String;I)V (47 bytes) @ 0x00007ff3020e0fb4 [0x00007ff3020e0d80+0x234]
j  sun.net.NetworkClient.doConnect(Ljava/lang/String;I)Ljava/net/Socket;+92
j  sun.net.www.http.HttpClient.openServer(Ljava/lang/String;I)V+4
j  sun.net.www.http.HttpClient.openServer()V+114
j  sun.net.www.http.HttpClient.<init>(Ljava/net/URL;Ljava/net/Proxy;I)V+125
j  sun.net.www.http.HttpClient.New(Ljava/net/URL;Ljava/net/Proxy;IZLsun/net/www/protocol/http/HttpURLConnection;)Lsun/net/www/http/HttpClient;+259
j  sun.net.www.http.HttpClient.New(Ljava/net/URL;Ljava/net/Proxy;ILsun/net/www/protocol/http/HttpURLConnection;)Lsun/net/www/http/HttpClient;+5
j  sun.net.www.protocol.http.HttpURLConnection.getNewHttpClient(Ljava/net/URL;Ljava/net/Proxy;I)Lsun/net/www/http/HttpClient;+4
j  sun.net.www.protocol.http.HttpURLConnection.plainConnect0()V+357
j  sun.net.www.protocol.http.HttpURLConnection.plainConnect()V+71
j  sun.net.www.protocol.http.HttpURLConnection.connect()V+20
j  sun.net.www.protocol.http.HttpURLConnection.getInputStream0()Ljava/io/InputStream;+195
j  sun.net.www.protocol.http.HttpURLConnection.getInputStream()Ljava/io/InputStream;+52
j  org.quartz.utils.UpdateChecker.getUpdateProperties(Ljava/net/URL;)Ljava/util/Properties;+13
j  org.quartz.utils.UpdateChecker.doCheck()V+17
j  org.quartz.utils.UpdateChecker.checkForUpdate()V+1
j  org.quartz.utils.UpdateChecker.run()V+1
j  java.util.TimerThread.mainLoop()V+221
j  java.util.TimerThread.run()V+1
v  ~StubRoutines::call_stub
```

最初没反应过来这个跟我写的代码有什么关系，就各种测试：把我本机的 JDK 也改成服务器上的 JDK1.8.0_171、把开启定时任务的代码各种改，但始终都报这个错

最后灵光一现，想起来本机测试的时候，每次在程序启动时都会看到一条类似这样的日志：

```java
14:02:49.027 [Timer-0] DEBUG org.quartz.utils.UpdateChecker - Checking for available updated version of Quartz...
14:02:49.607 [Timer-0] DEBUG org.quartz.utils.UpdateChecker - Quartz version update check failed: Server returned HTTP response code: 403 for URL: http://www.terracotta.org/kit/reflector?kitID=quartz&pageID=update.properties&id=blablabla
```

结合上面的报错日志，刚好也是由于 Quartz 的 `UpdateChecker` 类导致的网络相关错误，才明白。。因为服务器没有联网，这个联网更新检测不知道为毛就导致了 JVM 崩溃

参考一篇[博客](https://www.tanelikorri.com/blog/2011/09/quartz-and-the-update-checker/)，关闭 Quartz 启动时的版本更新检测，再次丢到服务器上启动，顺利

搞定！
