---
layout: post
title: 写了个简单的logstash-output-rocketmq插件
date: 2019-01-06
categories:
- 小代码
tags: [LogStash, Rocketmq]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

# 写在前面

近期参与一个新的项目，姑且是负责了数据存储相关模块的需求分析和设计（其实就是参考学习网络上流传的其他大厂分享的成熟方案），目前的方案是用 FileBeat + LogStash 做收集，之后再存到 HDFS、HBase、MySQL

本来是看到 LogStash 有现成的直接输出到 HDFS 的插件（[logstash-output-webhdfs](https://github.com/logstash-plugins/logstash-output-webhdfs)），就计划先把数据落到 HDFS，后续通过定时程序从 HDFS 取数据再分别存到 HBase 和 MySQL 去来着，不过被领导否了，说从 LogStash 一步到位，同时把数据扔到 HDFS、HBase、MySQL 去，那这样的话就得来一个消息队列在中间，这样一方面也方便我们自己对数据做一个把控，毕竟数据还要分别存到三个地方去，另一方面也方便以后做其他扩展，三下五除二就敲定了要用 Rocketmq（之前的项目中有使用，所以不想再多费精力用其他的消息队列）

然而网上搜了很久，LogStash 的官方输出插件里没有 Rocketmq，连个民间第三方的都没搜到，所以不得不想办法自己来写一个了。。

花了两天写了个超级简陋的版本，现在丢 github 上去了：[logstash-output-rocketmq](https://github.com/PriestTomb/logstash-output-rocketmq)，还煞有其事地准备了英文的 README （其实就是机翻）

---

# 环境说明

Ruby 2.6

LogStash 6.4

Rocketmq 4.2

---

# 自己动手

## 0. 学 Ruby

因为 LogStash 是用 Ruby 写的，又恰好不会这个语言，索性借此机会学习一下

看官方文档和菜鸟教程学了几天，把基本的语法概念了解了个大概，于是准备开始第二步

## 1. 学源码

先前在做方案设计的时候在网上找到了 LogStash 的一个第三方插件 [logstash-output-jdbc](https://github.com/theangryangel/logstash-output-jdbc)，当时想着能不能像 LogStash 直接输出 HDFS 那样，也直接输出到数据库去，就找到了这个。在自己写插件的过程中，参考了这个插件的源码居多（核心思路来自于此）

另外也看了官方的另外两个插件 [logstash-output-kafka](https://github.com/logstash-plugins/logstash-output-kafka) 和 [logstash-output-rabbitmq](https://github.com/logstash-plugins/logstash-output-rabbitmq)

偷偷吐槽一句，Ruby 的代码风格可真"飘逸"。。

## 2. 写插件

先介绍思路吧，因为 Rocketmq 没有 JRuby 实现的客户端（[logstash-output-rabbitmq](https://github.com/logstash-plugins/logstash-output-rabbitmq) 就是借助了 JRuby 实现的客户端来做的），所以这里使用了官方的客户端 jar 包，将使用客户端所需的关键 jar 包放到某个目录下，在初始化插件时加载，就可以在插件中直接调用 jar 包中的类和方法实现生产者的初始化和消息发送了

#### 2.1. 准备 example

参考官方文档 [How to write a Logstash output plugin](https://www.elastic.co/guide/en/logstash/current/_how_to_write_a_logstash_output_plugin.html)，下载 [logstash-output-example](https://github.com/logstash-plugins/logstash-output-example/)

打开看一下会发现还是有很多乱七八糟的文件的，最关键的几个文件是：

* lib/logstash/outputs/example.rb 插件的核心代码文件，主要用于定义插件名称、向 LogStash 注册插件、接收事件(并处理)等

* spec/outputs/example_spec.rb 插件的测试文件，使用 RSpec 语法，主要是用于功能测试

* logstash-output-example.gemspec 插件的说明文件，描述插件的基本信息、配置插件运行依赖的文件等，在看其他插件源码时甚至看到能在这里引入 Maven，不明觉厉

将一些 `example` 都修改成想实现的插件的名字就可以了，比如我这里就是叫 `rocketmq`

#### 2.2. 写核心文件

因为整个脚本的内容不多，我这里就直接附代码了，在代码注释中进行详细的介绍说明

```ruby
# encoding: utf-8
require "logstash/outputs/base"
require "logstash/namespace"
require "java"

# 自己写的插件类需要继承对应的 Base 类
class LogStash::Outputs::Rocketmq < LogStash::Outputs::Base

  # 设置插件可多线程并发执行
  # 默认为 single，即使配置多个 workers，也只能依次运行 multi_receive 方法（性能上来说变成单线程）
  concurrency :shared

  config_name "rocketmq"

  # 使用 config 定义插件运行所需要的基本参数
  # 本地 Logstash 的路径，必需，如 C:/ELK/logstash、/usr/local/logstash
  config :logstash_path, :validate => :string, :required => true

  # Rocketmq 的 NameServer 地址，必需，如 192.168.10.10:5678
  config :name_server_addr, :validate => :string, :required => true

  # Rocketmq 的 producer group
  config :producer_group, :validate => :string, :default => "defaultProducerGroup"

  # Message 的 topic，必需
  config :topic, :validate => :string, :required => true

  # Message 的 tag
  config :tag, :validate => :string, :default => "defaultTag"

  # Message 的 key
  config :key, :validate => :string, :default => "defaultKey"

  # 发送异常后的重试次数，默认 2 次
  config :retry_times, :validate => :number, :default => 2

  # 注册方法，在 LogStash 启动时会调用，相当于对插件进行初始化
  def register
    load_jar_files

    @stopping = Concurrent::AtomicBoolean.new(false)

    # 创建生产者对象
    @producer = org.apache.rocketmq.client.producer.DefaultMQProducer.new(producer_group)
    @producer.setNamesrvAddr(name_server_addr)
    @producer.start
  end

  # 接收事件方法，接收 events 对象，在其他插件里还有看到 multi_receive_encoded 方法
  # 入参是 events_and_data 对象，这两个方法有什么不同我反正是不懂
  def multi_receive(events)
    return if events.empty?
    events.each do |event|
      retrying_send(event)
    end
  end

  # 到我们指定的目录下加载依赖的 jar 文件
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

  # 生产者客户端核心功能方法
  def retrying_send(event)
    sent_times = 0

    begin
      # 配置 message 对象
      mq_message = org.apache.rocketmq.common.message.Message.new
      mq_message.setTopic(topic)
      mq_message.setTags(tag)
      mq_message.setKeys(key)
      mq_message.setBody("#{event}".bytes)
      result = @producer.send(mq_message)

      if result.nil?
        raise "Send message error! Result is null."
      end

      if org.apache.rocketmq.client.producer.SendStatus::SEND_OK != result.getSendStatus
        status_name = result.getSendStatus.name
        raise "Send message error! Result code is #{status_name}"
      end
    rescue => e
      @logger.error('An Exception Occured!!',
                    :message => e.message,
                    :exception => e.class)
      if @stopping.false? and (sent_times < retry_times)
        # 重试
        sent_times += 1
        retry
      else
        # 根据实际需求处理没发送成功的消息
        puts "Message send failed: #{event}"
      end
    end
  end

  # LogStash 停止时会调用该方法
  def close
    @stopping.make_true
    @producer.shutdown
  end
end
```

## 3. 打包安装测试

一方面懒得学 RSpec 写测试代码，一方面没用过 LogStash，不知道影响插件运行的最关键的 `multi_receive` 方法的入参 events 到底是什么内容，所以写了点测试输出的代码，直接打包安装进行测试

为了方便，我在自己本机上安装了 Filebeat 和 LogStash（幸亏这两个软件对 windows 都有不错的支持），然后用命令 `gem build logstash-output-rocketmq.gemspec` 对插件进行打包，得到 gem 文件后，再到 LogStash 的目录下用命令 `bin/logstash-plugin install logstash-output-rocketmq-0.1.0.gem` 进行安装

安装成功后对 FileBeat 和 LogStash 进行简单的配置，测试了一些日志文件，发现插件正常可用，于是暂时收工

## 4. 注意

真实的安装环境可能会没有网络，需要离线安装插件，所以要用其他命令

打包

`bin/logstash-plugin prepare-offline-pack logstash-output-rocketmq`

安装

`bin/logstash-plugin install file:///path/to/logstash-offline-plugins-6.4.0.zip`

---

# 最后

在 github 上也有说，这个版本写出来只能称得上是个 demo，毕竟也没考虑其他 LogStash 版本和 Rocketmq 版本，写出来目前可能够我的工作需要（后期按需再调整优化），因为没搜到网上有关于 Rocketmq 版的 LogStash output 插件，所以发出来做个分享

可能实际写得很烂哈哈，还请见谅，毕竟这几个东西我之前都还没用过。。如果有更好的思路或者代码有写错的地方，请发 issue 或者直接评论告诉我，thx
