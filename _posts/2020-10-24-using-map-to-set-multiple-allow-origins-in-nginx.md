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

# 版本说明

Nginx 1.14.0

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
    "~^(https?://localhost(:[0-9]+)?)$" $1;
    "~^(https?://127.0.0.1(:[0-9]+)?)$" $1;
    "~^(https?://172.10(.[\d]+){2}(:[0-9]+)?)$" $1;
    "~^(https?://192.168(.[\d]+){2}(:[0-9]+)?)$" $1;
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

* `$allow_origin` 是可以自定义的变量名，用于接收 map 返回的值

* `$1` 是 Nginx 对 PCRE 中关于[后向引用](https://www.php.net/manual/zh/regexp.reference.back-references.php)和[子组](https://www.php.net/manual/zh/regexp.reference.subpatterns.php)的兼容，用于获取匹配字符串的整个部分，并返回给 `$allow_origin`

---

# 关于 `$1` 的补充

Nginx 关于 server_name 的[文档](https://nginx.org/en/docs/http/server_names.html)中有提到关于正则匹配的部分兼容 PCRE，所以 `$1` 的含义可以参考 PCRE 中关于子组的说明：

> 子组通过圆括号分隔界定，并且它们可以嵌套。 将一个模式中的一部分标记为子组(子模式)主要是来做两件事情：
>
> * 将可选分支局部化。比如，模式 `cat(arcat|erpillar|)` 匹配 ”cat”， “cataract”， “caterpillar” 中的一个，如果没有圆括号的话，它匹配的则是 ”cataract”， “erpillar” 以及空字符串。
>
> * 将子组设定为捕获子组(向上面定义的). 当整个模式匹配后， 目标字符串中匹配子组的部分将会通过 pcre_exec() 的 ovector 参数回传给调用者。 左括号从左至右出现的次序就是对应子组的下标(从 1 开始)， 可以通过这些下标数字来获取捕获子模式匹配结果。
>
> 比如，如果字符串 ”the red king” 使用模式 `((red|white) (king|queen))` 进行匹配， 模式匹配到的结果是 array(“red king”, ”red king”, “red”, “king”) 的形式， ***其中第 0 个元素是整个模式匹配的结果，后面的三个元素依次为三个子组匹配的结果。 它们的下表分别为 1，2，3***。

在 PCRE 中使用 `$0` 来表示整个结果，但在 Nginx 中则是 `$1`。

暂时没找到官方文档中的说明，只在一个 6 年前的问题 [nginx \[emerg\] unknown “0” variable](https://serverfault.com/questions/629158/nginx-emerg-unknown-0-variable) 里看到有两位老哥的回复有提到：

> nginx does not have the $0 variable
>
> Nginx rewrite regexp do not have a $0 variable

经过简单的测试，1.14.0 版本下确实是用 `$1` 获取到整个匹配结果。
