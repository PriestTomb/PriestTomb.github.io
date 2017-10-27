---
layout: post
title: Java-玩玩WebService
date: 2017-10-27
categories:
- 后端技术
tags: [Java, WebService]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

## 写在前面

工作的项目是个门户项目，调了七七八八的外系统几百个接口，但从没向外提供过接口，所以一直都是写 WebService 客户端代码，还不清楚服务端到底怎么个玩法

这就抽空学习下 WebService 的服务端、客户端都有哪些写法

---

## 工具

* Eclipse Oxygen
* JDK 1.8
* axis 1.4
* axis2 1.7.6
* soapUI 3.6

---

## 服务端

#### 1. 新建 Dinamic Web Project

![新建服务端工程](http://oxujjb0ls.bkt.clouddn.com/image/20171027/%E6%96%B0%E5%BB%BAwebservice%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%B7%A5%E7%A8%8B.png)

#### 2. 新建服务端测试类

这里使用了 JAX-WS 发布服务端，注意要给类加上 @WebService 注解

```java
package com.priest.service;

import java.util.Date;
import javax.jws.WebService;
import javax.xml.ws.Endpoint;

@WebService
public class TestServer {

	public String testMethod(String inParam) {
		System.out.println("服务端收到：" + inParam);
		return "服务端于" + new Date() + "收到了客户端发送的：" + inParam;
	}

	public static void main(String[] args) {
		Endpoint.publish("http://localhost:8088/service/TestServer", new TestServer());
		System.out.println("服务端启动完毕");
	}

}
```

#### 3. 启动工程

![启动服务端](http://oxujjb0ls.bkt.clouddn.com/image/20171027/%E5%90%AF%E5%8A%A8%E6%9C%8D%E5%8A%A1%E7%AB%AF.png)

#### 4. 测试服务

###### 4.1 浏览器查看

![浏览器查看服务](http://oxujjb0ls.bkt.clouddn.com/image/20171027/%E6%B5%8F%E8%A7%88%E5%99%A8%E6%B5%8B%E8%AF%95.png)

###### 4.2 soapUI测试

![SOAPUI测试服务](http://oxujjb0ls.bkt.clouddn.com/image/20171027/soapUI%E6%B5%8B%E8%AF%95.png)


服务端完成，接下来以各种方式写客户端代码

---

## 客户端1

#### 1. 使用 wsimport 生成客户端

使用 java 自带的 wsimport 命令：

![wsimport生成客户端](http://oxujjb0ls.bkt.clouddn.com/image/20171027/wsimport%E7%94%9F%E6%88%90%E5%AE%A2%E6%88%B7%E7%AB%AF.png)

```
wsimport -s F:\workspace\eclipse_workspace\WebServiceClientDemo\src -p com.priest.client -keep http://localhost:8088/service/TestServer?wsdl
```


    -s 指定源码目录
    -p 指定生成的包路径
    -keep 保留生成的文件

测试了下，这个 `-keep` 加不加好像没什么区别？

#### 2. 查看生成的客户端类

![生成的客户端类](http://oxujjb0ls.bkt.clouddn.com/image/20171027/%E7%94%9F%E6%88%90%E7%9A%84%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%B1%BB.png)

#### 3. 写客户端测试类

```java
package com.priest.client;

public class TestClient1 {

	public static void main(String[] args) {
		TestServerService service = new TestServerService();
		TestServer ts = service.getTestServerPort();

		String result = ts.testMethod("客户端1测试");
		System.out.println("客户端1测试的结果：\n" + result);
	}

}
```

#### 4. 执行调用

![客户端1测试](http://oxujjb0ls.bkt.clouddn.com/image/20171027/%E5%AE%A2%E6%88%B7%E7%AB%AF1%E6%B5%8B%E8%AF%95.png)

---

## 客户端2

#### 1. 使用 wsimport 生成客户端

同客户端1，这里需要借用 wsimport 生成的客户端类

#### 2. 写客户端测试类

```java
package com.priest.client;

import java.net.URL;

import javax.xml.namespace.QName;
import javax.xml.ws.Service;

public class TestClient2 {
	public static void main(String[] args) {
		try {
			URL url = new URL("http://localhost:8088/service/TestServer?wsdl");
			QName qname = new QName("http://service.priest.com/", "TestServerService");
			Service service = Service.create(url, qname);
			TestServer ts = service.getPort(TestServer.class);
			String result = ts.testMethod("客户端2测试");
			System.out.println("客户端2测试的结果：\n" + result);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

#### 3. 执行调用

![客户端2测试](http://oxujjb0ls.bkt.clouddn.com/image/20171027/%E5%AE%A2%E6%88%B7%E7%AB%AF2%E6%B5%8B%E8%AF%95.png)

---

## 客户端3

#### 1. 引入 axis 1.4 的 jar 包

从[官网](https://mirrors.tuna.tsinghua.edu.cn/apache/axis/axis/java/1.4/)下载 axis_1.4.jar

项目中需要引入的 jar 包至少是这些：
* axis.jar
* jaxrpc.jar
* commons-logging-1.0.4.jar
* commons-discovery-0.2.jar
* wsdl4j-1.5.1.jar


#### 2. 写客户端测试类

```java
package com.priest.client2;

import java.net.URL;

import javax.xml.namespace.QName;

import org.apache.axis.client.Call;
import org.apache.axis.client.Service;

public class TestClient3 {

	public static void main(String[] args) {
		try {
			//服务地址
			URL url = new URL("http://localhost:8088/service/TestServer");
			//命名空间
			String nameSpace = "http://service.priest.com/";
			//方法名
			String method = "testMethod";

			Service service = new Service();
			Call call = (Call) service.createCall();

			call.setOperationName(new QName(nameSpace, method));
			call.setTargetEndpointAddress(url);
			call.setTimeout(3000);
			call.addParameter("arg0", org.apache.axis.Constants.XSD_STRING, javax.xml.rpc.ParameterMode.IN);
			call.setReturnType(org.apache.axis.Constants.XSD_STRING);

			String result = (String) call.invoke(new Object[] {"客户端3测试"});
			System.out.println("客户端3测试结果：\n" + result);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}

```

#### 3. 执行调用

![客户端3测试](http://oxujjb0ls.bkt.clouddn.com/image/20171027/%E5%AE%A2%E6%88%B7%E7%AB%AF3%E6%B5%8B%E8%AF%95.png)

有个警告，是：

```
Unable to find required classes (javax.activation.DataHandler and javax.mail.internet.MimeMultipart). Attachment support is disabled.
```

需要引入 `mail.jar` 和 `activation.jar`，不过不引也不会对 webservice 调用有影响

---

## 客户端4

#### 1. 使用 axis.jar 生成客户端

详细步骤参考这篇博客

#### 2. 查看生成的客户端类

![axis生成的客户端类](http://oxujjb0ls.bkt.clouddn.com/axis%E7%94%9F%E6%88%90%E7%9A%84%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%B1%BB.png)

#### 3. 写客户端测试类

```java
package com.priest.client4;

public class TestClient4 {

	public static void main(String[] args) {

		try {
			//重新指定url是为了防止本地客户端代码是旧的，而对方迁了服务地址
			String url = "http://localhost:8088/service/TestServer";
			TestServerServiceLocator locator = new TestServerServiceLocator();
			locator.setTestServerPortEndpointAddress(url);
			TestServerPortBindingStub stub = (TestServerPortBindingStub) locator.getTestServerPort();
			stub.setTimeout(3000);
			String result = stub.testMethod("客户端4测试");
			System.out.println("客户端4测试结果：\n" + result);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

}
```

#### 4. 执行调用

![客户端4测试](http://oxujjb0ls.bkt.clouddn.com/%E5%AE%A2%E6%88%B7%E7%AB%AF4%E6%B5%8B%E8%AF%95.png)

报错内容和上面 客户端3 章节里的一样，因为缺了两个 jar 包

---

## 客户端5

#### 1. 引入 axis2 的 jar 包

#### 2. 写客户端测试类


#### 3. 执行调用
