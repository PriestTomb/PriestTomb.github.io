---
layout: post
title: Nginx 代理域名地址时的 DNS 缓存问题
date: 2020-05-17
categories:
- 技术
tags: [Nginx]
status: publish
type: post
published: true
---

# 问题描述

接[上篇文章](https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2020/05/05/nginx-400-error-about-host/)中提到的 Nginx 解析域名地址的问题，用一句话描述就是“proxy_pass 中如果配置的是域名地址，Nginx 只有在 start / restart / reload 时，才会连接一次域名服务器解析域名，缓存解析的结果，后续则**不会根据解析结果的 TTL 进行自动更新**”，如果遇到了域名地址配置有多个 IP ，且还在动态变化，那就会出现 Nginx 把请求转发到一个过期的 IP 地址的情况，连接超时的报错日志类似这样：

```
[error] 20472#0: *455027 upstream timed out (110: Connection timed out) while connecting to upstream, client: 192.168.1.10, server: localhost, request: "GET /xxx/xxx HTTP/1.1", upstream: "http://103.95.221.193:80/xxx/xxx", host: "192.168.1.11:12345"
```

这个说法在[官方的一篇 2016 年的博客](https://www.nginx.com/blog/dns-service-discovery-nginx-plus/)中有提到：

> * NGINX caches the DNS records until the next restart or configuration reload, ignoring the records’ TTL values.

除此之外，除了一些分析源码的网络文章，暂时还没有找到其他的官方文档中说到这个细节

---

# 问题解决

### 1. 利用 upstream 设置负载均衡算法及失败次数

```
upstream backends {
    least_conn;
    server backends.example.com:8080 max_fails=3;
}

server {
    location / {
        proxy_pass http://backends;
    }
}
```

在 upstream 中可以对上游的服务器进行更详细的设置，解决 DNS 缓存的问题可以在 upstream 中指定需要的负载均衡算法，比如 `least_conn`，并指定 `max_fails`，以实现调用失败 N 次之后判定该服务异常，暂停转发该服务

注：

这个配置的示例是官方博客中的，看到这个配置时觉得有点奇怪，自己进行了模拟测试，测试的方案是在 hosts 文件中配置一个模拟的域名与三个 IP 地址，其中两个 IP 是正确的，另一个是内网不存在的 IP，测试的结果就是 Nginx 始终会将请求转发到那个错误的 IP 去，日志中一直能看到超时的报错，配置的 `max_fails` 仿佛没有任何作用（有补充配置了 `fail_timeout` ，也尝试配置了 `proxy_next_upstream` 、 `proxy_next_upstream_timeout` 和 `proxy_next_upstream_tries`）

不清楚用 hosts 配置的方式是不是必然会出现这样的情况，因为目前没条件测试真正想要的场景，所以不敢说博客中的这种配置是错的【如果以后碰巧有条件能测试验证，再回头来更新

最初学习 Nginx 的时候测试过 `max_fails` 这个配置，当时在 upstream 里配置的都是一些 IP 地址的上游服务。再次按 IP 地址进行测试，在 upstream 中配置两个正确的 IP 地址 和一个错误的 IP 地址，发现这样的配置就是能生效的，失败一定次数之后（实际失败的次数比设置的 `max_fails` 多，不清楚什么原因），Nginx 在 `fail_timeout` 时间内就不再转发请求到那个错误的 IP

### 2. 定义 resolver 并设置变量

```
resolver 10.0.0.2 valid=10s;
resolver_timeout 5s;

server {
    location / {
        set $backend_servers backends.example.com;
        proxy_pass http://$backend_servers:8080;
    }
}
```

`resolver` 的配置详情可看[官方文档](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver)，示例的配置是指定 DNS 服务器 10.0.0.2，指定 DNS 解析的有效时间为 10 秒，按博客[《Nginx动态解析upstream域名》](http://blog.sina.com.cn/s/blog_4da051a60102x1wb.html)中博主的测试，不是说 Nginx 每过 10 秒会自己重新调一次 DNS 解析，而是有请求转发时才检验一次有效期是否过期

不配置 `valid` 选项时，V1.1.9 之后的 Nginx 默认会使用 DNS 解析结果中的 TTL

在 `proxy_pass` 中使用变量，带来的作用就是在 TTL 过期时能再次调用 DNS 解析，从而解决一直使用缓存结果的问题

这大概是目前官方原版唯一解决 DNS 缓存的解决方案了，带来的弊端也如[《Nginx动态解析upstream域名》](http://blog.sina.com.cn/s/blog_4da051a60102x1wb.html)的博主所说，不能使用 upstream 模块特有的相关配置

Nginx Plus 版有更好的配置解决这些问题，另外使用 Lua 插件或许也能更完美的解决这个问题，暂时就没什么研究了
