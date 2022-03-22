---
layout: post
title: Logstash 插件(JRuby)-引入 jar 包并使用
date: 2022-03-22
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

# 2 引入 jar 包并使用

正因为 Logstash 插件使用的是 JRuby，所以可以引入一些方便的 jar 包配合使用。比如 logstash-output-jdbc 插件的实现，就是引入和加载 JDBC jar 包，就可以使用了。

在写过的一些插件中，用到了如 RocketMQ 客户端的功能，于是参考着这样实现了。

### 2.1 引入 RocketMQ jar 包

我的习惯是直接给插件加一个配置参数`logstash_path`，指引使用者自行配置其根路径，以找到存放外部 jar 包的路径。

```ruby
def load_jar_files
  jarpath = logstash_path + "/vendor/jar/rocketmq/*.jar"
  @logger.info("RocketMq plugin required jar files are loadding... Jar files path: ", path: jarpath)

  jars = Dir[jarpath]
  raise LogStash::ConfigurationError, 'RocketMq plugin init error, no jars found! Please check the jar files path!' if jars.empty?

  jars.each do |jar|
    @logger.trace('RocketMq plugin loaded a jar: ', jar: jar)
    require jar
  end
end
```

### 2.2 使用 Java 类

在上述例子中，引入 RocketMQ 的客户端 jar 包后，就可以按 Java 中的使用方式，使用相应的类了，当然，语法还是 Ruby 的。

```ruby
@producer = org.apache.rocketmq.client.producer.DefaultMQProducer.new(producer_group)
@producer.setNamesrvAddr(name_server_addr)
@producer.setVipChannelEnabled(use_vip_channel)
@producer.start
mq_message = org.apache.rocketmq.common.message.Message.new
mq_message.setTopic(topic)
mq_message.setTags(tag)
mq_message.setKeys(key)
java_body = java.lang.String.new(body)
mq_message.setBody(java_body.getBytes(org.apache.rocketmq.remoting.common.RemotingHelper::DEFAULT_CHARSET))
@producer.send(mq_message)
```
