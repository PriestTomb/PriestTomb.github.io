---
layout: post
title: Spring Data MongoDB 聚合并去重的一个例子
date: 2022-03-24
categories:
- 技术
tags: [Spring, MongoDB]
status: publish
type: post
published: true
---

# 1 需求举例

以这个表来举例吧，精简这三个字段

| id | userId | createTime |
| -- | -- | -- |
| 1 | 1 | 2021-01-01 12:00:00 |
| 2 | 2 | 2021-01-01 12:00:00 |
| 3 | 1 | 2021-01-02 12:00:00 |
| 4 | 1 | 2021-02-05 12:00:00 |
| 5 | 2 | 2021-03-10 12:00:00 |
| 5 | 3 | 2021-03-11 12:00:00 |
| 5 | 4 | 2021-03-12 12:00:00 |

需要 createTime 按月统计，并再统计每个月中有几个不同的用户（即对 userId 再去重），得到如下的结果

| month | totalCount | userCount |
| -- | -- | -- |
| 2021-01 | 3 | 2 |
| 2021-02 | 1 | 1 |
| 2021-03 | 3 | 3 |


---

<h3>混入防转防爬防抄袭声明：本文<a href="https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2022/03/24/spring-data-mongodb-group-by-and-distinct/">《Spring Data MongoDB 聚合并去重的一个例子》</a>首发且仅发布于<a href="https://priesttomb.github.io/">没有名字的博客</a></h3>

---

# 2 MongoDB 脚本

**$project**: 定义字段，不定义，在后续的 $group 中不能用。用 userId 字段代表 userId 字段，用 $dateToString 处理 createTime 字段，代表 month。

**第一次$group**: 指定 month + userId 共同作为聚合条件，并 sum(1) 汇总每个条件对应的条数。

**第二次$group**: 指定第一次聚合条件中的 month 为新的聚合条件，并 sum 之前统计好的条数，得到总的条数，并再次 sum(1) 条数（等同于 count），得到有多少个不同的 userId。

```javascript
db.testDoc.aggregate([
    {
        $match: {
            createTime: {
                $gte: ISODate("2021-01-01T00:00:00.000+08:00"),
                $lte: ISODate("2021-12-31T23:59:59.000+08:00")
            }
        }
    },{
        $project: {
            userId: "$userId",
            month: {
                $dateToString: {
                    format: "%Y-%m",
                    date: "$createTime"
                }
            }
        }
    },{
        $group: {
            _id: {
                "month": "$month",
                "userId": "$userId"
            },
            count: {
                $sum: 1
            }
        }
    },{
        $group: {
            _id: {
                "month": "$_id.month"
            },
            totalCount: {
                $sum: "$count"
            },
            userCount: {
                $sum: 1
            }
        }  
    },
    {
        $sort: {
            _id: 1
        }
    }
])
```

---

# 3 Java 代码

定义一个 Aggregation，开始往里拼各个子语句。后面定义一个 CountDto 对象接收这个有三个字段的统计结果。

```java
Aggregation aggregate = Aggregation.newAggregation(
    Aggregation.match(CriteriaBuilder.between("createTime", startTime, endTime)),
    Aggregation.project("userId")
        .and(DateOperators.DateToString.dateOf("createTime").toString("%Y-%m")).as("month"),
    Aggregation.group("month", "userId").count().as("muCount"),
    Aggregation.group("month").sum("muCount").as("totalCount").count().as("userCount"),
    Aggregation.sort(Direction.ASC, "_id"),
    Aggregation.project().andExpression("_id").as("month")
        .andExpression("totalCount").as("totalCount")
        .andExpression("userCount").as("userCount")
);
List<CountDto> resultList = mongoTemplate.aggregate
		(aggregate, TestDoc.class, CountDto.class).getMappedResults();
```

最后又加了一个 project 的主要目的是给最终 group 完的结果中的字段加自定义别名，比如像第 2 节的脚本所示，第二次 group 时指定 _id 为 month，但到了代码中，试验过用 month 接收不到，所以用 `andExpression` + `as` 强行再给 _id 加个别名。

还有，如果加了这个 project 用于指定自定义别名，就得把所有的字段都重新指定一遍，否则结果就缺失了。
