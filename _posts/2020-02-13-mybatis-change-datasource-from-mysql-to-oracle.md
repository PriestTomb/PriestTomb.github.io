---
layout: post
title: Spring Boot + MyBatis 数据源从 MySQL 切换到 Oracle
date: 2020-02-13
categories:
- 技术
tags: [MyBatis, SpringBoot, MySQL, Oracle]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

# 写在前面

一个小需求在开发阶段用的是部门常用的 MySQL 数据库，开发测试完准备部署的时候，突然来了句尽量使用生产环境现有的 Oracle 数据库（锅不在我不了解清楚，因为生产环境是 Oracle 数据库这个事儿我在开发前就已经反馈过的，但我们测试环境有一大堆的 MySQL 数据库，之前一直没有部署和用过 Oracle），那得，切个数据源再改改测测吧

本来以为脚本简简单单就是增改查，应该没什么太大问题的，谁知道最终还是遇到了好几个小问题

---

# 从 MySQL 到 Oracle

这里先附一个在 MySQL 库测试完全正常的脚本（用一段最小化的脚本来代替项目中的）

```xml
<insert id="insertXXX" parameterType="com.xxx.CustDo" useGeneratedKeys="true" keyProperty="custId">
    insert into cust (cust_name,content,create_date) values (#{custName},#{content},sysdate())
</insert>
```

接下来，来依次描述遇到了哪些问题并且怎么修改解决的吧。。

### 1. ORA-00942: 表或视图不存在

在 Oracle 库建表我用的是 navicat 进行的图形化操作，表名和字段之类的都用了小写，后来导出数据库 sql 语句一看，表名、字段都被加了双引号，就是强制只能匹配小写的那种

MyBatis 脚本里没引号的情况下，这时候会报找不到表

解决办法是重新用 SQL Plus 执行了一下建表语句，这种情况下，表名和字段应该是不再区分大小写，MyBatis 脚本就不用每个去修改加上引号了

### 2. Cause: java.sql.SQLException: 无效的列类型: 1111

报错的完整内容是

```java
Cause: org.apache.ibatis.type.TypeException: Error setting null for parameter #2 with JdbcType OTHER . Try setting a different JdbcType for this parameter or a different jdbcTypeForNull configuration property. Cause: java.sql.SQLException: 无效的列类型: 1111
```

这个问题是因为 DO 类有些属性是 NULL 值，执行插入语句时 MyBatis 不知道 NULL 值怎么转换

解决方法一是在脚本中加入每个字段对应的 jdbcType，但我这有一大堆脚本几百个字段，懒得这么去搞，就用了另一个方法

在全局配置文件中加一句

```xml
<configuration>
    <settings>
        <setting name="jdbcTypeForNull" value="NULL" />
    </settings>
</configuration>
```

参考了博客[《Mybatis：使用bean传值，当传入值为Null时，提示“无效的列类型”的解决办法》](https://www.cnblogs.com/mmlw/p/5808072.html)

### 3. ORA-00917: 缺失逗号

翻来覆去查了半天脚本到底哪里写错了（又很奇怪 MySQL 下明明都能执行），最后发现是 sysdate() 函数的锅，Oracle 里去掉了括号就行了

### 4. 关于自增主键

MySQL 直接创建了 UUID 主键，所以 MyBatis 脚本中就没有写这一列，但用到 Oracle 时，会报主键不能为 NULL 值，这才想起来 Oracle 还要搞一个主键的生成方式，记得以前就是建个 sequence，所以这次也搞了个 sequence，脚本中用 sql.nextval 去获取

### 5. 无效的列类型: getLong not implemented for class oracle.jdbc.driver.T4CRowidAccessor

原先因为有个需求是插入数据后，返回这条数据的主键 id，所以 MyBatis 中配置了这个

```
useGeneratedKeys="true" keyProperty="custId"
```

但在用 Oracle 之后就不能用 `useGeneratedKeys` 了，就把它配成了 false

另外想要插入成功后返回主键 id 的值，参考了别的博客后，这段小脚本变成了这样

```xml
<insert id="insertXXX" parameterType="com.xxx.CustDo" useGeneratedKeys="false" keyProperty="custId">
    <selectKey resultType="Long" order="BEFORE" keyProperty="custId">  
        select seq_cust.nextval as custId from dual
    </selectKey>

    insert into cust (cust_id,cust_name,content,create_date)
    values (#{custId},#{custName},#{content},sysdate)
</insert>
```

参考了博客[《getLong not implemented for class oracle.jdbc.driver.T4CRowidAccessor》](https://www.cnblogs.com/legendjslc/p/7159171.html#commentform)

### 6. ORA-01465: 无效的十六进制数字

表中的那个 content 字段是 BLOB 类型，在插入时就报了错，先参考了下面这种解决办法

写入时：rawtohex(#{content})

查询时：UTL_RAW.CAST_TO_VARCHAR2(查询字段)

参考了[《mybatis中对Oracle特殊字段做处理》](http://www.jeepxie.net/article/422388.html)

### 7. ORA-01460:转换请求无法实现或不合理解决

上面那种办法最初解决了问题，但后续测试过程中又报了错

因为我拿了一个几万个字节的字符串去写入，报的错误网上有不少的说法，我遇到的这个情况应该就是 SQL 语句太太太太太长了

搜了一下找到个比较正确的做法，之前我的 DO 类中，这个 BLOB 类型字段对应的属性定义的是 String 类型，现在把它改成 byte[] 类型，而脚本中也指定它的 jdbcType

所以脚本变成了这样

```xml
<insert id="insertXXX" parameterType="com.xxx.CustDo" useGeneratedKeys="false" keyProperty="custId">
    <selectKey resultType="Long" order="BEFORE" keyProperty="custId">  
        select seq_cust.nextval as custId from dual
    </selectKey>

    insert into cust (cust_id,cust_name,content,create_date)
    values (#{custId},#{custName},#{content,jdbcType=BLOB},sysdate)
</insert>
```

参考了博客[《mybatis oracle BLOB类型字段保存与读取》](https://www.cnblogs.com/always-online/p/4877962.html)

---

# 最后

至此大概解决了这个看起来仅仅是增改查的 MyBatis 脚本从写入 MySQL 数据库改为写入 Oracle 数据库的一些问题。。

还好仅仅是个小需求，涉及到的功能都很简单，没有过于复杂的表结构和 SQL 脚本，不过以后还是要注意，一定在开发前得到非常准确的生产环境。。
