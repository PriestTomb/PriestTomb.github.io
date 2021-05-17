---
layout: post
title: 从源码学习 Logstash 内存式队列的运行原理
date: 2021-01-03
categories:
- 技术
tags: [Logstash]
status: publish
type: post
published: true
---

# duration\_in\_millis 是什么

在上篇博客[《LogIQ —— 一个 Logstash 监控数据展示页面》](https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2020/12/10/monitoring-web-ui-for-logstash-logiq/)中关于管道状态数据展示中有提到执行耗时，在 Logstash 的监控 API 中能拿到的数据就是这个 `duration_in_millis` ，单位是毫秒，更确切的说，这个数据指的是过滤器和输出插件的执行耗时。对于输入插件而言，是另一个 `queue_push_duration_in_millis` 。

关于为什么要研究这两个数据，起因主要是在对 Logstash 做监控时，不只是需要看 Logstash 处理了多少数据，还想关注 Logstash 处理的快不快，如果慢，是哪里慢，因为一般我们会使用不止一个过滤器插件甚至输出插件。

比如之前项目中有一个应用场景，配置了五六个过滤器，输出时使用了一个第三方插件 [logstash-output-jdbc]() 。使用者发现日志采集比较慢，反馈到我这里时，下意识觉得是不是过滤器配置太多了，排查后发现其中一个过滤器确实比其他的慢很多，但更多的是因为 jdbc 插件默认不支持批量入库，一条一条入库当然会慢。考虑到过滤器是业务层面必须的配置，最后是魔改了 jdbc 插件，使其支持批量入库，将入库效率提升了近 4 倍，相比之下，一个过滤器的“缓慢”已经可以忽略不计。

比较插件的耗时问题，主要就是观察监控 API 中的 `duration_in_millis` ，最早接触监控 API 时，对这些参数不是很了解，怕对字面意思的理解和实际有偏差，虽然当时在官方论坛中找到了[官方人员对参数的解释](https://discuss.elastic.co/t/-node-stats-duration-in-millis-vs-queue-push-duration-in-millis/90000)，因为一会儿说读客户端写客户端，一会儿说持久化队列内存式队列，让一个新手看得满脸问号，最后还是翻看了源码来验证自己的猜想和理解。

下面的解析部分以 Logstash 的**内存式队列**为例，会涉及 **Logstash 启动、各插件间的数据传递、监控 API 中的个别性能指标**。

---

# 版本说明

Logstash 6.4.0

---

# 太长不看

* Logstash 启动时根据配置的队列类型进行初始化，内存式队列将根据 `pipeline.batch.size` 和 `pipeline.workers` 初始化一个有界阻塞队列（ArrayBlockingQueue）
* 利用阻塞队列，初始化一个写客户端和读客户端，实现一个生产者+消费者模型
* 写客户端即输入插件的核心，输入插件接收数据，转化为 Logstash 内的 event 对象，生产资源
* 读客户端即过滤器和输出插件的核心，消费资源，交由过滤器和输出插件依次执行
* `queue_push_duration_in_millis` 是写客户端生产资源时被阻塞的时间，该值越大，说明消费者性能越低
* `duration_in_millis` 是读客户端获取资源后，所有过滤器和输出插件执行业务处理的时间

---

# 相关源码解析

### 0. pipeline.rb

该脚本即为 Logstash 启动环节的关键，从代码中可以看到 pipeline 的初始化和启动，以及启动、运行各工作线程（包括输入、过滤、输出插件）的关键方法。

这里先附上 pipeline 初始化和工作线程运行的部分：

```ruby
def initialize(pipeline_config, namespaced_metric = nil, agent = nil)
  super
  # 初始化队列
  open_queue

  # 一些实例变量的初始化
  @worker_threads = []
  @signal_queue = java.util.concurrent.LinkedBlockingQueue.new
  ...
end

def worker_loop(batch_size, batch_delay)
  filter_queue_client.set_batch_dimensions(batch_size, batch_delay)
  output_events_map = Hash.new { |h, k| h[k] = [] }
  while true
    signal = @signal_queue.poll || NO_SIGNAL
    # 从队列中读取一批数据，并启动监测
    batch = filter_queue_client.read_batch.to_java
    batch_size = batch.filteredSize
    if batch_size > 0
      @events_consumed.add(batch_size)
      # 依次执行各过滤器插件
      filter_batch(batch)
    end
    # 刷新过滤器，将处理完的 event 传递给输出插件
    flush_filters_to_batch(batch, :final => false) if signal.flush?
    if batch.filteredSize > 0
      # 依次执行各输出插件
      output_batch(batch, output_events_map)
      filter_queue_client.close_batch(batch)
    end
    # keep break at end of loop, after the read_batch operation, some pipeline specs rely on this "final read_batch" before shutdown.
    break if (@worker_shutdown.get && !draining_queue?)
  end
  # 收到停止信号后的 pipeline 排空操作
  ...
end
```

### 1. open_queue

上面 pipeline.rb 中可以看到调用到了 `open_queue` 方法进行队列的初始化，该方法的方法体是 Java 代码，定义在 AbstractPipelineExt.openQueue(ThreadContext)

```java
private AbstractWrappedQueueExt queue;
private JRubyAbstractQueueWriteClientExt inputQueueClient;
private QueueReadClientBase filterQueueClient;

@JRubyMethod(name = "open_queue")
public final IRubyObject openQueue(final ThreadContext context) throws IOException {
    try {
        queue = QueueFactoryExt.create(context, null, settings);
    } catch (final Exception ex) {
        LOGGER.error("Logstash failed to create queue.", ex);
        throw new IllegalStateException(ex);
    }
    inputQueueClient = queue.writeClient(context);
    filterQueueClient = queue.readClient();

    // 初始化性能监测指标
    ...
    return context.nil;
}
```

先看 QueueFactoryExt.create(ThreadContext, IRubyObject, IRubyObject)

```java
@JRubyMethod(meta = true)
public static AbstractWrappedQueueExt create(final ThreadContext context, final IRubyObject recv,
    final IRubyObject settings) throws IOException {
    final String type = getSetting(context, settings, "queue.type").asJavaString();
    if ("persisted".equals(type)) {
        // 初始化持久化队列
        ...
    } else if ("memory".equals(type)) {
        return new JrubyWrappedSynchronousQueueExt(
            context.runtime, RubyUtil.WRAPPED_SYNCHRONOUS_QUEUE_CLASS
        ).initialize(
            context, context.runtime.newFixnum(
                getSetting(context, settings, "pipeline.batch.size")
                    .convertToInteger().getIntValue()
                    * getSetting(context, settings, "pipeline.workers")
                    .convertToInteger().getIntValue()
            )
        );
    } else {
        // 异常处理
        ...
    }
}
```

启动内存式队列时，最终初始化的队列对象为 JrubyWrappedSynchronousQueueExt ，而队列的核心为一个有界阻塞队列，其长度为 Logstash 配置文件中的 worker 数量乘上每个批量的大小。**请记住这个阻塞队列**。

```java
private BlockingQueue<JrubyEventExtLibrary.RubyEvent> queue;

@JRubyMethod
@SuppressWarnings("unchecked")
public JrubyWrappedSynchronousQueueExt initialize(final ThreadContext context,
    IRubyObject size) {
    int typedSize = ((RubyNumeric)size).getIntValue();
    this.queue = new ArrayBlockingQueue<>(typedSize);
    return this;
}
```

### 2. init write client & read client

回到上面 open_queue 方法，初始化队列完成后，接着初始化两个 Client ：inputQueueClient —— 写客户端、filterQueueClient —— 读客户端。

这两个客户端的初始化方法也在上面的 JrubyWrappedSynchronousQueueExt 类中：

```java
@Override
protected JRubyAbstractQueueWriteClientExt getWriteClient(final ThreadContext context) {
    return JrubyMemoryWriteClientExt.create(queue);
}

@Override
protected QueueReadClientBase getReadClient() {
    // batch size and timeout are currently hard-coded to 125 and 50ms as values observed
    // to be reasonable tradeoffs between latency and throughput per PR #8707
    return JrubyMemoryReadClientExt.create(queue, 125, 50);
}
```

可以看出，读、写客户端的初始化均使用到该阻塞队列，所以可以得到这样一个结论：**Logstash pipeline 的核心使用了一个有界的阻塞队列，输入插件作为写客户端和输出插件（包括过滤器）作为读客户端，实现了一个生产者和消费者的模型**。

所以接下来继续看生产者和消费者的实现细节。

### 3. write client push events to queue

先从 pipeline.rb 中输入插件的初始化看起：

```ruby
def start_inputs
  ...
  # 注册所有输入插件
  register_plugins(@inputs)
  # 启动所有输入插件
  @inputs.each { |input| start_input(input) }
end

def start_input(plugin)
  @input_threads << Thread.new { inputworker(plugin) }
end

def inputworker(plugin)
  Util::set_thread_name("[#{pipeline_id}]<#{plugin.class.config_name}")
  begin
    # 执行输入插件的 run 方法，传入一个 writeClient
    plugin.run(wrapped_write_client(plugin.id.to_sym))
  rescue => e
    ...
  ensure
    plugin.do_close
  end
end
```

这个 `wrapped_write_client` 同样也是一个 Java 方法，定义在上面提到的 AbstractPipelineExt 类中，它最终初始化了一个 JRubyWrappedWriteClientExt 对象并返回，而该 WrappedWriteClient 初始化时，也传入了上面步骤 2 中初始化过的 QueueWriteClient。

```java
@JRubyMethod(name = "wrapped_write_client", visibility = Visibility.PROTECTED)
public final JRubyWrappedWriteClientExt wrappedWriteClient(final ThreadContext context,
    final IRubyObject pluginId) {
    return new JRubyWrappedWriteClientExt(context.runtime, RubyUtil.WRAPPED_WRITE_CLIENT_CLASS)
        .initialize(inputQueueClient, pipelineId.asJavaString(), metric, pluginId);
}
```

这里使用委托模式实现了两种写客户端，WrappedWriteClient 是封装给 Ruby 实现的输入插件直接使用的，其内部还包括了对各种监控指标的计算；而 QueueWriteClient 是 Logstash 内部真正用于队列操作的客户端实现，有 push 单条数据、push 批量数据等方法。

关于输入插件如何将数据写入队列，这里以 [logstash-input-beats](https://github.com/logstash-plugins/logstash-input-beats) 插件为例：

```ruby
#------------------ beats.rb ------------------
def run(output_queue)
  # 上面提到的执行各输入插件的 run 方法，这里传入的正是一个 WrappedWriteClient
  # 初始化一个监听器（由 netty 实现）并启动监听
  message_listener = MessageListener.new(output_queue, self)
  @server.setMessageListener(message_listener)
  @server.listen
end

#------------------ message_listener.rb ------------------
def initialize(queue, input)
  @connections_list = ThreadSafe::Hash.new
  @queue = queue
  # 其他初始化
  ...
end

def onNewMessage(ctx, message)
  hash = message.getData
  # 将核心数据内容取出
  target_field = extract_target_field(hash)

  if target_field.nil?
    # 将接收到的内容直接初始化一个 event 对象，并 push 进队列
    event = LogStash::Event.new(hash)
    @nocodec_transformer.transform(event)
    @queue << event
  else
    # 使用自定义的 codec callback 处理接收到的内容，后续同样是初始化一个 event
    # 对象，并 push 进队列
    codec(ctx).accept(CodecCallbackListener.new(target_field,
                                                hash,
                                                message.getIdentityStream(),
                                                @codec_transformer,
                                                @queue))
  end
end
```

其余的插件理论上来说应该都差不多，这里还有个有意思的地方，`@queue << event` 中的 `<<` 是 Ruby 中向数组中追加元素的操作符，在 Logstash 中对该操作符进行了一个“重写”，也就是 WrappedWriteClient 最终向队列生产数据的代码：

```java
@JRubyMethod(name = {"push", "<<"}, required = 1)
public IRubyObject push(final ThreadContext context, final IRubyObject event)
    throws InterruptedException {
    final long start = System.nanoTime();
    // 计数 + 1
    incrementCounters(1L);
    // 调用 QueueWriteClient 向阻塞队列中 put 一条消息
    final IRubyObject res = writeClient.doPush(context, (JrubyEventExtLibrary.RubyEvent) event);
    // 累加耗时
    incrementTimers(start);
    return res;
}
```

这里每条消息 put 成功后累加的耗时，就是每个输入插件的性能指标里名为 `queue_push_duration_in_millis` 的耗时。正是由于队列为有界阻塞队列，如果队列已满，put 操作会阻塞等待，所以该值越大，说明 Logstash 的输出环节处理越慢，也就是性能越差。

至此，理清楚了 Logstash 输入插件的大致逻辑和 `queue_push_duration_in_millis` 的含义。

### 4. read client read events from queue

同样，读客户端的源头还要从 pipeline.rb 中看起：

```ruby
def worker_loop(batch_size, batch_delay)
  filter_queue_client.set_batch_dimensions(batch_size, batch_delay)
  output_events_map = Hash.new { |h, k| h[k] = [] }
  while true
    signal = @signal_queue.poll || NO_SIGNAL

    # 读客户端拉取一批数据
    batch = filter_queue_client.read_batch.to_java
    batch_size = batch.filteredSize
    if batch_size > 0
      @events_consumed.add(batch_size)
      # 执行所有的 filter 插件的逻辑
      filter_batch(batch)
    end
    flush_filters_to_batch(batch, :final => false) if signal.flush?
    if batch.filteredSize > 0
      # 执行所有的 output 插件的逻辑
      output_batch(batch, output_events_map)
      filter_queue_client.close_batch(batch)
    end
    break if (@worker_shutdown.get && !draining_queue?)
  end
  # 收到关停信号后的逻辑
  ...
end

def filter_batch(batch)
  filter_func(batch.to_a).each do |e|
    batch.merge(e) unless e.cancelled?
  end
  filter_queue_client.add_filtered_metrics(batch.filtered_size)
  @events_filtered.add(batch.filteredSize)
rescue Exception => e
  ...
end

def output_batch(batch, output_events_map)
  batch.to_a.each do |event|
    output_func(event).each do |output|
      output_events_map[output].push(event)
    end
  end
  output_events_map.each do |output, events|
    output.multi_receive(events)
    events.clear
  end
  filter_queue_client.add_output_metrics(batch.filtered_size)
end
```

整体分为两步，第一步，读客户端从队列中拉取一部分数据，第二步，执行过滤器、输出插件的处理逻辑。

先看读客户端是怎么拉取数据的，`filter_queue_client` 同样是定义在上面提到的 AbstractPipelineExt 类中，它返回了步骤 2 中初始化的 JrubyMemoryReadClientExt 对象~~（这里额外插一句，初始化时代码中写死了 batchSize 和 waitForMillis 两个参数，就是说 Logstash 配置文件中的该参数不会生效，原因是开发人员经过一定的测试，选定了一个他们认为性能较好配置）~~

--- update 2021-04-10 ---

关于 Java 代码中写死的 batchSize 和 waitForMillis 两个参数，经过一些测试后，发现 logstash.yml 中配置了之后，实际是会生效的，于是回过头来继续看看是怎么回事。

这个文件的[第一次提交](https://github.com/elastic/logstash/pull/8991/files/bfcfdabfaf58414abd3ec70bebb94a2972f57b0d)是源自他们在 Logstash 6.0 版本后从 Ruby 代码更新成 Java 实现，Ruby代码里最早就写成了两个固定值，batchSize = 125，waitForMillis = 5，而 [Issue #8707](https://github.com/elastic/logstash/issues/8707) 和 [PR #8702](https://github.com/elastic/logstash/pull/8702) 当时测试的主要是 waitForMillis 由 5 毫秒调整为 50 毫秒，batchSize 125 还是延续了以前的设定。

那么重点来了，这两个值在代码里初始化时写死，可以说是一个历史遗留问题，那为什么没人去改正它们，因为对 `pipeline.batch.size` 和 `pipeline.batch.delay` 这两个参数来说，其实是在 Ruby 代码里设置的，也就在上面的 pipeline.rb 文件的 `worker_loop` 方法的第一行 `filter_queue_client.set_batch_dimensions(batch_size, batch_delay)`，看代码的时候不小心忽视了这一行，翻看了其 Java 部分的代码，该 `set_batch_dimensions` 方法是在 JrubyMemoryReadClientExt.java 类的父类中定义的。

所以，简单来说，读客户端初始化时，代码中直接搬老版 Ruby 代码的做法，写死了两个参数，但最终在插件运行时，还会重新将两个参数设置成 logstash.yml 文件中配置的数值。

就是这样，错误修正结束，下面回到原文。

该类的 readBatch 方法如下：

```java
@Override
public QueueBatch readBatch() throws InterruptedException {
    MemoryReadBatch batch = MemoryReadBatch.create(
            LsQueueUtils.drain(queue, batchSize, waitForNanos));
    startMetrics(batch);
    return batch;
}
```

使用工具类从队列中排出数据，并转换成 Batch 对象返回。因为是阻塞队列，可能会遇到队列中没有数据的情况，所以 drain 方法还有一个等待超时时间，Logstash 有其自己的设计，这里就把实现部分摘抄出来，不做过多的解释了（感觉解释起来更乱，不如直接读源码）：

```java
public static LinkedHashSet<JrubyEventExtLibrary.RubyEvent> drain(
    final BlockingQueue<JrubyEventExtLibrary.RubyEvent> queue, final int count, final long nanos
) throws InterruptedException {
    int left = count;
    //todo: make this an ArrayList once we remove the Ruby pipeline/execution
    final LinkedHashSet<JrubyEventExtLibrary.RubyEvent> collection =
        new LinkedHashSet<>(4 * count / 3 + 1);
    do {
        final int drained = drain(queue, collection, left, nanos);
        if (drained == 0) {
            break;
        }
        left -= drained;
    } while (left > 0);
    return collection;
}

private static int drain(final BlockingQueue<JrubyEventExtLibrary.RubyEvent> queue,
    final Collection<JrubyEventExtLibrary.RubyEvent> collection, final int count,
    final long nanos) throws InterruptedException {
    int added = 0;
    do {
        added += queue.drainTo(collection, count - added);
        if (added < count) {
            final JrubyEventExtLibrary.RubyEvent event =
                queue.poll(nanos, TimeUnit.NANOSECONDS);
            if (event == null) {
                break;
            }
            collection.add(event);
            added++;
        }
    } while (added < count);
    return added;
}
```

从队列中取到数据，接着就是第二步，由配置的插件们处理数据，因为没怎么研究 Ruby 部分怎么根据配置文件初始化过滤器和输出插件的（涉及 config_ast.rb），这里就只从 Java 代码部分简单描述下输出插件的实现。

在代码中能找到 output 相关的三个类：AbstractOutputDelegatorExt、OutputDelegatorExt、OutputStrategyExt，从命名可以看出，输出插件使用了委托模式和策略模式实现。

在 OutputStrategyExt 类中有三种实现，分别是：Legacy —— 一个 Logstash v6 之后被废弃的实现、Single —— 现在版本中的单线程输出插件、Shared —— 现在版本中的多线程输出插件。

而如果搜索 Ruby 代码中 output 调用的 `multi_receive` 方法，会在 AbstractOutputDelegatorExt 类中找到实现，代码如下：

```java
@JRubyMethod(name = OUTPUT_METHOD_NAME)
public IRubyObject multiReceive(final IRubyObject events) {
    final RubyArray batch = (RubyArray) events;
    final int count = batch.size();
    // 增加输入计数
    eventMetricIn.increment((long) count);
    final long start = System.nanoTime();
    // 执行 output 插件处理逻辑
    doOutput(batch);
    // 增加耗时
    eventMetricTime.increment(
        TimeUnit.MILLISECONDS.convert(System.nanoTime() - start, TimeUnit.NANOSECONDS)
    );
    // 增加输出计数
    eventMetricOut.increment((long) count);
    return this;
}
```

继续跟 `doOutput` 方法可以在 OutputStrategyExt 类中发现，它最终又会调用 Ruby 代码中每个输出插件实现的 `multi_receive` 方法。

而上面的代码中涉及的三个指标数据就是输出插件的 `in`、`out` 和 `duration_in_millis`，所以每个输出插件的 `duration_in_millis` 可以说就是其执行业务处理的真正耗时。

至此，理清楚了 Logstash 输出插件（包括过滤器）的大致逻辑和 `duration_in_millis` 的含义。

---

# 最后

这是在 Logstash v6.4.0 版本的源码基础上进行的梳理学习，因为没能在本地启动源码进行更直接的调试分析，所以可能会有部分解析存在错误，希望可以在评论中指出和讨论。

另外，梳理这部分源码逻辑的初衷是为了查清楚 `queue_push_duration_in_millis` 和 `duration_in_millis` 两个监控指标的真实含义，来确定自己对官方监控 API 中的数据的理解，所以文章中的解析没有非常深入和专业（描述过多会显得啰嗦，也没有写这种源码解析类博客的经验），更多的是为了记录下自己觉得关键的代码细节，方便以后翻看。

最后的最后，还是忍不住想要吐槽 Logstash 这种源码中混杂着 Java 和 Ruby 的情况，一会儿代码在 Java 里，一会儿代码又在 Ruby 里，真的是让人晕头转向。不过官方在 7.0 版本之后推出了 Java 版本的插件开发，不知道未来会不会把剩下的那些部分也全改成 Java 实现。

最后的最后的最后，这篇博客拖拖拉拉从12月写到了新年，真是太难了。。。
