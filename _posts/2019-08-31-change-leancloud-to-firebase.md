---
layout: post
title: Jekyll站点使用Firebase替代LeanCloud记录文章的访问次数
date: 2019-08-31
categories:
- 技术
tags: [Firebase, LeanCloud, Jekyll]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

# 弃用 LeanCloud

博客原先一直在用 LeanCloud 存储文章阅读次数、点赞次数的相关数据，2019年7月前后，LeanCloud 通知接下来需要实名认证才可以继续使用，结合之前他们因为有恶意应用存储了所谓“敏感”的内容，导致被停止解析域名，所以觉得实名认证也是没办法的办法【毕竟国内嘛

本来以为就是简单的输姓名身份证，但是。。LeanCloud 居然是要用手持身份证这种万恶的认证方式，所以，只能说再见了

---

# 启用 Firebase

找了一下国内国外的相似产品，最后决定直接用 Google 家的产品 [Firebase](https://firebase.google.com/)

### 0. 注册

直接用 Gmail 帐号登录

### 1. 创建一个项目

![创建项目](/images/blog_img/20190831/创建项目.png)

### 2. 创建一个应用

![创建应用](/images/blog_img/20190831/创建应用.png)

### 3. 使用实时数据库

![启用实时数据库](/images/blog_img/20190831/启用实时数据库.png)

### 4. 学习增删改查api

参考[官方的文章](https://firebase.google.com/docs/web/setup)先把 Firebase 的 SDK 引入，并初始化 Firebase

```javascript
var firebaseConfig = {
  // 配置key、url、id等等
};
firebase.initializeApp(firebaseConfig);
```

参考[官方的文档](https://firebase.google.com/docs/database/web/read-and-write)学习增删改相关的语法

```javascript
//写入数据，用 set()
firebase.database().ref('users/' + userId).set({
  username: name,
  email: email,
  profile_picture : imageUrl
});

//读取数据，用 once()
firebase.database().ref('/users/' + userId).once('value').then(function(snapshot) {
  var username = (snapshot.val() && snapshot.val().username) || 'Anonymous';
  // ...
});

//更新数据，用 update()
var updates = {};
updates['/posts/' + newPostKey] = postData;
updates['/user-posts/' + uid + '/' + newPostKey] = postData;
return firebase.database().ref().update(updates);
```

---

# 修改博客相关代码

博客的阅读次数和点赞记录的主要逻辑还是和原来一样，可以分别参考[《jekyll使用LeanCloud记录文章的访问次数》](https://priesttomb.github.io/%E6%97%A5%E5%B8%B8/2017/11/06/jekyll%E4%BD%BF%E7%94%A8LeanCloud%E8%AE%B0%E5%BD%95%E6%96%87%E7%AB%A0%E7%9A%84%E8%AE%BF%E9%97%AE%E6%AC%A1%E6%95%B0/)和[《为博客新增了文章点赞的功能》](https://priesttomb.github.io/%E6%97%A5%E5%B8%B8/2017/11/23/add-new-function-about-like-this-post/)

由于之前 LeanCloud 的数据库还是类似关系型数据库，而 Firebase 的实时数据库是一个 KV 数据库（好像是？），所以文章表和访客表还是重新设计了一下：

* 文章表的 key 是由每篇文章的 url 进行 md5 得到的，value 主要保存文章标题、被点赞次数、访问次数、更新时间

* 访客表的 key 是由每个访客的 ip 拼接访问文章的 url 再做 md5 得到的，value 主要保存记录的创建时间、更新时间、是否点赞了这篇文章

关于这部分的 js 代码，如有需要，可以直接查看 [footer.html](https://github.com/PriestTomb/PriestTomb.github.io/blob/master/_includes/footer.html)

---

# 遗留问题

改造测试了之后，目前虽然一切“正常”，但也不能算得上完美，还是有以下几个问题的：

* LeanCloud 中可以直接设置只接受指定域名来的数据库请求，这样就不用怕暴露在仓库源码中的相关应用 key 被其他人恶意利用，但在 Firebase 中实在没找到有类似的设置，感觉 Auth 模块的配置不太好满足这个简单的需求，所以暴露了 Firebase 的相关 key 值目前还是有风险的

* Firebase 目前在国内无法直接访问，所以会少了很大一部分没有翻墙的用户的访问、点赞记录

* Firebase 的实时数据库在控制台中展示时没有 LeanCloud 控制台里那么直观，LeanCloud 中可以很方便排序、查询，但 Firebase 只能列出一个巨型 JSON 数据，如果想查看当前访问次数排行榜、点赞排行榜、当日来访用户等数据，估计只能自己写一些“接口”去查了

* Firebase 实时数据库存中文时偶尔会出现乱码，原因不明，在初始化各篇文章的原始数据时，明明第一次初始化写入的数据有乱码，删掉数据之后再写一次，发现又正常了，或者第一次正常，后面更新记录时又变成了乱码，很是奇怪

感觉还要再细细研究呐。。

如果有小伙伴已经解决了相关的问题，欢迎评论分享一哈，谢谢
