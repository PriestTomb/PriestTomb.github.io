---
layout: post
title: elasticsearch-curator中的age filter在东八区下是如何计算的
date: 2019-08-03
categories:
- 技术
tags: [ELK, Elasticsearch, curator]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

# 写在前面

[curator](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/about.html) 是 Elasticsearch 官方提供的一个管理工具，如果想要定时新建或删除索引、执行 force merge 等，那就可以使用它来实现了

在刚使用 curator 的时候，主要的目的就是删除 N 天以前的索引、新建 N 天以后的索引/别名等，所以用到了 [age](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/filtertype_age.html) 这个 filter 来过滤出符合条件的索引

但计算要比较的时间节点还是踩了一点坑，正是因为时区的问题

---

# 版本说明

Elasticsearch 6.4.0

curator 5.7.6

---

# 官方概念

### 1. age 的配置

示例：

```yaml
- filtertype: age
  source: name
  direction: older
  timestring: '%Y.%m.%d'
  unit: days
  unit_count: 3
```

说明：

| 配置项 | 描述 |
| --- | --- |
| `source` | 可以配置 `name` 或 `creation_date`，`name` 是根据索引的名称，需要再配置  `timestring`；`creation_date` 是索引的创建时间，不需要再配置 `timestring` |
| `direction` | 可以配置 `older` 或 `younger`，代表过滤条件是比时间点大还是小 |
| `timestring` | 必须是一个合法的 Python strftime 格式的字符串 |
| `unit` | 单位，详见下面第二节的表格 |
| `unit_count` | 单位的倍数，详见下面第三节的计算公式 |


### 2. unit 的单位

| Unit | 秒数 | 备注 |
| -- | -- | -- |
| `seconds` | 1 | 1秒 |
| `minutes` | 60 | 以 60 秒算 |
| `hours` | 3600 | 以 60 分算 (60\*60) |
| `days` | 86400 | 以 24 小时算 (24\*60\*60) |
| `weeks` | 604800 | 以 7 天算 (7\*24\*60\*60) |
| `months` | 2592000 | 以 30 天算 (30\*24\*60\*60) |
| `years` | 31536000 | 以 365 天算 (365\*24\*60\*60) |


### 3. 比较时间点的计算公式

curator 先根据配置，计算出一个用来比较的基准时间点，再根据 `direction` 对所有符合条件的索引进行比较过滤，而这个时间点的时间戳就是用下面的公式计算得出的

`point_of_reference = epoch - ((number of seconds in unit) * unit_count)`

---

# 使用场景说明

测试的索引是随时间滚动的，所以索引的名称类似 `test_20190801_v1`，其中带有当天时间，末尾还有一个版本号标识

下面先举个栗子

现在是北京时间 2019 年 8 月 1 日凌晨 3 点，执行 curator 脚本，我想过滤出 2019 年 8 月 2 日那一天的索引，需要这么写

```yaml
- filtertype: age
  source: name
  direction: younger
  timestring: '%Y%m%d'
  unit: days
  unit_count: -1
```

但假如现在是北京时间 2019 年 8 月 1 日上午 10 点，我才执行这个 curator 脚本，同样想过滤出 2019 年 8 月 2 日的索引，就需要改一个数字了

```yaml
- filtertype: age
  source: name
  direction: younger
  timestring: '%Y%m%d'
  unit: days
  unit_count: 0
```

不同的地方就是 `unit_count`，为什么 3 点和 10 点会不同，其根源就在于我们想要的 2019 年 8 月 2 日的索引（索引名称 `test_20190802_v1`），在当前东八区下被比较的时候，索引的小时数不是 0 点，而是 8 点

代入计算公式解释一下：

如果是凌晨 3 点执行，则用于比较的时间点就是当前时间减去 -1 天，就是加 1 天，变成了 2019 年 8 月 2 日 3 点，转成时间戳就是 1564599600 秒，比较方式是 `younger`，就是说要过滤出的索引的时间要比 1564599600 大，而**我们需要过滤出的 20190802 的索引，实际上还有一个小时数，是 8**，2019 年 8 月 2 日 8 点转成时间戳就是 1564617600 秒，所以可以过滤出来

但如果是上午 10 点执行，如果 unit_count 继续使用 -1 ，则用于比较的时间点就变成了 2019 年 8 月 2 日 10 点，转成时间戳就是 1564624800 秒，而 2019 年 8 月 2 日的索引，上面已经得出时间戳是 1564617600 ，反而小了，就会被过滤掉，所以 unit_count 只能用 0，直接用当前的时间 2019 年 8 月 1 日 10 点做比较时间，这样既能过滤掉 20190801 的索引，又能过滤出 20190802 的索引

再来一个栗子

现在是北京时间 2019 年 8 月 1 日凌晨 3 点，执行 curator 脚本，我想过滤出 3 天以前的索引，要这么写

```yaml
- filtertype: age
  source: name
  direction: older
  timestring: '%Y%m%d'
  unit: days
  unit_count: 3
```

同样，假如现在是北京时间 2019 年 8 月 1 日上午 10 点，我才执行这个 curator 脚本，同样想过滤出 3 天以前的索引

```yaml
- filtertype: age
  source: name
  direction: older
  timestring: '%Y%m%d'
  unit: days
  unit_count: 4
```

---

# 总结

简单总结一下就是，执行时间如果在北京时间 8 点前，unit_count 正常，如果在 8 点后，unit_count + 1

当然，unit 也可以直接用 hours 来配置计算（笑
