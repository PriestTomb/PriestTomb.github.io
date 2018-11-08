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

![两个插件的jar包.png](https://i.loli.net/2018/11/07/5be2ef67e7315.png)

#### 2. 解压安装

解压后将两个 jar 包放在 Eclipse 的 dropins 目录下：

![解压.png](https://i.loli.net/2018/11/07/5be2ef6775004.png)

重启 Eclipse 后就可以使用这两个插件了

#### 3. 代码生成

Ctrl + N 选择代码生成器 ↓

![选择客户端生成器.png](https://i.loli.net/2018/11/07/5be2ef6942798.png)

选择从 WSDL 生成代码 ↓

![选择生成java代码.png](https://i.loli.net/2018/11/07/5be2ef675bc64.png)

选择一个 wsdl 文件 或 xml 文件的路径，或者直接输入 WebService 服务的线上地址 ↓

![选择wsdl文件.png](https://i.loli.net/2018/11/07/5be2ef689c892.png)

插件会自动解析，这里简单可以使用默认选项 ↓

![默认.png](https://i.loli.net/2018/11/07/5be2ef6a2801a.png)

选择生成的代码放在哪个路径下，选择项目路径即可 ↓

![选择路径.png](https://i.loli.net/2018/11/07/5be2ef6a1c949.png)

Finish ↓

![成功.png](https://i.loli.net/2018/11/07/5be2ef68082b5.png)

#### 4. 查看生成的代码

![axis2生成的客户端类.png](https://i.loli.net/2018/11/07/5be2ef66e4751.png)
