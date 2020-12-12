---
layout: post
title: LogIQ —— 一个 Logstash 监控数据展示页面
date: 2020-12-10
categories:
- 技术
tags: [Logstash]
status: publish
type: post
published: true
---

# 如何监控 Logstash

网上目前关于 Logstash 监控大体有两类文章，一是介绍官方的 X-Pack 监控，二是介绍 Logstash 自己的监控 API 。

第一种，[X-Pack](https://www.elastic.co/guide/en/kibana/current/xpack-monitoring.html) 是官方的插件，配合 ElasticSearch 和 Kibana 能采集展示 ELK 全家桶里各种组件的运行数据，其问题也就在于需要依赖 ElasticSearch 和 Kibana ，如果刚好有部署了全套的 ELK ，这种方案当然最方便，但如果只是使用 Logstash ，实在是没必要为了监控再搭建一个 ES 和 Kibana 。

第二种，Logstash 自身有提供一些[监控 API](https://www.elastic.co/guide/en/logstash/current/monitoring-logstash.html) ，可通过 HTTP 请求查询 Logstash 节点的信息、节点状态、插件信息等，返回的查询结果大多为 JSON 格式，这种方案无疑是只使用 Logstash 服务时的最佳选择，唯一的问题就是，官方没有提供一个能更直观查看各项数据的 UI 页面。

这篇博客就不再详细介绍这两种方案了，官方文档和网上的文章已经很多了，下面就来安利一个利用监控 API 做的展示页面。

---

# LogIQ

前几年刚学习 Logstash 的时候，考虑到监控的问题，也是先了解到了官方提供的这些监控 API ，随后随手一搜，搜到了一个开源的展示页面—— [LogIQ](https://github.com/sloniki/logiq) 。

试用过后发现还不错，主要是源码也挺简单，还好用的还是上古的 jQuery（雾），随后也参与提交了一些内容，先贴个示例图：

![示例图](/images/blog_img/20201210/example.png)

---

# 数据展示

### 0. Events Stats

主要展示输入、过滤器、输出插件分别处理的事件数量，以及处理事件的总耗时、事件推送被阻塞的等待总耗时（可侧面反映出 Logstash 的处理压力）。

### 1. JVM

主要展示 JVM 相关的环境信息。

### 2. Process Stats

主要展示进程相关的环境信息。

### 3. Pipelines Stats

分别展示了管道、输入插件、过滤器、输出插件的基本运行数据，还有一个关于执行耗时的饼状图。

这部分应该是大多数使用者比较关心的实时数据，各个插件的输入输出事件数量，输入插件被阻塞的耗时，输出插件的执行耗时。

单纯的数字往往也会失去意义，能够以图表的形式展现才能更加直观，现在的版本中先实现了一个代表各过滤器、输出插件的执行耗时的饼状图，可以让使用者直接比较各个执行插件的执行效率。

### 4. General Info

Logstash 服务及所在服务器的一般信息。

### 5. Hot Threads

Logstash 内部各主要线程的关键信息。

### 6. Plugins Info

当前 Logstash 已经安装的所有插件的列表。

---

# 使用说明

纯粹一个 html 页面，好像本身也没太多需要特殊说明的注意事项，仅有的常见问题是来自那些 API 接口。

首先需要开启这些接口，在 logstash.yml 中找到 `http.host` 配置地址，本机环回地址也行，内网 IP 也行，只有配了才能调用。

其次访问接口一般会有跨域问题存在，所以需要特别处理一下，像项目的 README 中所说，使用 Nginx 做一个代理即可。

```
server {
    listen       9601;
    location / {
      if ($request_method = 'GET') {
      add_header 'Access-Control-Allow-Origin' '*';
      add_header 'Access-Control-Allow-Methods' 'GET';
      add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range';
      add_header 'Access-Control-Expose-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range';
      }
      proxy_pass http://logstash:9600;
    }
}
```

---

# 最后

这个项目的应用场景正如文章开头所说，适合那些只部署使用 Logstash ，没有多余的 ES + Kibana 的环境。借助官方提供的 API 就能获取很多关键的信息，用合适的图表展示出来即可，所以后续还会添加其他丰富的图表来提高页面的实用性（挖坑，待填）。
