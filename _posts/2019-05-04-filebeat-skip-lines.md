---
layout: post
title: Filebeat采集数据不完整、一行数据被截断（inode重用问题）
date: 2019-05-04
categories: [技术]
tags: [ELK, Filebeat]
status: publish
type: post
published: true
---

# 写在前面

用 Filebeat + Logstash 采集日志文件数据时，偶然发现了采集的数据出现了不完整的情况，因为在 Logstash 中配置了 json 过滤器，所以遇到这种不完整的数据会报出一条解析失败的日志

经过一点点排查，最后断定是 Filebeat 采集的时候在某种外部特殊操作的影响下，会出现这种情况，有一行完整的数据被从中间截断，在同事的协助下，在网上看到了[一篇博客](https://www.cnblogs.com/blogjun/articles/8108440.html)，在文章的最后描述了一个 inode 重用的问题

> 在 Linux 文件系统上，Filebeat 使用 inode 和设备来识别文件。从磁盘中删除文件时，可将 inode 分配给新文件。在涉及文件旋转的使用情况下，如果旧文件被删除并且之后立即创建新文件，则新文件可能与删除的文件具有完全相同的 inode。在这种情况下，Filebeat 假定新文件与旧文件相同，并尝试在旧位置继续读取，这是不正确的。

做了一些测试、收集了相关的数据之后，确定了就是这个问题导致的

---

# 问题场景描述

因为**磁盘容量有限** + **日志文件比较大**，所以在长时间测试 Filebeat + Logstash 采集日志数据时，使用了一个 shell 脚本定时扫描目录下的日志文件，删除其中被采集过的旧文件

且 Filebeat 的 input 配置基本使用了默认配置，即只指定了采集数据的目录

在正常采集的过程中，如果 shell 脚本删除了一些旧的日志文件，则会有一定概率出现采集到的数据中有不完整的现象

---

# 验证 inode 重用

### 测试方法

启动测试程序，开始生成日志文件（按计划每 200MB 一个文件，文件名带有一个递增编号），Filebeat 采集几分钟之后，复制保存一份 data 目录下的 registry 文件（记录每个采集文件信息的注册信息文件），接着按日志文件的生成时间从最早的一个依次开始删除，观察采集的数据是否出现了被截断的现象，如果出现，就停止测试，查看最新的 registry 文件（和 Logstash 中报错的日志），与先前的对比

### 测试结果

Logstash 报错日志：

```json
{
  "source": "/path/to/testlog.407.log",
  "tags": [
    "beats_input_codec_plain_applied"
  ],
  "prospector": {
    "type": "log"
  },
  "host": {
    "name": "localhost"
  },
  "@version": "1",
  "message": "省略很长很长很长的被截断的日志数据",
  "@timestamp": "2019-04-25T07:31:38.905Z",
  "json": {
    "error": {
      "message": "Error decoding JSON: json: cannot unmarshal number into Go value of type map[string]interface {}",
      "type": "json"
    }
  },
  "beat": {
    "hostname": "localhost",
    "name": "localhost",
    "version": "6.4.0"
  },
  "input": {
    "type": "log"
  },
  "offset": 213068291
}
```

可以看出这条被截断的异常数据属于 testlog.407.log 这个文件，且这条数据被采集时的 offset 是 **213068291**

接着查看最新的 registry 文件，找到 testlog.407.log 的采集信息：

```json
{
  "source": "/path/to/testlog.407.log",
  "offset": 215862613,
  "timestamp": "2019-04-25T15:31:46.531617922+08:00",
  "ttl": -1,
  "type": "log",
  "meta": null,
  "FileStateOS": {
    "inode": 9961475,
    "device": 64528
  }
}
```

可以看出这个文件被分配的 inode 是 **9961475**，而且这个文件的大小，即 offset 为 **215862613**

最后再看之前保存的旧 registry 文件，找到 inode 等于 9961475 的数据：

```json
{
  "source": "/path/to/testlog.358.log",
  "offset": 213068291,
  "timestamp": "2019-04-25T15:29:06.76424744+08:00",
  "ttl": -1,
  "type": "log",
  "meta": null,
  "FileStateOS": {
    "inode": 9961475,
    "device": 64528
  }
}
```

可以看出 **9961475 这个 inode 确实被重用了**，之前分配给了编号 358 这个文件，在删除 358 之后，分配给了编号 407 这个文件

而且重点是，**新的 407 的大小 215862613 比旧的 358 的 213068291 要大**，所以在 Logstash 的错误日志中，报错的这条数据就是从 407 这个文件的 **213068291** 位置开始截取的，这就符合了文章开头中描述的 ***Filebeat 假定新文件与旧文件相同，并尝试在旧位置继续读取***

---

# 解决问题

既然验证了这个看似奇葩的问题是由 inode 重用导致，那就可以按官方的推荐（在查看官方的配置说明时，刚好找到了官方对这个问题的说明：[Inode reuse causes Filebeat to skip lines?](https://www.elastic.co/guide/en/beats/filebeat/current/inode-reuse-issue.html)），使用 `clean_inactive` 和 `clean_removed` 这两个配置来解决这个问题

给这两个配置指定了时间，就代表在这个时间之后，Filebeat 将从 registry 文件中自动删除过期的文件注册信息，这样就能避免以上问题
