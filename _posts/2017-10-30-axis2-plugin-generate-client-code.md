---
layout: post
title: Eclipse安装Axis2插件并生成客户端代码
date: 2017-10-30
categories:
- 后端技术
tags: [WebService]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

#### 1. 下载 jar 包

从[官网](http://axis.apache.org/axis2/java/core/download.html)下载最新版本的 plugin for Eclipse(当前最新版本是1.7.6)

![插件安装包](http://oxujjb0ls.bkt.clouddn.com/image/axis2%E6%8F%92%E4%BB%B6%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BB%A3%E7%A0%81/%E4%B8%A4%E4%B8%AA%E6%8F%92%E4%BB%B6%E7%9A%84jar%E5%8C%85.png)

#### 2. 解压安装

解压后将两个 jar 包放在 Eclipse 的 dropins 目录下：

![解压安装](http://oxujjb0ls.bkt.clouddn.com/image/axis2%E6%8F%92%E4%BB%B6%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BB%A3%E7%A0%81/%E8%A7%A3%E5%8E%8B.png)

重启 Eclipse 后就可以使用这两个插件了

#### 3. 代码生成

Ctrl + N 选择代码生成器 ↓

![选择代码生成器](http://oxujjb0ls.bkt.clouddn.com/image/axis2%E6%8F%92%E4%BB%B6%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BB%A3%E7%A0%81/%E9%80%89%E6%8B%A9%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%94%9F%E6%88%90%E5%99%A8.png)

选择从 WSDL 生成代码 ↓

![选择生成代码](http://oxujjb0ls.bkt.clouddn.com/image/axis2%E6%8F%92%E4%BB%B6%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BB%A3%E7%A0%81/%E9%80%89%E6%8B%A9%E7%94%9F%E6%88%90java%E4%BB%A3%E7%A0%81.png)

选择一个 wsdl 文件 或 xml 文件的路径，或者直接输入 WebService 服务的线上地址 ↓

![选择文件](http://oxujjb0ls.bkt.clouddn.com/image/axis2%E6%8F%92%E4%BB%B6%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BB%A3%E7%A0%81/%E9%80%89%E6%8B%A9wsdl%E6%96%87%E4%BB%B6.png)

插件会自动解析，这里简单可以使用默认选项 ↓

![默认](http://oxujjb0ls.bkt.clouddn.com/image/axis2%E6%8F%92%E4%BB%B6%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BB%A3%E7%A0%81/%E9%BB%98%E8%AE%A4.png)

选择生成的代码放在哪个路径下，选择项目路径即可 ↓

![选择路径](http://oxujjb0ls.bkt.clouddn.com/image/axis2%E6%8F%92%E4%BB%B6%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BB%A3%E7%A0%81/%E9%80%89%E6%8B%A9%E8%B7%AF%E5%BE%84.png)

Finish ↓

![finish](http://oxujjb0ls.bkt.clouddn.com/image/axis2%E6%8F%92%E4%BB%B6%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BB%A3%E7%A0%81/%E6%88%90%E5%8A%9F.png)

#### 4. 查看生成的代码

![axis2插件生成的代码](http://oxujjb0ls.bkt.clouddn.com/image/axis2%E6%8F%92%E4%BB%B6%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BB%A3%E7%A0%81/axis2%E7%94%9F%E6%88%90%E7%9A%84%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%B1%BB.png)
