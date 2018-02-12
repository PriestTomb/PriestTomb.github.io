---
layout: post
title: 处理Gitalk中由于文章URL过长导致的Validation Failed(422)
date: 2018-02-12
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

之前博客中使用 gitment 作评论时，遇到过这个报错：

```
Error: Validation Failed.
```

HTTP 编码为 422，大家应该都已经知道了这个问题的原因：文章的 URL 过长，生成 issue 时超过了 label 的长度限制

最近刚从 gitment 改成 gitalk，因为在 gitalk 中可以设置这些参数，今天在 gitalk 的 [issue#102](https://github.com/gitalk/gitalk/issues/102) 里看到一个小哥提了个思路：

![解决](http://oxujjb0ls.bkt.clouddn.com/image/gitalk%E9%97%AE%E9%A2%98/%E8%A7%A3%E5%86%B3%E6%80%9D%E8%B7%AF.png)

觉得可行，于是就试了一下

#### 0. 找一个 js 实现的 md5

网上随便找了找，用的是这个[JavaScript-MD5](https://github.com/blueimp/JavaScript-MD5)

#### 1. 引入 js，对 url 加密

```
{% raw %}
<script src="{{ site.baseurl }}/js/md5.min.js"></script>

var gitalk = new Gitalk({
  clientID: 'xxxxxxxxxxx',
  clientSecret: 'xxxxxxxxxxxxxxxxxxx',
  repo: 'xxxx.github.io',
  owner: 'xxxx',
  admin: ['xxxx'],
  id: md5(location.href)
})
{% endraw %}
```

#### 2. 测试验证

提交后打开之前有问题的文章，发现已经可以正常创建 issue 了

#### 3. 遗留问题

在 gitalk 的 [issue#104](https://github.com/gitalk/gitalk/issues/104) 中，看到有这个问题：

> 使用的是 URL 作为 issue 的 tag 来进行关联的，在微信中打开会默认增加一些请求参数，导致使用 URL 无法关联 issue 而会重新创建 issue。

因为没做过微信相关的开发，也没在微信内置浏览器中开过博客，所以也是头一次听说这种问题，自己也测试了一下，发现确实是这样的：

![微信打开有问题](http://oxujjb0ls.bkt.clouddn.com/image/gitalk%E9%97%AE%E9%A2%98/%E5%BE%AE%E4%BF%A1%E6%89%93%E5%BC%80%E9%97%AE%E9%A2%98.png)

这样一来，label 过长的问题虽然解决了，但因为微信内置浏览器会改 title，相当于还是留下了一个问题，下午稍微研究了下，暂时也不知道怎么解决

先留个坑吧。。
