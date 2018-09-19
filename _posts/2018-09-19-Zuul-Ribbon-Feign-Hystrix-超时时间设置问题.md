---
layout: post
title: Zuul、Ribbon、Feign、Hystrix使用时的超时时间(timeout)设置问题
date: 2018-09-19
categories: [分布式]
tags: [SpringCloud, Zuul, Ribbon, Feign, Hystrix]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

## 写在前面

因为测试 Feign + Hystrix 搭配模式下的降级(fallback)超时时间自定义问题，算是踩了个坑，然后就顺便查+测试了下 Zuul、Ribbon + Hystrix 模式下分别怎么设置

测试这些东西费了不少力气，因为这几个模块要么搭配使用、要么有内部依赖别的模块、要么对其他模块做了封装，这个配置项就变得千奇百怪，而且网上的东西，一直觉得有个很"严重"的问题，就是版本不明，版本号都不一样，解决方案或者说配置方式可能完全不同，而很多的文章中也没有提及他们用的是哪个版本，搞得我是晕头转向（毕竟我不是这些服务模块的开发者或者长期的使用者，不是非常了解这些东西的版本演进过程）

所以这里是查了不少的资料，测试通过了一些方案，也算是自己总结记录一下

**注意！**

**这里都是基于有 Eureka 做服务中心为前提的**

---

## 工具

* Eclipse Oxygen

* **Spring Boot 2.0.5.RELEASE**

* **Spring Cloud Finchley.SR1**

* Eureka 1.9.3

* Zuul 1.3.1

* Ribbon 2.2.5

* Feign 9.5.1

* Hystrix 1.5.12

---

## Feign + Hystrix

这个栗子的源码看[这里](https://github.com/PriestTomb/Spring-Cloud-Demo/tree/master/Eureka%2BFeign%2BHystrix)

#### 0. 默认基本配置

最基本的配置，是 Hystrix 自己的一长串配置：`hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds`，但在 Feign 模块中，单独设置这个超时时间不行，还要额外设置 Ribbon 的超时时间，比如：

```yml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 5000

ribbon:
  ReadTimeout: 5000
  ConnectTimeout: 5000
```

关于 Hystrix 的配置，这里有[官方的说明](https://github.com/Netflix/Hystrix/wiki/Configuration#execution.isolation.thread.timeoutInMilliseconds)：

| --- | :-- |
| Default Value | 1000 |
| Default Property | hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds |
| Instance Property | hystrix.command.*HystrixCommandKey*.execution.isolation.thread.timeoutInMilliseconds |
| How to Set Instance Default | HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(int value) |

可以看到实例配置中，替代 default 的，是 HystrixCommandKey，这个值在下面会说到

#### 1. 不同实例分别配置

如果更进一步，想把超时时间细分到不同的 service 实例上也可以实现，比如：

```java
@FeignClient(
	value = "hello-service",
	fallback = MyFeignClientHystric.class)
public interface MyFeignClient {
	@RequestMapping("/hello")
	String sayHelloByFeign();

	@RequestMapping("/why")
	String sayWhyByFeign();
}
```

```yml
hystrix:
  command:
    "MyFeignClient#sayWhyByFeign()":
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 9000
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 2000

ribbon:
  ReadTimeout: 5000
  ConnectTimeout: 5000
```

这种写法是把默认的超时时间改成2秒，而把另外一个自定义的 Feign 客户端中的某方法超时时间定成9秒(格式是`类名#方法名()`，如果方法有入参，也要把入参的类型拼上)，这里的 `MyFeignClient#sayWhyByFeign()` 就代表了上面说到的 `commandKey`，而这种写法，则是 Feign 模块中特殊的：

> ryanjbaxter commented on 26 Jul
>
> If you had a Feign client called MyClient and it had a method called search that took in a single String parameter than you would use the following property
hystrix.command.MyClient#search(String).execution.isolation.thread.timeoutInMilliseconds

看 [issue](https://github.com/spring-cloud/spring-cloud-netflix/issues/1864) 里 Spring Cloud 的官方人员的说法，这种格式是他们进行的封装，所以我们要设置，就只能这么写

---

## Ribbon + Hystrix

这个栗子的源码看[这里](https://github.com/PriestTomb/Spring-Cloud-Demo/tree/master/Eureka%2BRibbon%2BHystrix)

#### 0. 默认基本配置

在使用 Ribbon 时，只需要配置 Hystrix 的超时时间就可以生效，不需要额外配置 Ribbon 的超时时间，比如：

```yml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 9000
```

#### 1. 不同实例分别配置

想细分服务超时时间时：

如果是同一个服务实例下的不同接口，想使用不同的超时时间，可以把 `@HystrixCommand` 中的 `commandKey` 定义成不同的值，然后在 yml 中分别设置

```java
@HystrixCommand(
	commandKey = "helloService-sayHello",
	fallbackMethod = "sayHelloDefault")
public String sayHelloByRibbon() {
	return restTemplate.getForObject("http://HELLO-SERVICE/hello", String.class);
}

public String sayHelloDefault() {
	return "hello service error, this is default say hello method";
}

@HystrixCommand(
	commandKey = "helloService-sayWhy",
	fallbackMethod = "sayWhyDefault")
public String sayWhyByRibbon() {
	return restTemplate.getForObject("http://HELLO-SERVICE/why", String.class);
}

public String sayWhyDefault() {
	return "hello service error, this is default say why method";
}
```

```yml
hystrix:
  command:
    helloService-sayWhy:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 5000
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 1500
```

如果想统一设置同一个服务实例中各方法的超时时间，经测试，可以把不同方法上的 `commandKey` 设置成相同的值，这样在 yml 中对该 key 做超时配置就能同时生效了：

```java
@HystrixCommand(
	commandKey = "helloService",
	fallbackMethod = "sayHelloDefault")
public String sayHelloByRibbon() {
	return restTemplate.getForObject("http://HELLO-SERVICE/hello", String.class);
}

public String sayHelloDefault() {
	return "hello service error, this is default say hello method";
}

@HystrixCommand(
	commandKey = "helloService",
	fallbackMethod = "sayWhyDefault")
public String sayWhyByRibbon() {
	return restTemplate.getForObject("http://HELLO-SERVICE/why", String.class);
}

public String sayWhyDefault() {
	return "hello service error, this is default say why method";
}
```

```yml
hystrix:
  command:
    helloService:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 5000
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 1000
```


---

## Zuul

这个栗子的源码看[这里](https://github.com/PriestTomb/Spring-Cloud-Demo/tree/master/Eureka%2BZuul)，Zuul 中的降级是用了 `FallbackProvider`，简单的使用可以看我源码中的 `HelloFallbackProvider.java` 和 `HiFallbackProvider.java`，我也是参考了[官方的文档说明和例子](https://cloud.spring.io/spring-cloud-netflix/multi/multi__router_and_filter_zuul.html#hystrix-fallbacks-for-routes)

#### 0. 默认基本配置

zuul 中配置超时时间，据[官方的介绍](http://cloud.spring.io/spring-cloud-netflix/single/spring-cloud-netflix.html#_zuul_timeouts)，分两种情况：

* 用 serviceId 进行路由时，使用 `ribbon.ReadTimeout` 和 `ribbon.SocketTimeout` 设置

* 用指定 url 进行路由时，使用 `zuul.host.connect-timeout-millis` 和  `zuul.host.socket-timeout-millis` 设置

因为我的代码中是用 serviceId 的方式，所以参考了第一种配置，比如：

```yml
zuul:
  routes:
    helloService:
      path: /hello-service/**
      serviceId: hello-service
    hiService:
      path: /hi-service/**
      serviceId: hi-service

ribbon:
  ConnectTimeout: 5000
  ReadTimeout: 5000
```

#### 1. 不同实例分别配置

Ribbon 的配置项还可以加一个 `ClientName` 为前缀（这个方法的出处在官方的 [wiki](https://github.com/Netflix/ribbon/wiki/Programmers-Guide#client-configuration-options)），区分不同客户端下的配置，这个 `ClientName` 我是直接用了 serviceId，测试了正常，但还可以用或者说应该用什么值，这个我还没有找到官方的说明。。

```yml
zuul:
  routes:
    helloService:
      path: /hello-service/**
      serviceId: hello-service
    hiService:
      path: /hi-service/**
      serviceId: hi-service

hello-service:
  ribbon:
    ConnectTimeout: 5000
    ReadTimeout: 5000

hi-service:
  ribbon:
    ConnectTimeout: 500
    ReadTimeout: 500

hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
```

另外还做了测试，如果同时配置了 Ribbon 和 Hystrix 的超时时间，则以最小的为准，比如上述的配置中，如果 hi-service 的接口调用超过了 0.5 秒，则就会触发超时

---

## 总结

目前的学习和测试结果来看：

* 单纯的 Ribbon + Hystrix 搭配使用时，配置是最灵活的，两者没有相互干涉，可以自由定义 `commandKey` 来实现超时时间的配置

* Feign + Hystrix 搭配时，由于 Feign 封装了 Hystrix 所需的 `commandKey`，我们不能自定义，所以同一个 FeignClient 下的服务接口不能方便的统一配置，如果有相应的业务需求，或许只能对每个特殊的接口方法做独立的超时配置（找到新方法的话再回来更新）

* Zuul + Hystrix 搭配时，和上述的情况相反，能对不同服务实例做不同的超时配置，但不能再细化到服务下的具体接口方法
