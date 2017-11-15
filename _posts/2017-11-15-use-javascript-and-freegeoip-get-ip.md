---
layout: post
title: 纯JavaScript获取IP(通过freegeoip)
date: 2017-11-15
categories:
- 乱七八糟的前端
tags: [JavaScript, 获取IP]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

## 写在前面

之前因为给博客添加"访问次数"功能，除了正常的刷新喜+1外，额外加了一点 IP 和访问时间的限制，就用到了纯 js 获取 ip 的功能

中文站点上的文章都是推荐的国内的一些老旧接口，比如 sohu 家的、 weibo 家的、 qq 家的，虽然很好用，但部到 github pages 上之后就。。因为 github pages 已经是 https 站点了，所以不能再用国内的那些旧的 http 接口

**写到这里我忽然想起来一件事，于是去测试了一下，发现。。把 sohu 的地址改成 https 也能正常访问。。**

![背锅](http://oxujjb0ls.bkt.clouddn.com/image/%E8%A1%A8%E6%83%85%E5%8C%85/%E8%BF%99%E4%B8%AA%E9%94%85%E6%88%91%E8%83%8C.jpg)

让我们先忽略这个问题。。继续。。

于是在 Stack Overflow 上搜了下 [How to get client's IP address using JavaScript only?](https://stackoverflow.com/questions/391979/how-to-get-clients-ip-address-using-javascript-only)，找到一些国外的网站，能通过 js 调用，而且是 https 站点的，就找到 [freegeoip](https://freegeoip.net) 来用了

## 代码

我这里主要用 jQuery 的 ajax 请求来取 ip，格式使用 json 和 xml 两种

#### 1. json 格式

```javascript
var ip;
var options = {
  type: 'GET',
  dataType: "json",
  //async: false, //jquery3中可以直接使用回调函数，不用再指定async
  url: "https://freegeoip.net/json/?callback=?"
};
$.ajax(options)
.done(function(data, textStatus, jqXHR) {
  if(textStatus == "success") {
    try {
      ip = data.ip;
    } catch (e) {
      if(window.console){
        console.log(e.message);
      }
    }
  }
});
```

#### 2. xml 格式

```javascript
var ip;
var options = {
  type: 'GET',
  dataType: "xml",
  //async: false, //jquery3中可以直接使用回调函数，不用再指定async
  url: "https://freegeoip.net/xml/"
};
$.ajax(options)
.done(function(data, textStatus, jqXHR) {
  if(textStatus == "success") {
    try {
      var $resp = $(data).children('response');
      ip = $resp.children('ip').text();
    } catch (e) {
      if(window.console){
        console.log(e.message);
      }
    }
  }
});
```
