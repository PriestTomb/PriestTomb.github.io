---
layout: post
title: 用LeanCloud和sessionStorage实现热门文章功能
date: 2018-09-08
categories:
- 日常
tags: [jekyll博客]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

## 闲来无事

上周闲着无聊，给博客加了“**热门文章**”功能（其实并不热门），在博客右边的开荒了一片空地用来显示

整体实现依旧很简单，因为 LeanCloud 上有单独的一张文章表，我要用的数据刚好都在：文章名称、文章链接、文章访问次数，于是就从这张表按访问次数逆序取前5条，稍写下样式，拼接<li\>展示，搞定

![结果展示.jpg](https://i.loli.net/2018/11/07/5be2f3e97ef33.jpg)

不过正是因为占了博客的右侧空地，这也导致了目前没有好的小屏、移动端的展示设计，就只好在小屏、移动端的情况下隐藏了这一模块，等后期继续开发右侧空地的内容时，再详细设计【对，就是拖延

---

## 需要优化一哈

实现之后就发现有一点小问题，因为 gayhub pages 都是静态页面，在博客内切换页面时，每次都要重复去查 LeanCloud，虽然没啥影响，但毕竟是无用且多余的操作，洁癖心理决定还是优化优化

这周在学 Redis，因为 Redis 就适合做这种缓存，所以第一反应是 gayhub pages 能不能支持 js 连到 Redis，查了一下，emm 放弃，因为 Redis 是服务端的东西，gayhub pages 这边没办法

于是继续咨询咕果大哥，有哪些 javascript 能用的前端缓存，然后就发现了 [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) 和 [sessionStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage) 这个东西，很是符合我的需求

---

## 用用 sessionSorage

简单学了一下，因为 localStorage 和 sessionStorage 的差别是，localStorage 永久保存，除非用户主动使用浏览器的清理、或者程序中调用清理方法，而 sessionStorage 只要当前的标签页关掉，就自动清掉。所以我这里就选用 sessionStorage

用起来很简单，主要就用了两个方法（因为不涉及删除操作）

#### window.sessionStorage

通过 window.sessionStorage 可以判断浏览器是否支持使用 sessionStorage，还可以获取 storage 对象来做后续操作

#### setItem/getItem

官网的栗子：

```javascript
//保存
sessionStorage.setItem('key', 'value');
//获取
var data = sessionStorage.getItem('key');
```

---

## 优化

最终就用了这个方案：

在原有调用 LeanCloud API 的代码基础上，增加了对 window.sessionStorage 的判断，如果可以用，就再判断下 storage 中有没有我定义的热门文章数据，如果有，取出来直接用，如果没有，就调 API 取，然后保存进去

这样的话，只要是当前同一个标签页，不论在博客内点了多少次不同的链接，查看了多少页面，只有第一次会发起请求查询 LeanCloud，其余的都直接查缓存，这样就降低了 LeanCloud 的压力（哪来的什么压力）

详细的代码可以直接看 [footer.html](https://github.com/PriestTomb/PriestTomb.github.io/blob/master/_includes/footer.html) 中的 `getTopReadArticleList()` 和 `refreshTopRead()` 两个函数（emm 很简单的东西）

Chrome 中 F12 可以清楚得看到这个数据

![查看保存的数据.jpg](https://i.loli.net/2018/11/07/5be2f3e9d3434.jpg)
