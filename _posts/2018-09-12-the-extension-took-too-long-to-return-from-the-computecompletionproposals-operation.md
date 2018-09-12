---
layout: post
title: Eclipse代码补全超时报错问题(too long to return from the computeCompletionProposals operation)
date: 2018-09-12
categories:
- 编程工具
tags: [Eclipse]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

## 又双叒叕报错啊

近期学习 Spring Boot 和 Spring Cloud，在 Eclipse 下经常出现代码补全的时候卡未响应然后报错，就是这个

![Eclipse报错](http://oxujjb0ls.bkt.clouddn.com/image/eclipse%E4%BB%A3%E7%A0%81%E8%A1%A5%E5%85%A8%E6%8A%A5%E9%94%99/%E6%8A%A5%E9%94%99%E4%BF%A1%E6%81%AF.png)

---

## 解决一下

网上随便搜搜，这是个历史悠久的问题（很奇怪我以前没遇到过。。），最开始看到的解决方案是点报错里的那个链接，到设置页面取消 `Java Proposals(Code Recommenders)`，同时勾选另外两个 `Java Proposals`

![解决方案1](http://oxujjb0ls.bkt.clouddn.com/image/eclipse%E4%BB%A3%E7%A0%81%E8%A1%A5%E5%85%A8%E6%8A%A5%E9%94%99/%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%881.png)

实际证明可以"解决"，确实不会卡死了，但是。。会造成代码提示不再"智能"，想补全的内容还要自己按好多次 `Alt+/` 才能找到，这就很尴尬了，这根本就不算解决方案！

综合测试了其他很多方案，我的总结是这三个方面可以尽可能避免这个BUG：

* 加大 Eclipse 的运行内存

> 修改 eclipse.ini 中的 `-Xmx` 和 `-Xms`，结合自己电脑的内存情况，可以稍微加大一些

* 加大自动补全提示的延迟时间

> Window -> Preferences -> Java -> Editor -> Content Assist，将 `Auto activation delay(ms)` 加大一些，比如 500 甚至 1000

* 关闭不必要的 Proposal

> Window -> Preferences -> Java -> Editor -> Content Assist -> Advanced，这个设置是避免 BUG 的关键，越大的工程涉及到的代码、jar 包越多，每次自动补全时，Eclipse 都要去遍历很多东西。像我目前的工作，其实没必要关联其他不相关的 Proposals，关联的越多，Eclipse 也是要多查很多东西，也就导致更慢。所以仅保留自己开发所必需的 Proposals 就行了

---

## 搞定

所以最终我的解决方案就是：

* 把 Eclipse 的内存调大到 2G

* 自动补全的延迟时间改成 500

* 保留最少的推荐方案，如下

![我的解决方案](http://oxujjb0ls.bkt.clouddn.com/image/eclipse%E4%BB%A3%E7%A0%81%E8%A1%A5%E5%85%A8%E6%8A%A5%E9%94%99/%E6%88%91%E7%9A%84%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88.png)

---

## 参考

[https://segmentfault.com/a/1190000005693173](https://segmentfault.com/a/1190000005693173)
