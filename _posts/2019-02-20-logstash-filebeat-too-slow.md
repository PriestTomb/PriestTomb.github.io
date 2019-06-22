---
layout: post
title: Filebeat+Logstash处理大批量数据时卡慢的问题探究
date: 2019-02-20
categories: [问题探究]
tags: [ELK, Logstash, Filebeat]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

# 写在前面

上篇博客[《写了个简单的logstash-output-rocketmq插件》](https://priesttomb.github.io/%E5%B0%8F%E4%BB%A3%E7%A0%81/2019/01/06/logstash-output-rocketmq/)中说到，最近用到了 Filebeat + Logstash 做日志采集，在测试阶段，有一个场景：预先生成一个 1G 大小的日志文件，其中单行数据（即一条日志） 50 kb，让 Filebeat 和 Logstash 去采集，观察性能和机器的负载

测试后，发现一个很奇怪的现象，**每隔 40-50 秒才会处理一批数据**，因为 Logstash 的输出插件是新写的，加了一些测试用的输出代码，所以初步观察后发现，输出插件只在 1-2 秒内就能快速处理完这一批数据，那么问题就来了——**到底是什么原因导致处理数据这么慢？**

花了些时间一步步检查分析了这个问题，这里简单把过程记录下来

---

# 环境说明

Filebeat 6.4

LogStash 6.4

---

# 剥茧抽丝

## 0. 这一批一批处理数据是什么情况

前面测试的几个场景都觉得 Logstash 处理数据很连贯，没有像这次很明显每次处理一批数据，所以先看了下打印的日志，发现每次处理是 2048 条

查了一下，发现这个 2048 条源自 Filebeat 的默认配置 `bulk_max_size`（[官方文档](https://www.elastic.co/guide/en/beats/filebeat/6.4/logstash-output.html#_literal_bulk_max_size_literal_2)），Logstash 每次会请求 2048 条 events。这个值配置的更大，可能会降低频繁的发送带来的开销，但也可能带来 API 出错、连接中断等异常错误，反而降低吞吐量

把这个配置稍微改小一些，比如 1024、512，发现 Logstash 处理会变快，但相邻两次处理一批数据之间依旧有较长的时间“停顿”，所以还要继续深挖

## 1. 排查是不是网络传输的耗时

虽然 Logstash 和 Filebeat 所在的服务器都在内网，按理说 100 MB 的数据不应该存在过大的网络延时，但为了排查，还是用 `tcpdump` 命令抓了两台服务器间的数据包

用 Wireshark 看了之后发现，每隔 40-50 秒确实有一批数据包在传输，但传输的总耗时也就 1-2 秒，这就证明不是网络传输带来的过长耗时

不过同时发现一个小细节，每个数据包都很小，从内容看好像不是完整的一条条 50 kb 大小的日志数据

## 2. 传输的数据为什么那么小

第一反应肯定是数据被压缩了，去查了 Filebeat 的配置，果然发现对数据进行了压缩，涉及配置  `compression_level`（[官方文档](https://www.elastic.co/guide/en/beats/filebeat/6.4/logstash-output.html#_literal_compression_level_literal_2)），默认使用了 gzip 压缩，压缩级别为 3

钻牛角尖一下，是不是压缩解压带来了这个耗时？

把配置改成 0，即不压缩，再次测试并抓包，发现网络传输的耗时有所增加（局域网下不明显），但卡慢现象依旧存在

## 3. 是不是 Logstash 的输入插件太慢

做了两个测试：

* Logstash 改用 file 输入插件，由 Logstash 直接读取日志文件
* Filebeat 部署在 Logstash 所在的服务器

发现 file 插件读取同样规格的日志文件时非常快，另外，即使 Filebeat 和 Logstash 在同一台机器上，卡慢现象依旧存在

简单的几个方面先测试排除问题，那只能直接调试插件代码来定位问题了，先从 ruby 脚本开始

## 4. 调试 logstash-input-beats 插件 ruby 脚本部分

查看 beats 插件的关键类 [`beats`](https://github.com/logstash-plugins/logstash-input-beats/blob/master/lib/logstash/inputs/beats.rb)，看到关键的 `run` 方法：

```ruby
def run(output_queue)
  message_listener = MessageListener.new(output_queue, self)
  @server.setMessageListener(message_listener)
  @server.listen
end
```

启动了一个 beats server，new 了一个 listener 对象，并开始监听

查看 [`MessageListener`](https://github.com/logstash-plugins/logstash-input-beats/blob/master/lib/logstash/inputs/beats/message_listener.rb) 类，找到 `onNewMessage` 方法

```ruby
def onNewMessage(ctx, message)
  hash = message.getData
  ip_address = ip_address(ctx)

  hash['@metadata']['ip_address'] = ip_address unless ip_address.nil? || hash['@metadata'].nil?
  target_field = extract_target_field(hash)

  extract_tls_peer(hash, ctx)

  if target_field.nil?
    event = LogStash::Event.new(hash)
    @nocodec_transformer.transform(event)
    @queue << event
  else
    codec(ctx).accept(CodecCallbackListener.new(target_field,
                                                hash,
                                                message.getIdentityStream(),
                                                @codec_transformer,
                                                @queue))
  end
end
```

简单来说就是 beats server 收到一条消息，会在这个方法中转换成后续 filter 和 output 插件需要的 event 对象

在这个方法中加了一些打印方法，统计各步的耗时，发现每次收到一批消息，只有第一条消息在 `ip_address` 方法的耗时会较长一些，但整体的效率还是非常高

so，问题还不在这里

## 5. 调试 logstash-input-beats 插件 java 部分

最早调试 java 部分其实走了点弯路，我是先从 logstash-input-beats 插件的安装目录下，找到了对应的 jar 包（logstash-input-beats-5.1.6.jar），把这个 jar 包和相关其他依赖（包括 log4j 和 netty 之类的）导入一个 java 工程中，参考上述 ruby 脚本中的写法，也写一个 beats server，开始调试

但 debug 的过程中逐渐遇到问题，因为要对 jar 包里的相关代码进行断点，但**部分类被反编译后，代码和源码还是有差别，导致断点断不到准确的行和方法**，调试起来非常麻烦，后面突然反应过来 logstash-input-beats 是开源的，就直接从 [github 上](https://github.com/logstash-plugins/logstash-input-beats)把代码下下来，把 src 目录中 beats 和 netty 的代码直接 copy 到测试的 java 工程中，继续打断点 / 加测试代码（幸运的是，关键类的源码还没有和 5.1.6 版本的插件有冲突）

在自己写的 beats server 中，对 `onNewMessage` 方法加断点，根据调用栈一步一步往前推，看看 server 这边到底都做了什么，简单整理出了以下流程（部分附上类和方法名，关键的贴了点代码）：

**1. 读取字节到 ByteBuf 对象中**

io.netty.channel.nio.AbstractNioMessageChannel.NioMessageUnsafe.read()

**2. 触发 channelRead**

先后调用到

io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext, Object)

和

io.netty.handler.codec.ByteToMessageDecoder.channelRead(ChannelHandlerContext, Object)

在 `ByteToMessageDecoder` 类中，会累加 ByteBuf 对象，然后调用 Decode 方法

调试到这里时，就已经发现，**卡慢是出在 Decode 环节**

**3. Decode**

先调用到

io.netty.handler.codec.ByteToMessageDecoder.decodeRemovalReentryProtection(ChannelHandlerContext, ByteBuf, List<Object>)

再调用 decode 方法时，会调用到 beats 插件自己的实现

org.logstash.beats.BeatsParser.decode(ChannelHandlerContext, ByteBuf, List<Object>)

该方法中，主要的逻辑集中在这两个 case ：

```java
case READ_COMPRESSED_FRAME: {
    logger.trace("Running: READ_COMPRESSED_FRAME");
    // Use the compressed size as the safe start for the buffer.
    ByteBuf buffer = ctx.alloc().buffer(requiredBytes);
    try (
            ByteBufOutputStream buffOutput = new ByteBufOutputStream(buffer);
            InflaterOutputStream inflater = new InflaterOutputStream(buffOutput, new Inflater())
    ) {
        in.readBytes(inflater, requiredBytes);
        transition(States.READ_HEADER);
        try {
            while (buffer.readableBytes() > 0) {
                decode(ctx, buffer, out);
            }
        } finally {
            buffer.release();
        }
    }

    break;
}
case READ_JSON: {
    logger.trace("Running: READ_JSON");
    ((V2Batch)batch).addMessage(sequence, in, requiredBytes);
    if(batch.isComplete()) {
        if(logger.isTraceEnabled()) {
            logger.trace("Sending batch size: " + this.batch.size() + ", windowSize: " + batch.getBatchSize() + " , seq: " + sequence);
            }
            out.add(batch);
            batchComplete();
        }
        transition(States.READ_HEADER);
        break;
    }
}
```

在 `READ_COMPRESSED_FRAME` 中会循环处理外部接收到的整个 ByteBuf 对象，每次解码其中一条消息，而每次循环，最终都将走到 `READ_JSON` 逻辑中去，这里是将每条消息添加到 beats 实现的 `V2Batch` 对象

**4. addMessage**

点开 `V2Batch` 类的 `addMessage` 方法，大概就猜到了“问题”出在哪里

```java
void addMessage(int sequenceNumber, ByteBuf buffer, int size) {
    written++;
    if (internalBuffer.writableBytes() < size + (2 * SIZE_OF_INT)){
        internalBuffer.capacity(internalBuffer.capacity() + size + (2 * SIZE_OF_INT));
    }
    internalBuffer.writeInt(sequenceNumber);
    internalBuffer.writeInt(size);
    buffer.readBytes(internalBuffer, size);
}
```

从代码里可以看到 `V2Batch` 类中维护了一个 `internalBuffer`，而每次要往 `internalBuffer` 中写数据时，都要判断一下 buffer 的可写容量，如果小于本次要读取的长度(+8)，则......扩容这个长度(+8)

加了点输出代码，看了下 `capacity` 这步的耗时，最终证实，就是**一次次的扩容带来的耗时递增**，从最初的每次扩容 0 ms  逐步增长到后期的每次 30 ms 、 40 ms ，而这个循环，每收到一批数据就要执行 2048 次。。

但我为什么会给问题二字加引号呢，因为这个方法的注释是这么写的：

```
Adds a message to the batch, which will be constructed into an actual Message lazily.
```

最后一个词，lazily。。

作者出于什么目的，每次只扩容所需长度，而不是容量翻倍增长，看了作者提交记录和 github 上的 issue [Too many alloc/memcpy in V2Batch](https://github.com/logstash-plugins/logstash-input-beats/issues/338) ，都没看到有相关的解释，但既然作者定义这个方法就是 lazily，那暂且这样吧

## 6. 优化

能造成这种明显卡慢的条件是 **Logstash 单批次收到的 events 过多**，并且**每个 event 体积偏大**

虽然这只是测试场景中比较极端的一个，基本不会发生，但也不得不考虑一旦出现，这么慢的处理效率会不会有不好的影响，所以为了避免以后由于这种卡慢出问题，还是先对这个方法做一个简单的优化

```
while (internalBuffer.writableBytes() < size + (2 * SIZE_OF_INT)){
	internalBuffer.capacity(internalBuffer.capacity() * 2);
}
```

每次直接两倍扩容，ByteBuf 的默认最大容量好像是 2147483647 ，目测项目的实际应用中，不可能出现一次收集这么大量的数据。。

优化后再次测试，2048 条日志只需要扩容十几次，耗时骤降

---

# 最后

发现这个问题的时候因为还有其他的任务要做，就暂时没投入过多的时间研究，先放在一边了，抽空去论坛提了个帖子，但过了十几天也没人回复。。【论坛的活跃度好像还是偏低了点

最后还是自己花时间一点点调试找到了根源，整个过程很简单，不过还挺好玩的。。不知道 beats 插件的作者后续会不会有相关的优化或者解释，等官方有更新的话，再回来更新下文章
