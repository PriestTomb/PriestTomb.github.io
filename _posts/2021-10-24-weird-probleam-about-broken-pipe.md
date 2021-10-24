---
layout: post
title: form 表单默认提交导致的诡异 Broken pipe 异常问题
date: 2021-10-24
categories:
- 技术
tags: [html, thymeleaf, exception]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

# 1 诡异的异常

一个使用了 Thymeleaf 的 MVC 项目，生产环境的日志中偶尔会看到一些奇怪的报错，说 Thymeleaf 在返回某些页面的时候客户端中断了连接，导致失败。最近在修改某个功能模块时，在本地测试时居然也发现了这个问题，异常日志大概这样（省略了过多的堆栈信息）：

```
[2021-10-13 10:41:29,376][qtp1459864059-5191][WARN][org.eclipse.jetty.server.HttpChannel:573] /list
org.springframework.web.util.NestedServletException: Request processing failed; nested exception is org.thymeleaf.exceptions.TemplateInputException: An error happened during template parsing (template: "class path resource [templates/html/list.html]")
    ...
Caused by: org.thymeleaf.exceptions.TemplateInputException: An error happened during template parsing (template: "class path resource [templates/html/list.html]")
    ...
Caused by: org.attoparser.ParseException: An error happened during template rendering (template: "html/list" - line 241, col 21)
    ...
Caused by: org.thymeleaf.exceptions.TemplateOutputException: An error happened during template rendering (template: "html/list" - line 241, col 21)
    ...
Caused by: org.eclipse.jetty.io.RuntimeIOException: org.eclipse.jetty.io.EofException
    ...
Caused by: org.eclipse.jetty.io.EofException
    ...
Caused by: java.io.IOException: Broken pipe
    ...
```

虽然后台报错，但奇怪的是前端看不出有什么问题，页面能正常刷新显示，于是勾起了好奇心，想看看到底是什么鬼 BUG 才能有这么神奇的现象。

---

# 2 排查问题

经过了一番折腾和测试，最终还是发现了问题产生的原因，其中也走了一些无用的弯路。

## 2.1 调试代码

因为报错的日志中一直都显示出刷新页面内容第多少行多少列的时候产生了异常，所以最初忽视了 Broken pipe 问题，直接去调试 Thymeleaf 写文件的代码，加断点一点点看。

不过实际调试起来不太顺利，很麻烦，所以消耗了不少时间，最后也没调试出结果来。

## 2.2 F12 大法

习惯性打开 F12 查看一下前端究竟有什么表现，发现了在后端服务有报错的时候，前端会有一条失败的 XHR 请求，然后会有一条成功的正常请求。

对比两条请求，发现失败的没有带请求参数。

## 2.3 祭出 Wireshark

绕了一圈，回过头来盯着 `Broken pipe` 发呆，既然是连接的问题，那还是得抓包看一下，看看连接到底是怎么断开的。

于是启动 Wireshark，使用 `Adapter for loopback traffic capture` 抓取访问本地环回地址的网络包（新版本的 Wireshark 会提示是否安装 Npcap，选择这个才可以，老版本使用的 WinPcap 不支持）。

测试过后发现，失败的那个连接会在建立后，刚刚由客户端发起请求（如上所述，没有带请求参数），就有 [FIN, ACK] 包，然后就没有然后了。同时发现这个断开连接的请求包看着很奇怪，好像是只有两包（Wireshark 截图在下面 2.5 节）。

而如果主动调用这个没带参数、怀疑有问题的接口地址，整个流程又是正常的，页面可正常刷新显示，不会报错。

## 2.4 发现细节

在尝试几个角度而没有什么发现之后，这才想起来要仔细复现一下 BUG 出现的操作流程，后来发现是搜索框回车查询时有问题。

然后，在后端接口服务加了一个断点，确定了是有两个请求到了后端服务，第一个来的，参数为 null，第二个来的，参数正常。

再然后，在前端 js 中回车监听事件里加一个 alert，借弹窗控制请求的发出。此时发现，明明弹窗了，后端接口依然被调用到了，而且就是那个参数为 null 的请求。

接下来，反复尝试几次后，终于找到了 BUG 是怎么出现的。

## 2.5 复现问题

![复现流程.png](/images/blog_img/20211024/复现流程.png)

上面的网络包中分为三块，其中的操作细节分别为：

1) 第一块：在前端页面的输入框中按下回车，前端页面弹出弹窗，正确的 XHR 请求还没有发出，但此时后端接口被调用，但被断点拦截，没有执行；

2) 第二块：关闭页面中的弹窗，正确的 XHR 请求发出，同样后端接口被调用，但被断点拦截，没有执行；

3) 第三块：后端执行第一个接口调用，此时错误出现，控制台输出错误日志，网络包提示 RST。

---

<h3>混入防转防爬防抄袭声明：本文<a href="https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2021/10/24/weird-probleam-about-broken-pipe/">《form 表单默认提交导致的诡异 Broken pipe 异常问题》</a>首发且仅发布于<a href="https://priesttomb.github.io/">没有名字的博客</a></h3>

---
# 3 1/2真相

从上面复现问题的流程中，可以肯定，第二个 XHR 请求的发出，导致了第一个请求主动断开了连接，即编号 1817 和 1818 两包，客户端 61180 端口发出 [FIN, ACK] 关闭连接，服务端 8077 端口回复 [ACK] 表示 OK。

而这两包也是让我依然困惑的，印象中 Wireshark 会把挥手四包展示成三包，但这次抓包怎么就只剩两包了。。服务端回复 [ACK] 之后，客户端怎么没有后续了？以至于我最初一直觉得是服务端发起的关闭，编号 1817 是客户端表示收到并关闭，但又找不到服务端发起的第一包 [FIN]。。

虽然抓包的结果还没搞清楚，但至少把问题定位在了前端发起两次请求上，于是继续搜索类似的案例，最终发现了一个所谓的 form 表单默认提交的东西，在 [stackoverflow 上](https://stackoverflow.com/questions/1131781/is-it-a-good-practice-to-use-an-empty-url-for-a-html-forms-action-attribute-a)又看到了细节描述。

简单而言就是当一个 form 表单中仅有一个 input 元素时，在该输入框内按下回车，会自动触发表单的提交动作，调用 form 的 action 属性。这个自动触发是所有浏览器默认支持的，而当 action 为空时，则默认为当前页面的 URL。

而回头看测试出问题的页面，确实 form 表单没有写 action，并且只有一个输入框（外加一个 \<a\> 标签实现的按钮），所以确定这就是问题的根源：

1) 按下回车时，先触发了表单的默认自动提交（表现为刷新页面），又触发了回车监听事件（表现为正常的 XHR 请求）；

2) 第二次正常的请求又导致第一次刷新请求的连接断开；

3) 后端接口响应第一次请求，返回前端页面文件时才发现已经 over 了，异常抛出；

4) 后端接口响应第二次请求，一切正常，页面正常刷新显示。

---

# 4 2/2真相？

虽说找到了问题，也解决了问题（给 form 表单再加一个隐藏的 input 输入框就可以避免自动提交），但还是有些细节不太清楚：

比如 Wireshark 抓的包中为什么第一个请求断开连接时只有两包？

比如第二个请求发起时为什么第一个请求会关闭？是跟 MVC 有关还是浏览器有关？

算了，先挖坑吧，这东西抠得很细就有点让人头疼，有闲工夫的时候可以再翻出来研究学习一下。
