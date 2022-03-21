---
layout: post
title: Logstash 插件(JRuby)-启动线程
date: 2022-03-21
categories:
- 技术
tags: [ELK, Logstash, Ruby]
status: publish
type: post
published: true
---

# 1 写在前面

在写 Logstash 插件的过程中，借鉴和学习到一些 JRuby 的书写技巧，随手记录一下，仅供参考。

---

# 2 启动线程

在 Logstash 插件中启动线程的目的大概有两种：一是需要多个线程，并行处理接收到的一批 Event，进一步提高效率；另一个则是在主线程运行过程中，由一个子线程定时处理一些事情。

写过的一些插件中，目前用到的都是第二种操作。

### 2.1 启动 Ruby 线程

```ruby
Thread.new(@interval) do |interval|
  while @stopping.false? do
    sleep(interval)
    do_something()
  end
end
```

### 2.2 启动 Java 线程

```ruby
java.lang.Thread.new {
  while @stopping.false? do
    sleep(@interval)
    do_something()
  end
}.start
```
