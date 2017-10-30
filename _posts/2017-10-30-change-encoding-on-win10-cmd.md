---
layout: post
title: Win10控制台乱码问题
date: 2017-10-30
categories:
- 系统环境
tags: [windows]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

之前编译 WebService 客户端代码时发现 Win10 的控制台有乱码，平时习惯 Win + R 输 cmd 的方式打开默认的 cmd 控制台，发现乱码后又看了眼管理员模式，发现不是乱码：

![发现乱码](http://oxujjb0ls.bkt.clouddn.com/image/win10%E6%8E%A7%E5%88%B6%E5%8F%B0%E7%BC%96%E7%A0%81/%E4%B9%B1%E7%A0%81.png)

用 chcp 命令查了下，发现是两个控制台的编码不同，一个是 utf8，一个是 gbk：

![chcp查看编码](http://oxujjb0ls.bkt.clouddn.com/image/win10%E6%8E%A7%E5%88%B6%E5%8F%B0%E7%BC%96%E7%A0%81/chcp%E5%91%BD%E4%BB%A4%E6%9F%A5%E7%9C%8B.png)

默认控制台中执行命令：

```
chcp 936
```

回车后会刷屏，显示：

    活动代码页：936

再执行之前的命令发现乱码正常了：

![检查是否正常](http://oxujjb0ls.bkt.clouddn.com/image/win10%E6%8E%A7%E5%88%B6%E5%8F%B0%E7%BC%96%E7%A0%81/%E6%94%B9%E7%BC%96%E7%A0%81%E5%86%8D%E6%B5%8B%E8%AF%95.png)

---

后续发现把控制台关掉重开，编码又会恢复65001，Win10 下没什么好办法，有一种是改注册表：

Win + R 输入 regedit，打开注册表，找到：

计算机\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Command Processor

新建字符串值 autorun，值为'chcp 936'

![新建autorun](http://oxujjb0ls.bkt.clouddn.com/image/win10%E6%8E%A7%E5%88%B6%E5%8F%B0%E7%BC%96%E7%A0%81/%E6%96%B0%E5%BB%BAautorun.png)

这样再打开控制台，就会默认先执行该命令：

![默认先执行命令](http://oxujjb0ls.bkt.clouddn.com/image/win10%E6%8E%A7%E5%88%B6%E5%8F%B0%E7%BC%96%E7%A0%81/%E9%BB%98%E8%AE%A4%E5%85%88%E6%89%A7%E8%A1%8C%E4%BA%86chcp%20936.png)
