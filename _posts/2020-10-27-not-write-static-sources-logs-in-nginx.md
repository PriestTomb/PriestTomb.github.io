---
layout: post
title: Nginx 过滤静态资源文件的访问日志
date: 2020-10-27
categories:
- 技术
tags: [Nginx]
status: publish
type: post
published: true
---

# 凌乱的日志

日常使用的 Nginx 大都既做静态资源服务器，也做反向代理服务器，尤其有些时候考虑到跨域问题，会对静态资源和后端接口使用同一个监听端口，如果不做一下过滤处理，会在 access_log 中看到大量的例如 js、css、jpg 等静态资源的请求，比较影响查看后端接口调用的日志

本来没有很在意这个东西，不过在浏览一篇关于 Nginx 优化的文章时，发现了一种用 map 定义一个是否写日志的参数的方法，结合最近使用 map 做动态的跨域配置，索性也是学习及记录一下 map 的另一个使用场景

---

# 使用 map 过滤访问静态资源文件的日志

```
http {
    log_format  main  '$remote_addr [$time_local] $request $status '
                      'uct="$upstream_connect_time" rt="$request_time"';

    map $uri $not_static {
        default 1;
        ~^(.*\.(gif|jpg|jpeg|png|bmp|swf|js|css|woff|ttf)$)  0;
    }

    server {
        listen		23456;
        server_name	localhost;
        access_log	logs/test.log main if=$not_static;
    }
}
```

解释说明：

* 自定义一个 log_format，标识为 main

* 对请求中的 uri 做匹配，如果是以 gif、jpg、css、js 等作为结尾的资源，则  `$not_static` 为0，否则为1

* 对访问23456端口的请求，access\_log 指定使用标识为 main 的自定义日志格式，且仅当 `$not_static` 为1时才记录日志，关于 if 参数，可参考[官方文档](http://nginx.org/en/docs/http/ngx_http_log_module.html#access_log)

* 有一点需要注意，access\_log 中使用 if 参数时，必须显式指定一个 log\_format，否则会报错：`nginx: [emerg] unknown log format "if=$not_static"`

---

# 另一种动静分离日志写法

```
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|js|css|woff|ttf)$ {
    #access_log off; #不输出访问静态资源的日志
    access_log logs/static_resources.log;
}
```
