---
layout: post
title: SpringMVC+Hibernate+Maven+MySQL实现增删改查的一个小Demo
date: 2017-11-27
categories:
- 小代码
tags: [SpringMVC, Hibernate]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

## 写在前面

工作过的项目用的是非常老旧的框架：Spring + Hibernate + DWR，DAO 层的绝大多数现成的方法基本还是内部老前辈们写的，什么 queryList、queryMap、querySingleObj 乱七八糟的，平时都是直接拿来用，也是方便

最近闲下来，就了解一下当下(其实也不是)比较主流的框架，SSH 好像都比较少了，目前大都是 SpringMVC 了来着，持久层要么是 Hibernate，要么是 MyBatis，因为好歹用过 Hibernate，所以先学着写了个小 demo 熟悉下这个框架

---

## 项目结构

Eclipse 新建一个 Dynamic Web Project，然后转成的 Maven Project

![项目结构.png](https://i.loli.net/2018/11/07/5be2f17d78a2d.png)

---

## 数据库表

随便建一张 book 表：

```sql
CREATE TABLE `book` (
  `id` int(6) NOT NULL AUTO_INCREMENT,
  `book_name` varchar(20) DEFAULT NULL,
  `book_author` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=20 DEFAULT CHARSET=utf8;
```

---

## 主要代码配置

#### 0. pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>BookDemo</groupId>
	<artifactId>BookDemo</artifactId>
	<version>1.0.0</version>
	<packaging>war</packaging>
	<build>
		<finalName>BookDemo</finalName>
		<sourceDirectory>src</sourceDirectory>
		<plugins>
			<plugin>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.6.1</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
			<plugin>
				<artifactId>maven-war-plugin</artifactId>
				<version>3.0.0</version>
				<configuration>
					<warSourceDirectory>WebContent</warSourceDirectory>
				</configuration>
			</plugin>
		</plugins>
	</build>

	<properties>
		<spring.version>4.1.5.RELEASE</spring.version>
		<hibernate.version>4.3.8.Final</hibernate.version>
		<mysql.version>5.1.40</mysql.version>
		<junit-version>4.11</junit-version>
		<servlet-api-version>3.1.0</servlet-api-version>
		<jsp-version>2.1</jsp-version>
		<jstl-version>1.2</jstl-version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-core</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-orm</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-test</artifactId>
			<version>${spring.version}</version>
			<scope>test</scope>
		</dependency>

		<!-- Hibernate 4 dependencies -->
		<dependency>
			<groupId>org.hibernate</groupId>
			<artifactId>hibernate-core</artifactId>
			<version>${hibernate.version}</version>
		</dependency>

		<dependency>
			<groupId>org.hibernate</groupId>
			<artifactId>hibernate-c3p0</artifactId>
			<version>${hibernate.version}</version>
		</dependency>

		<!--MYSQL Connector -->
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>${mysql.version}</version>
		</dependency>

		<!-- Servlet and JSP -->
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>javax.servlet-api</artifactId>
			<version>${servlet-api-version}</version>
		</dependency>

		<dependency>
			<groupId>javax.servlet.jsp</groupId>
			<artifactId>jsp-api</artifactId>
			<version>${jsp-version}</version>
			<scope>provided</scope>
		</dependency>

		<!-- JSTL dependency -->
		<dependency>
			<groupId>jstl</groupId>
			<artifactId>jstl</artifactId>
			<version>${jstl-version}</version>
		</dependency>

		<!-- JUnit -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>${junit-version}</version>
			<scope>test</scope>
		</dependency>
	</dependencies>
</project>
```


#### 1. web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="http://java.sun.com/xml/ns/javaee"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
	version="3.0">
	<display-name>Archetype Created Web Application</display-name>
	<welcome-file-list>
		<welcome-file>index.jsp</welcome-file>
	</welcome-file-list>
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/bookDemo-servlet.xml</param-value>
	</context-param>
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
	<servlet>
		<servlet-name>bookDemo</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>bookDemo</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>

	<!-- 编码过滤器 -->
	<filter>
		<filter-name>SpringEncodingFilter</filter-name>
		<filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
		<init-param>
			<param-name>encoding</param-name>
			<param-value>UTF-8</param-value>
		</init-param>
		<init-param>
			<param-name>forceEncoding</param-name>
			<param-value>true</param-value>
		</init-param>
	</filter>
	<filter-mapping>
		<filter-name>SpringEncodingFilter</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>
</web-app>
```

#### 2. applicationContext.xml(bookDemo-servlet.xml)

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:mvc="http://www.springframework.org/schema/mvc" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="
        http://www.springframework.org/schema/beans     
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd">

	<context:component-scan base-package="com.priest.book" />

	<bean class="org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping" />
	<bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter" />

	<bean
		class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<property name="prefix" value="/WEB-INF/views/" />
		<property name="suffix" value=".jsp" />
	</bean>

	<!--配置数据源 -->
	<bean id="dataSource"
		class="org.springframework.jdbc.datasource.DriverManagerDataSource">
		<property name="driverClassName" value="com.mysql.jdbc.Driver" />
		<property name="url" value="jdbc:mysql://127.0.0.1:3306/springmvc_demo?useUnicode=true&amp;characterEncoding=UTF-8" />
		<property name="username" value="root" />
		<property name="password" value="mysql" />
	</bean>

	<!--配置hibernate -->
	<bean id="sessionFactory"
		class="org.springframework.orm.hibernate4.LocalSessionFactoryBean">
		<property name="dataSource" ref="dataSource"></property>
		<property name="hibernateProperties">
			<props>
				<prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
				<prop key="hibernate.show_sql">true</prop>
				<prop key="hibernate.format_sql">true</prop>
			</props>
		</property>
		<property name="packagesToScan" value="com.priest.book.model"></property>
	</bean>
	<bean id="txManager"
		class="org.springframework.orm.hibernate4.HibernateTransactionManager">
		<property name="sessionFactory" ref="sessionFactory"></property>
	</bean>
	<tx:annotation-driven transaction-manager="txManager" />
</beans>
```


#### 3. BookVO.java

```java
import java.io.Serializable;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;

@Entity
@Table(name = "book")
public class BookVO implements Serializable{
	private static final long serialVersionUID = 1L;

	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	private int id;

	@Column(name="book_name")
	private String bookName;

	@Column(name="book_author")
	private String bookAuthor;

	public int getId() {
		return id;
	}

	public void setId(int id) {
		this.id = id;
	}

	public String getBookName() {
		return bookName;
	}

	public void setBookName(String bookName) {
		this.bookName = bookName;
	}

	public String getBookAuthor() {
		return bookAuthor;
	}

	public void setBookAuthor(String bookAuthor) {
		this.bookAuthor = bookAuthor;
	}
}
```


#### 4. IBookDao.java

```java
import java.util.List;
import com.priest.book.model.BookVO;

public interface IBookDao {
	public void saveOrUpdateBook(BookVO book);

	public List<BookVO> getBooks();

	public BookVO getBookById(int id);

	public void deleteBookById(int id);
}
```


#### 5. BookDaoImpl.java

```java
import java.util.List;
import org.hibernate.SQLQuery;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.object.SqlQuery;
import org.springframework.stereotype.Repository;
import com.priest.book.model.BookVO;

@Repository
public class BookDaoImpl implements IBookDao {

	@Autowired
	private SessionFactory sessionFactory;

	@Override
	public void saveOrUpdateBook(BookVO book) {
		Session session = sessionFactory.getCurrentSession();
		session.saveOrUpdate(book);
	}

	@Override
	public List<BookVO> getBooks() {
		Session session = sessionFactory.getCurrentSession();

		SQLQuery query = session.createSQLQuery("select * from book").addEntity(BookVO.class);
		List<BookVO> list = query.list();

		//List<BookVO> list = session.createQuery(" from BookVO ").list();

		return list;
	}

	@Override
	public BookVO getBookById(int id) {
		Session session = sessionFactory.getCurrentSession();
		BookVO book = new BookVO();

//		SQLQuery query = session.createSQLQuery("select * from book where id = ?").addEntity(BookVO.class);
//		query.setInteger(0, id);
//		List<BookVO> result = query.list();
//		if(result != null && result.size() > 0) {
//			book = result.get(0);
//		}

		book = (BookVO) session.get(BookVO.class, id);

		return book;
	}

	@Override
	public void deleteBookById(int id) {
		Session session = sessionFactory.getCurrentSession();

//		SQLQuery query = session.createSQLQuery(" delete from book where id = ? ").addEntity(BookVO.class);
//		query.setInteger(0, id);
//		query.executeUpdate();

		BookVO book = (BookVO) session.get(BookVO.class, id);
		session.delete(book);
	}
}
```


#### 6. IBookService.java

```java
import java.util.List;
import com.priest.book.model.BookVO;

public interface IBookService {
	public void saveOrUpdateBook(BookVO book);

	public List<BookVO> getBooks();

	public BookVO getBookById(int id);

	public void deleteBookById(int id);
}
```


#### 7. BookServiceImpl.java

```java
import java.util.List;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.priest.book.dao.IBookDao;
import com.priest.book.model.BookVO;

@Service
@Transactional
public class BookServiceImpl implements IBookService {

	@Autowired
	private IBookDao bookDao;

	@Override
	public void saveOrUpdateBook(BookVO book) {
		bookDao.saveOrUpdateBook(book);
	}

	@Override
	public List<BookVO> getBooks() {
		return bookDao.getBooks();
	}

	@Override
	public BookVO getBookById(int id) {
		return bookDao.getBookById(id);
	}

	@Override
	public void deleteBookById(int id) {
		bookDao.deleteBookById(id);
	}
}
```


#### 8. BookController.java

```java
import java.util.List;
import javax.servlet.http.HttpServletRequest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.servlet.ModelAndView;
import com.priest.book.model.BookVO;
import com.priest.book.service.IBookService;

@Controller
@RequestMapping("/book")
public class BookController {

	@Autowired
	private IBookService bookService;

	@RequestMapping(value= {"/bookList","/"}, method = RequestMethod.GET)
	public ModelAndView listAllBooks(ModelAndView model) {
		List<BookVO> books = bookService.getBooks();

		model.addObject("bookList", books);
		model.setViewName("books");

		return model;
	}

	//另一种写法
//	@RequestMapping(value= {"/bookList","/"}, method = RequestMethod.GET)
//	public String listAllBooks(Model model) {
//		List<BookVO> books = bookService.getBooks();
//		
//		model.addAttribute("bookList", books);
//		
//		return "books";
//	}

	@RequestMapping(value = "/newBook", method = RequestMethod.GET)
	public ModelAndView newBook(ModelAndView model) {
		BookVO book = new BookVO();
		model.addObject("book", book);
		model.setViewName("addOrUpdateBook");
		return model;
	}

	@RequestMapping(value = "saveOrUpdateBook", method = RequestMethod.POST)
	public ModelAndView saveOrUpdateBook(@ModelAttribute BookVO book) {
		bookService.saveOrUpdateBook(book);
		return new ModelAndView("operationSuccess");
	}

	@RequestMapping(value = "modifyBook", method = RequestMethod.GET)
	public ModelAndView modifyBook(HttpServletRequest request) {
		int id = Integer.parseInt(request.getParameter("id"));
		BookVO book = bookService.getBookById(id);
		ModelAndView model = new ModelAndView("addOrUpdateBook");
		model.addObject("book", book);

		return model;
	}

	@RequestMapping(value = "deleteBook", method = RequestMethod.GET)
	public ModelAndView deleteBook(HttpServletRequest request) {
		int id = Integer.parseInt(request.getParameter("id"));
		bookService.deleteBookById(id);

		return new ModelAndView("operationSuccess");
	}
}
```


#### 9. books.jsp

```html
<%@ page language="java" contentType="text/html; charset=utf-8" pageEncoding="utf-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">

<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>

<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>书</title>
</head>
<body>
	<table border="1">
		<tr>
			<th>ID</th>
			<th>书名</th>
			<th>作者</th>
			<th>操作</th>
		</tr>
		<c:forEach items="${bookList}" var="book">
			<tr>
				<td>${book.id}</td>
				<td>${book.bookName}</td>
				<td>${book.bookAuthor}</td>
				<td><a href="<c:url value='/book/modifyBook?id=${book.id}' />">编辑</a>
					<a href="<c:url value='/book/deleteBook?id=${book.id}' />">删除</a>
				</td>
			</tr>
		</c:forEach>
	</table>
	<br />
	<a href="<c:url value='/book/newBook' />">添加图书</a>
</body>
</html>
```


#### 10. addOrUpdateBook.jsp

```html
<%@ page language="java" contentType="text/html; charset=utf-8" pageEncoding="utf-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">

<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form"%>

<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>新增/修改图书</title>
</head>
<body>
	<h3>新增/修改图书</h3>
	<form:form action="saveOrUpdateBook" method="post" modelAttribute="book">
		<table>
			<form:hidden path="id" />
			<tr>
				<td>书名：</td>
				<td><form:input path="bookName" /></td>
			</tr>
			<tr>
				<td>作者：</td>
				<td><form:input path="bookAuthor" /></td>
			</tr>
			<tr>
				<td><input type="submit" value="保存" /></td>
			</tr>
		</table>
	</form:form>
</body>
</html>
```


#### 11. operationSuccess.jsp

```html
<%@ page language="java" contentType="text/html; charset=utf-8" pageEncoding="utf-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">

<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>

<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>书</title>
</head>
<body>
	操作成功！<br/>
	<a href="<c:url value='/book/bookList' />">返回列表</a>
</body>
</html>
```

---

## 运行结果

新打开的页面：

![列表页.png](https://i.loli.net/2018/11/07/5be2f17d6247f.png)

操作的页面：

![操作页.png](https://i.loli.net/2018/11/07/5be2f17ce1c60.png)

操作完成：

![操作完成.png](https://i.loli.net/2018/11/07/5be2f17cc8d6d.png)

返回列表：

![返回列表.png](https://i.loli.net/2018/11/07/5be2f17ceea24.png)

---

## 遇到的问题

#### 0. jar包使用异常

第一个最头大的问题当属引入了 jar 包，类中引用一直提示找不到的问题，在我[上一篇博客]({{ site.url }}/编程工具/2017/11/26/Eclipse通过Maven引入jar包但类中依旧找不到的问题/)中讲了这个问题

#### 1. jstl 获取对象属性

在 jsp 中本来可以使用 **对象.属性** 的方式取得内容，就像这样：

```html
<c:forEach items="${bookList}" var="book">
	<tr>
		<td>${book.id}</td>
		<td>${book.bookName}</td>
		<td>${book.bookAuthor}</td>
	</tr>
</c:forEach>
```

在写代码的过程中，由于一个地方的失误：

![hibernate配置model错误.png](https://i.loli.net/2018/11/07/5be2f17ce6f76.png)

把 model 的包路径写错了，导致：

![映射失败外加绑定失败时返回的是object对象.png](https://i.loli.net/2018/11/07/5be2f17de57b6.png)

查到的对象不能绑定为 BookVO，而是 Object，这样的话在 jstl 中使用 对象.属性 的时候就报错了

虽然是自己的一个小失误，除了配置中修改错误，映射正确的 VO 类外，这时候还可以使用下标的方式取的内容：

```html
<c:forEach items="${bookList}" var="book">
	<tr>
		<td>${book[0]}</td>
		<td>${book[1]}</td>
		<td>${book[2]}</td>
	</tr>
</c:forEach>
```

#### 2. 乱码

项目中有几个地方要注意编码：

* MySQL 数据库的配置

* .xml 配置文件中数据源链接

* jsp 页面的编码

最后参考学习，在 web.xml 中添加了 spring 的编码过滤器：

```xml
<filter>
	<filter-name>SpringEncodingFilter</filter-name>
	<filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
	<init-param>
		<param-name>encoding</param-name>
		<param-value>UTF-8</param-value>
	</init-param>
	<init-param>
		<param-name>forceEncoding</param-name>
		<param-value>true</param-value>
	</init-param>
</filter>
<filter-mapping>
	<filter-name>SpringEncodingFilter</filter-name>
	<url-pattern>/*</url-pattern>
</filter-mapping>
```

#### 3. 主键自增

一开始只在代码里给 VO 类注解了 ID 是自增，调用保存的时候报错：

```
java.sql.SQLException: Field 'id' doesn't have a default value
```

查询后发现是数据库里忘记给主键设置自增导致的

---

## 最后

参考学习：

[http://javawebtutor.com/articles/spring/spring-mvc-hibernate-crud-example.php](http://javawebtutor.com/articles/spring/spring-mvc-hibernate-crud-example.php)

[https://howtodoinjava.com/spring/spring-mvc/spring-mvc-hello-world-example/](https://howtodoinjava.com/spring/spring-mvc/spring-mvc-hello-world-example/)
