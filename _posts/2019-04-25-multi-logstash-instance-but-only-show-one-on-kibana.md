---
layout: post
title: 多个Logstash实例在Kibana上只显示出一个
date: 2019-04-25
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

# 一个小问题

安装 ElasticSearch 和 Kibana 后，通过[配置 Logstash 的 x-pack](https://www.elastic.co/guide/en/logstash/current/configuring-logstash.html) 实现可视化监控是非常便捷的监控手段，ELK 全家桶中的 Filebeat 也是一样

但在使用过程中发现，明明配置和启动了多个 Logstash 实例，但在 Kibana 的监控页面只能看到 1 个 Node，点进去之后却发现这个 Node 所表示的 Logstash 实例还会变化（根据 ip 可以看出）

研究了一下才明白，是这几台服务器上启动的 Logstash 使用的 uuid 是同一个导致的，在 Logstash 的根目录下的 data 目录中，会发现有个叫 uuid 的文件，这个文件标识了当前启动的 Logstash 实例

因为当初安装 Logstash 时，是先在其中一台服务器上安装调配好，然后打包发到其他几台服务器上解压出来使用的，这就导致了这几个 Logstash 全部使用了相同的 uuid，所以在 Kibana 中就只识别出了一个节点。虽然不影响 Logstash 的正常使用，但在借助 Kibana 实现监控时就出了这么个小问题

解决方法就是**关掉 Logstash 服务，删除该 uuid 文件，再启动 Logstash 服务**，就会新生成一个 uuid 了，Kibana 中就可以看到不同的 Logstash 实例了

---

# 参考

[Kibana monitoring shows only 1 Logstash instance](https://discuss.elastic.co/t/kibana-monitoring-shows-only-1-logstash-instance/137564/4?u=priesttomb)
