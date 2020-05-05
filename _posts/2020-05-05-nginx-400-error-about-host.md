---
layout: post
title: Nginx 转发域名接口 400 Bad Request 报错
date: 2020-05-05
categories:
- 技术
tags: [Nginx]
status: publish
type: post
published: true
---

# 问题描述

在内网服务器上部署了一个服务，其中有某些接口是外网的**域名地址**，所以找了内网里一台能访问外网的服务器，通过 Nginx 进行转发，但是出现了通过 Nginx 转发出去的请求直接返回 400 Bad Request，查了很久才知道问题在哪里，也算是 Nginx 配置的不熟跟 HTTP 协议的不熟导致的吧

![拓扑图](/images/blog_img/20200505/拓扑.png)

简单来说，拓扑结构如上图

Nginx 最初使用的简单配置如下

```
upstream test-interface {
    server abc.test.cn;
}

server {
    listen       12345;
    server_name  localhost;
    location / {
        proxy_pass http://test-interface;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

所出现的现象就是，从 192.168.1.10 或者 192.168.1.11 请求 192.168.1.11 的 12345 端口，都会返回 400

---

# 问题解决

搜索 Nginx 400 报错，很容易搜出一大堆的文章来，其中有说 cookie 过长、Host 为空、报文格式出错等等，最初先参考一些设置，给请求头加上了 Host ，比如 `proxy_set_header Host $host;` 或者 `proxy_set_header Host $http_host;` ，但还是报错，于是只好抓包进行查看，包括经过 Nginx 12345 端口转发的请求，和直接在 Nginx 服务器上调用外网接口的请求

在比对了之后发现，能调用成功的请求头中， Host 就是接口的域名地址，而调用失败的请求头中，该字段都是 IP 或者其他什么东西

![成功的请求](/images/blog_img/20200505/正确的请求包.png)

先说解决的办法，有两种，以文章开头的配置内容为例，进行修改：

### 1. 设置静态的 Host 值

```
upstream test-interface {
    server abc.test.cn;
}

server {
    listen       12345;
    server_name  localhost;
    location / {
        proxy_pass http://test-interface;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host abc.test.cn;
    }
}
```

### 2. 不使用 upstream ，直接 proxy pass 域名地址

```
server {
    listen       12345;
    server_name  localhost;
    location / {
        proxy_pass http://abc.test.cn;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

这两种配置都可以成功转发，因为转发的是域名地址，其实还涉及到 Nginx 在 DNS 解析方面的一个配置问题，暂且挖个坑，回头学习学习再来填上

---

# 再折腾折腾

多测试了一些配置并抓包看了一下请求头中的 Host ，比如

### 0. `upstream` + `proxy_set_header Host $http_host;`

```
upstream test-interface {
    server abc.test.cn;
}

server {
    listen       12345;
    server_name  localhost;
    location / {
        proxy_pass http://test-interface;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $http_host;
    }
}
```

Host 为 Nginx 服务器的 IP + 监听端口号：192.168.1.11:12345 ，请求失败

### 1. `upstream` + `proxy_set_header Host $host;`

```
upstream test-interface {
    server abc.test.cn;
}

server {
    listen       12345;
    server_name  localhost;
    location / {
        proxy_pass http://test-interface;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
    }
}
```

Host 为 Nginx 服务器的 IP ：192.168.1.11 ，请求失败

### 2. `upstream` + 无 `proxy_set_header Host`

```
upstream test-interface {
    server abc.test.cn;
}

server {
    listen       12345;
    server_name  localhost;
    location / {
        proxy_pass http://test-interface;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

不显式设置 Host 时， Nginx 会默认重新定义 Host 为 `$proxy_host` ， 抓包可以看到 Host 为 upstream 的名字：test-interface ，请求失败

### 3. 直接 proxy_pass 域名 + `proxy_set_header Host $host;`

```
server {
    listen       12345;
    server_name  localhost;
    location / {
        proxy_pass http://abc.test.cn;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
    }
}
```

Host 被显式指定为 $host ，如上会变成 Nginx 服务器的 IP ： 192.168.1.11 ，请求失败

### 4. 直接 proxy_pass 域名 + `proxy_set_header Host "";`

```
server {
    listen       12345;
    server_name  localhost;
    location / {
        proxy_pass http://abc.test.cn;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host "";
    }
}
```

Nginx 配置中显式将 Host 置为空，抓包中看不到请求头中有 Host ，请求失败

---

# 参考

[NGINX Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

[HTTP headers Host](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Host)

[400 Bad Request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/400)
