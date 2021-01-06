---
layout: post
title: 多个Logstash实例在Kibana上只显示一个pipeline
date: 2019-04-27
categories: [小问题处理]
tags: [ELK, Logstash, Kibana]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

接上篇[《多个Logstash实例在Kibana上只显示出一个》](https://priesttomb.github.io/%E5%B0%8F%E9%97%AE%E9%A2%98%E5%A4%84%E7%90%86/2019/04/25/multi-logstash-instance-but-only-show-one-on-kibana/)，使用了不同的 uuid，显示了多个 Logstash Node 之后，发现 Pipelines 这项也是只显示了一个，点进去可以看到 Number of Nodes 又显示了多个

![1.png](/images/blog_img/20190427/1.png)

看到这个名字 `main` 大概想起来，因为目前用的 Logstash 没有配置多 pipelines 共用，所以很多配置都是使用默认的，比如 logstash.yml 文件中的 `pipeline.id`，默认为 `main`

于是对服务器上不同的 Logstash 分别配置了 `pipeline.id`，重启服务后，Kibana 上就正常区分了不同的 pipelines 实例

![2.png](/images/blog_img/20190427/2.png)
