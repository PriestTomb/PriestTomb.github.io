---
layout: post
title: Eclipse通过Maven引入jar包但类中依旧找不到的问题
date: 2017-11-26
categories:
- 编程工具
tags: [Eclipse, Maven]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

今天用 Maven 做小项目时遇到个非常奇怪的问题：

**pom.xml 中配置了 hibernate，检查项目的 Libraries -> Maven Dependencies，jar 包已经下好了，也在了，但在编写代码的时候，好多类都提示找不到，比如 org.hibernate.SessionFactory，引入就报错**

尝试了修改 pom.xml 中配置的版本，依旧报错

尝试了关闭项目，重新打开，依旧报错

尝试了新建个项目，使用了同样的 pom.xml 配置，依旧报错

尝试到最后，意识到可能不是配置的问题，跑去把 Maven 的本地仓库下 hibernate 相关的目录全删掉，在 Eclipse 中重新保存 pom.xml 让它重新下载，最后终于不再报错！
