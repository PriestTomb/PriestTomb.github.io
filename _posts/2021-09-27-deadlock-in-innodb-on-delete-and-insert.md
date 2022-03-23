---
layout: post
title: 记一次 innodb 间隙锁导致的死锁分析
date: 2021-09-27
categories:
- 技术
tags: [mysql, innodb, deadlock]
status: publish
type: post
published: true
---

# 1 竟然死锁

项目中有一个简单的批量审核功能，自己测试时没问题，在给同事演示的时候突然就有报错，一时让人尴尬，于是登到服务器，查一下后台的报错日志：

```
Cause: com.mysql.jdbc.exceptions.jdbc4. MySQLTransactionRollbackException :
Deadlock found when trying to get lock; try restarting transaction
```

居然出现死锁了？之前默认按批量 20 的操作，这次想批量快一点，选了 100，难道这就有问题了？

因为对数据库这块一向研究不深，头回遇到个死锁问题还有点小激动。。决定查一查到底是什么导致的死锁。

---

# 2 太长不看

基于这次的死锁场景所总结：

* 使用一个普通**非唯一索引**进行 delete 操作时，会加**间隙锁**

* 不管是共享间隙锁还是排他间隙锁，是可以**共存的**，多个事务可以对同一个间隙进行加锁

* 多个事务在拥有相同的间隙锁后，再同时向该间隙内进行 insert 操作，死锁出现

* 为了避免死锁，根据具体的业务情况，尽量避免不必要的 delete 操作

---

# 3 环境说明

MySQL 5.7

事务隔离级别 Repeated Read

问题表结构简化示例

```sql
CREATE TABLE `test_deadlock` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `index_col` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_index_col` (`index_col`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

---

# 4 问题流程

Mybatis 的报错日志中显示是执行一条 insert 语句（表结构如第 3 节中所示）时出现了死锁导致事务回滚，简化说明下这个事务到底做了哪些事：

1) 向一张表 A 中新增一条数据，记为 a

2) 以 a 的主键 id 为关联，向一张关联表 B 中新增一条数据，记为 b

3) 以 a 的主键 id 为关联，删除关联表 C 中所有与该 id 有关的数据

4) 以 a 的主键 id 为关联，重新向关联表 C 中新增多条数据，记为 c1、c2、c3...

关联表 C 即为问题表，`index_col` 即关联到表 A 的主键。一个事务中可能会向 C 表插入多条记录，但重点是，做插入操作前，会先用加了普通索引的 `index_col` 字段做删除操作。

---

<h3>混入防转防爬防抄袭声明：本文<a href="https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2021/09/27/deadlock-in-innodb-on-delete-and-insert/">《记一次 innodb 间隙锁导致的死锁分析》</a>首发且仅发布于<a href="https://priesttomb.github.io/">没有名字的博客</a></h3>

---

# 5 死锁日志

经过一些无头绪的测试无果后，查到了查询 innodb 死锁日志的命令 `show engine innodb status;`，这里贴一下最终复现问题场景时的日志：

```
*** (1) TRANSACTION:
TRANSACTION 1371, ACTIVE 182 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1
MySQL thread id 10, OS thread handle 9948, query id 606 localhost ::1 root update
insert into test_deadlock(`index_col`) values (1)
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 23 page no 4 n bits 72 index idx_index_col of table `test`.`test_deadlock` trx id 1371 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) TRANSACTION:
TRANSACTION 1372, ACTIVE 68 sec inserting, thread declared inside InnoDB 5000
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1
MySQL thread id 14, OS thread handle 13120, query id 616 localhost ::1 root update
insert into test_deadlock(`index_col`) values (2)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 23 page no 4 n bits 72 index idx_index_col of table `test`.`test_deadlock` trx id 1372 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 23 page no 4 n bits 72 index idx_index_col of table `test`.`test_deadlock` trx id 1372 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
```

这个东西实话说完全看不懂，简单看了几篇讲死锁日志的文章如[《解决死锁之路（终结篇）- 再见死锁》](https://cloud.tencent.com/developer/article/1494926)之后，更傻了，我还是分不清这日志里等待的锁到底是间隙锁还是插入意向锁。

---

# 6 胡乱搜索

因为报错的日志都指向了 insert 语句，所以最初以为是插入操作时自增主键是不是有一个自增锁，导致的什么表级锁，就去搜索插入时出现死锁的文章，在看[《【Mysql】两条insert 语句产生的死锁》](http://blog.itpub.net/29096438/viewspace-2146310/)时，博主提到**“基于以上，事务除了INSERT，可能还存在DELETE/UPDATE，并且这些操作是走的二级索引来查找更新记录”**。

这时想起来，业务流程里确实有先删除再新增，于是又去搜删除时死锁的文章，看[《Mysql innodb 间隙锁: 删除不存在的记录会产生间隙锁》](https://blog.csdn.net/tianya_lu/article/details/112562018)时得到了一个新的观点**“删除操作时如果记录不存在，会加间隙锁”**，同时，文章中还提到具体的“间隙”。

顺手搜索了一下官方文档中[间隙锁](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-gap-locks)的描述，发现提到了：

> Gap locks can co-exist. A gap lock taken by one transaction does not prevent another transaction from taking a gap lock on the same gap.

此时，再回想一下出问题的事务，以及死锁日志中的两条 insert 语句，已经冥冥中感觉找到了问题所在。

---

# 7 复现场景

先详细描述下死锁日志中出问题的两条 insert 语句在实际操作中还有什么细节：

1) 两条语句分别属于两个事务

2) 这两条 insert 语句执行之前，还各自有一条 delete 语句

3) delete 语句没有使用主键 ID 进行精确删除，而是使用了一个加了普通非唯一索引的列（即表 A 的主键）

4) 两条 delete 语句使用的普通非唯一索引是相邻的，因为这两个事务刚刚连续分别向表 A 插入了两条记录

5) 因为 4，这两个索引在表 C 中还不存在，而且这两个索引比表 C 中已有的索引值都大

根据这些细节，来对这一个 delete + insert 操作做一个步骤划分：

|    | session A | session B |
| -- | -- | -- |
|  1 | begin | begin |
|  2 | delete from test_deadlock where index_col = 10; | |
|  3 | | delete from test_deadlock where index_col = 11; |
|  4 | insert into test_deadlock(index_col) values(10); | |
|  5 | waiting | insert into test_deadlock(index_col) values(11); |
|  6 | success | dead lock and rollback |

按照这个步骤，进行测试的时候，发现产生死锁的日志与生产环境真实的报错一模一样，且这个测试步骤与还原的业务操作也完全一致，所以确定，就是这样的流程导致了死锁产生。

下面来简单分析下，这种操作究竟产生了什么样的间隙锁。

正如前面 5) 中所述，这两个索引（例子中的 index_col = 10 / 11）其实还不存在，如果假设目前表中 index_col 的最大值为 9，那么在删除时，两个事务都持有了一个 `(9,+∞)` 的间隙锁，所以当 session A 想要执行 insert 操作时，就会提示事务等待，事务日志大概是这样：

```
------- TRX HAS BEEN WAITING 6 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 23 page no 4 n bits 72 index idx_index_col of table `test`.`test_deadlock` trx id 1371 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
```

然后，当 session B 执行 insert 操作，发现它也要等待 session A 的间隙锁，两者互相等待，死锁出现。随即引擎根据两个事务占有的锁、影响的行等信息，判定 session B 回滚，session A 执行成功。

---

# 8 假装解决

回过头来会发现，在这个场景中，其实压根没有必要执行 delete 操作，因为数据本身就还不存在，那么为什么代码里会写先 delete 呢？

这个问题聊起来就有点尴尬了，因为这个先删除后新增的方法是多个地方共同调用的，其他业务方法中需要做删除 + 新增操作，实现数据的更新，而在出问题的场景中，只是纯粹复用了这个方法，觉得反正没数据，先删也不影响业务，而没考虑多执行了一步无意义的删除会不会有其他幺蛾子产生。

结果显而易见，真就出了点幺蛾子。。

所以为了避免并发操作下非常容易出现的死锁，调整了业务逻辑，在纯粹新增时，不执行多余的删除操作，于是，死锁再也没有出现。

---

# 9 写在最后

此文只是简单记录下一种“可能”很常见的代码写法中，会冷不丁产生死锁问题。

还是前面说到的，因为对数据库知识实在不擅长，所以就没有深入讲解这些锁以及对日志的分析，这类文章网上非常多，个人在尝试解决这个死锁问题时，搜索到不少详细的博客文章，学习到不少（吸收了很少。

而对间隙锁的范围，仅仅简单测试了一些，至少在这次的问题场景中确认了间隙范围无误。另外还看到有人提到非唯一索引和唯一索引对间隙锁的影响，个人没有测试，只是在一篇[《MySQL DELETE 删除语句加锁分析》](https://www.fordba.com/lock-analyse-of-delete.html)中看到博主做了比较详细的测试。
