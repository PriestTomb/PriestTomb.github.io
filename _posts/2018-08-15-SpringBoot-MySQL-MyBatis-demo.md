---
layout: post
title: SpringBoot+MySQL+MyBatis的一个小Demo
date: 2018-08-15
categories:
- 技术
tags: [SpringBoot, MySQL, MyBatis]
status: publish
type: post
published: true
---

## 写在前面

之前用 JPA 体验了一下 SpringBoot（[这篇](https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2018/08/06/SpringBoot-MySQL-JPA-demo/)），最近也想试试 MyBatis，就在上一个 demo 的基础上改用 MyBatis 实现了一下。

---

## 项目结构

![项目结构.png](/images/blog_img/20180815/项目结构.png)

---

## 主要代码配置

源码同样丢 github 上去了，可以点[这里](https://github.com/PriestTomb/SpringBootDemo/tree/master/SpringBoot-MySQL-MyBatis)查看

下面列一些主要的代码和配置

#### 0. application.yml

```yml
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/SpringBootDemo?autoReconnect=true&useSSL=false&useUnicode=true&characterEncoding=utf-8
    username: root
    password: mysql

server:
  port: 8080

mybatis:
  type-aliases-package: com.priesttomb.demo.vo
  mapper-locations: classpath:mybatis/mapper/*.xml
```

这里简单的配置了 MyBatis 的别名包路径，和相关 mapper.xml 文件的路径

稍微测试了一下，如果在 mapper.xml 里把各个实体类的路径写完整的话，这里的 `type-aliases-package` 可以不用配（比如下面的 BookMapper.xml 中，Book 类的路径都带了完整的包路径）

#### 1. BookMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.priesttomb.demo.mapper.IBookMapper">
	<resultMap type="com.priesttomb.demo.vo.Book" id="BaseResultMap">
		<result column="id" property="id" />
		<result column="name" property="name" />
		<result column="price" property="price" />
	</resultMap>

	<sql id="Base_Column">
		id, name, price
	</sql>

	<select id="findAll" resultMap="BaseResultMap">
		select
		<include refid="Base_Column" />
		from book
	</select>

	<select id="findById" resultMap="BaseResultMap" parameterType="int">
		select
		<include refid="Base_Column" />
		from book
		where id = #{id}
	</select>

	<select id="findByName" resultMap="BaseResultMap" parameterType="java.lang.String">
		select
		<include refid="Base_Column" />
		from book
		where name = #{name}
	</select>

	<insert id="save" useGeneratedKeys="true" keyProperty="id" parameterType="com.priesttomb.demo.vo.Book">
		insert into book (name, price)
		values (#{name}, #{price})
	</insert>

	<delete id="deleteById">
		delete from book where id = #{id}
	</delete>

	<update id="updateBook" parameterType="com.priesttomb.demo.vo.Book">
		update book
		<set>
			<if test="name != null and name != ''">name = #{name},</if>
			<if test="price != null">price = #{price}</if>
		</set>
		where id = #{id}
	</update>
</mapper>
```

mapper.xml 这儿就是配一下 sql 了，`<insert>` 和 `<update>` 需要配一下 `parameterType`，因为到时候接收的参数就是一个实体类对象

#### 2. SpringBootMySqlMyBatisApplication.java

```java
package com.priesttomb.demo;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan("com.priesttomb.demo.mapper")
public class SpringBootMySqlMyBatisApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootMySqlMyBatisApplication.class, args);
	}
}
```

启动类这里可以加一个 `@MapperScan` 注解，要么就直接在对应的 Mapper 类上加 `@Mapper` 注解，都行

#### 3. IBookMapper.java

```java
package com.priesttomb.demo.mapper;

import java.util.List;
import org.apache.ibatis.annotations.Mapper;
import com.priesttomb.demo.vo.Book;

//@Mapper
public interface IBookMapper {
	Long save(Book book);

	List<Book> findAll();

	Book findById(Integer id);

	Book findByName(String name);

	void deleteById(Integer id);

	int updateBook(Book book);
}
```

正如上面所说，也可以直接在这儿加 `@Mapper` 注解

#### 4. BookController.java

```java
package com.priesttomb.demo.controller;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import com.priesttomb.demo.service.IBookService;
import com.priesttomb.demo.vo.Book;

@RestController
public class BookController {

	@Autowired
	private IBookService bookService;

	/**
	 * 添加一本书（用form-data传来的各个字段）
	 * @param name
	 * @param price
	 */
//	@PostMapping(value = "/book")
//	public void addBook(@RequestParam(value="name") String name,@RequestParam(value="price") Integer price) {
//		Book book = new Book(name, price);
//		bookService.saveBook(book);
//	}
	/**
	 * 添加一本书（使用json格式直接传递一个对象）
	 */
	@PostMapping(value = "/book", consumes = "application/json")
	public void addBook(@RequestBody Book book) {
		bookService.saveBook(book);
	}

	/**
	 * 获取所有的书
	 * @return
	 */
	@GetMapping(value = "/book")
	public List<Book> getAllBooks() {
		return bookService.getAllBooks();
	}

	/**
	 * 根据书的id获取书
	 * @param id
	 * @return
	 */
	@GetMapping(value = "/book/{id}")
	public Book getBookById(@PathVariable("id") Integer id) {
		return bookService.getBookById(id);
	}

	/**
	 * 根据书的id删除书
	 * @param id
	 */
	@DeleteMapping(value = "/book/{id}")
	public void delBookById(@PathVariable("id") Integer id) {
		bookService.delBookById(id);
	}

	/**
	 * 根据书名查书
	 * @param name
	 * @return
	 */
	@GetMapping(value = "/book/name/{name}")
	public Book findByName(@PathVariable("name") String name) {
		return bookService.findByName(name);
	}

	/**
	 * 更新一本书（使用json格式直接传递一个对象）
	 * @param book
	 * @param id
	 * @return
	 */
	@PutMapping(value = "/book/{id}", consumes = "application/json")
	public String updateById(@RequestBody Book book, @PathVariable("id") Integer id) {
		int result = bookService.updateBook(book);
		return result == 1 ? "succ" : "fail";
	}
}
```

新增和更新时传递 Book 对象，这里考虑了下用了 json

---

## 运行测试

还是用 [Postman](https://www.getpostman.com/apps) 测试，在使用 PUT 或者 POST 时，要使用 json 传递对象，就要设置下 Content-Type：

![contenttype设置成json.png](/images/blog_img/20180815/contenttype设置成json.png)

拼接对象：

![PUT更新.png](/images/blog_img/20180815/PUT更新.png)

测试正常，over
