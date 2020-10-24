---
layout: post
title: 使用 map 实现 Nginx 允许多个域名跨域
date: 2020-10-24
categories:
- 技术
tags: [Nginx, 跨域]
status: publish
type: post
published: true
---

# 常见的 Nginx 配置允许跨域

```
server {
    listen       11111;
    server_name  localhost;

    location ~ /xxx/xx {
        if ($request_method = 'OPTIONS') {
            return 204;
        }
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
        add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';
        proxy_pass http://1.2.3.4:5678;
    }
}
```

指定 `Access-Control-Allow-Origin` 为 '*' ，即为最简单暴力的允许所有访问跨域

---

# 允许 Cookie

有些场景下需要使用 Cookie，这时 Nginx 需要加一句 `add_header Access-Control-Allow-Credentials 'true';`，但此时会发现浏览器报错，说该参数为 true 时，allow origin 不能设置为 '*'，如果手动指定了多个域名，那同样会被浏览器提示错误，说 allow origin 不能设置多个，这些是协议层面的限制

---

# 使用 map

在 Nginx 中可以使用 map 得到一个自定义变量，简单的使用可以参考[官方文档](https://nginx.org/en/docs/http/ngx_http_map_module.html)，在上面提到的场景中，可以对请求中的 origin 做一个过滤处理，把符合要求的请求域名放到一个变量中，在设置 allow origin 时使用该变量就能实现一个动态的、多个的允许跨域域名

一个示例配置如下：

```
map $http_origin $allow_origin {
    default "";
    "~^(https?://localhost(:[0-9]+)?)" $1;
    "~^(https?://127.0.0.1(:[0-9]+)?)" $1;
    "~^(https?://172.10(.[\d]+){2}(:[0-9]+)?)" $1;
    "~^(https?://192.168(.[\d]+){2}(:[0-9]+)?)" $1;
}

server {
    listen       11111;
    server_name  localhost;

    location ~ /xxx/xx {
        if ($request_method = 'OPTIONS') {
            return 204;
        }
        add_header Access-Control-Allow-Origin $allow_origin;
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
        add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';
        add_header Access-Control-Allow-Credentials 'true';
        proxy_pass http://1.2.3.4:5678;
    }
}
```

**解释说明：**

* `$http_origin` 是 Nginx 的[内部变量](https://nginx.org/en/docs/http/ngx_http_core_module.html#var_http_)，用于获取请求头中的 origin

* `$allow_origin` 是可以自定义的变量名
