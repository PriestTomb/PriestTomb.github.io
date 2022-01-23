---
layout: post
title: jekyll使用LeanCloud记录文章的访问次数
date: 2017-11-06
categories:
- 日常
tags: [jekyll博客]
status: publish
type: post
published: true
---

# 写在前面的前面

**2020-01-22 注**

该博文用于介绍 LeanCloud 创建相关内容及博客中如何实现对访问次数的记录等，但由于 LeanCloud 于 2019 年 7 月之后，需要使用手持身份证做实名，所以个人博客现已经放弃使用 LeanCloud，改用 Google Firebase

相关博文详见[《Jekyll站点使用Firebase替代LeanCloud记录文章的访问次数》](https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2019/08/31/change-leancloud-to-firebase/)

# 写在前面

在还没为博客引入评论功能前，先研究了下怎么像 hexo 那样能统计每篇文章的访问次数，查了几篇博客，是说借第三方的云存储，说白了就是一个云数据库，建张表，存着各博客的访问次数，有新访问时，就更新下数据。。

看了其他的博客都是很随意的刷一次页面就 +1 访问，总觉得哪里怪怪的。。于是多折腾了一个条件限制：

> 同一个 IP，如果在一分钟内重复刷新，也不会喜+1

说着容易(其实确实很容易)，那就看看怎么实现吧

---

# 云端相关

### 1. 注册 LeanCloud

我也不知道为什么好多人都是推荐用 [LeanCloud](https://leancloud.cn/)，就像图床大家喜欢用 [七牛](https://www.qiniu.com/) 差不多吧。。

### 2. 新建应用

使用免费的开发版也足够了，小博客的访问量哪里会有很多咯(QAQ)

![leancloud创建应用.png](/images/blog_img/20171106/leancloud创建应用.png)

### 3. 新建 Class

应用创建完之后再新建 Class，可以理解成数据库里的一张张表

这里需要建两张表：

![新建两个CLASS.png](/images/blog_img/20171106/新建两个CLASS.png)

首先是访问次数的表，新增三个字段：文章URL、文章标题、已访问的次数

![times表三个字段.png](/images/blog_img/20171106/times表三个字段.png)

其次是每个访客的记录表，新增两个字段：访客IP、访问的文章URL

![record表两个字段.png](/images/blog_img/20171106/record表两个字段.png)

### 4. 安全设置

关闭不需要的服务，配置安全域名，本地4000地址测试时可以先加上，测试完再删掉也行

![关闭不使用的服务及新增安全域名.png](/images/blog_img/20171106/关闭不使用的服务及新增安全域名.png)

### 5. 记下 AppID 和 AppKey

![记录id和key值.png](/images/blog_img/20171106/记录id和key值.png)

像官方所说：

> MasterKey 是最高权限的 Key，一旦泄露，请立刻使用「重置」按钮重置

---

# 本地代码

### 1. \_config.yml

配置文件中新增以下配置：

```
leancloud:
  enable: true
  app_id: xxxxxxxxxxxxxxx
  app_key: xxxxxxxxxxxxxxx
```

### 2. footer.html

任意一个 \_includes 目录下的 且 在 default.html 中引入的文件都行，我使用了 footer.html

```
{% raw %}
{% if site.leancloud.enable %}
  <script src="https://code.jquery.com/jquery-3.2.0.min.js"></script>
  <script src="https://cdn1.lncld.net/static/js/av-core-mini-0.6.1.js"></script>
  <script>AV.initialize("{{site.leancloud.app_id}}", "{{site.leancloud.app_key}}");</script>
  <script>
    //新增访问次数
    function addCount(Counter) {
      // 页面（博客文章）中的信息：leancloud_visitors
      // id为page.url， data-flag-title为page.title
      var $visitors = $(".leancloud_visitors");
      var url = $visitors.attr('id').trim();
      var title = $visitors.attr('data-flag-title').trim();
      var query = new AV.Query(Counter);

      // 只根据文章的url查询LeanCloud服务器中的数据
      query.equalTo("post_url", url);
      query.find({
        success: function(results) {
          if (results.length > 0) {//说明LeanCloud中已经记录了这篇文章
            var counter = results[0];
            counter.fetchWhenSave(true);
            counter.increment("visited_times");// 将点击次数加1
            counter.save(null, {
              success: function(counter) {
                var $element = $(document.getElementById(url));
                var newTimes = counter.get('visited_times');
                $element.find('.leancloud-visitors-count').text(newTimes);
              },
              error: function(counter, error) {
                console.log('Failed to save Visitor num, with error message: ' + error.message);
              }
            });
          } else {
            // 执行这里，说明LeanCloud中还没有记录此文章
            var newcounter = new Counter();
            /* Set ACL */
            var acl = new AV.ACL();
            acl.setPublicReadAccess(true);
            acl.setPublicWriteAccess(true);
            newcounter.setACL(acl);
            /* End Set ACL */
            newcounter.set("post_title", title);// 把文章标题
            newcounter.set("post_url", url); // 文章url
            newcounter.set("visited_times", 1); // 初始点击次数：1次
            newcounter.save(null, { // 上传到LeanCloud服务器中
              success: function(newcounter) {
                var $element = $(document.getElementById(url));
                var newTimes = newcounter.get('visited_times');
                $element.find('.leancloud-visitors-count').text(newTimes);
              },
              error: function(newcounter, error) {
                console.log('Failed to create');
              }
            });
          }
        },
        error: function(error) {
          console.log('Error:' + error.code + " " + error.message);
        }
      });
    }

    //仅根据url和title查出当前访问次数，不做+1操作
    function showCount(Counter) {
      var $visitors = $(".leancloud_visitors");
      var url = $visitors.attr('id').trim();
      var title = $visitors.attr('data-flag-title').trim();
      var query = new AV.Query(Counter);

      // 只根据文章的url查询LeanCloud服务器中的数据
      query.equalTo("post_url", url);
      query.find({
        success: function(results) {
          if (results.length > 0) {//说明LeanCloud中已经记录了这篇文章
            var counter = results[0];
            var $element = $(document.getElementById(url));
            var newTimes = counter.get('visited_times');
            $element.find('.leancloud-visitors-count').text(newTimes);
          } else {
            //如果表里没查到记录，那就是异常情况了
            console.log('异常情况，不应该没记录的');
          }
        },
        error: function(error) {
          console.log('Error:' + error.code + " " + error.message);
        }
      });
    }

    //调用API获取IP
    function getVisitorIpAndJudge() {
      var ip;
      var options = {
        type: 'POST',
        dataType: "json",
        //async: false, //jquery3中可以直接使用回调函数，不用再指定async
        url: "https://freegeoip.net/json/?callback=?"
      };
      $.ajax(options)
      .done(function(data, textStatus, jqXHR) {
        if(textStatus == "success") {
          ip = data.ip;
        }
        judgeVisitor(ip)
      });
    }

    //判断访客是否已访问过该文章，及访问时间，符合条件则增加一次访问次数
    function judgeVisitor(ip) {
      var Counter = AV.Object.extend("visited_times");
      var Visitor = AV.Object.extend("visitors_record");

      var $postInfo = $(".leancloud_visitors");
      var post_url = $postInfo.attr('id').trim();

      var query = new AV.Query(Visitor);

      query.equalTo("visitor_ip", ip);
      query.equalTo("post_url", post_url);
      query.find({
        success: function(results) {
          if (results.length > 0) {
            console.log('该IP已访问过该文章');

            var oldVisitor = results[0];

            var lastTime = oldVisitor.updatedAt;
            var curTime = new Date();

            var timePassed = curTime.getTime() - lastTime.getTime();

            if(timePassed > 1 * 60 * 1000) {
              console.log('距离该IP上一次访问该文章已超过了1分钟，更新访问记录，并增加访问次数');

              addCount(Counter);

              oldVisitor.fetchWhenSave(true);
              oldVisitor.save(null, {
                success: function(oldVisitor) { },
                error: function(oldVisitor, error) {
                  console.log('Failed to save visitor record, with error message: ' + error.message);
                }
              });
            } else {
              console.log('这是该IP 1分钟内重复访问该文章，不更新访问记录，不增加访问次数');
              showCount(Counter);
            }
          } else {
            console.log('该IP第一次访问该文章，保存新的访问记录，并增加访问次数');

            addCount(Counter);

            var newVisitor = new Visitor();
            /* Set ACL */
            var acl = new AV.ACL();
            acl.setPublicReadAccess(true);
            acl.setPublicWriteAccess(true);
            newVisitor.setACL(acl);
            newVisitor.set("visitor_ip", ip);
            newVisitor.set("post_url", post_url);
            newVisitor.save(null, { // 上传到LeanCloud服务器中
              success: function(newVisitor) { },
              error: function(newVisitor, error) {
                console.log('Failed to create visitor record, with error message: ' + error.message);
              }
            });
          }
        },
        error: function(error) {
          console.log('Error:' + error.code + " " + error.message);
          addCount(Counter);
        }
      });
    }

    $(function() {
      if ($('.leancloud_visitors').length == 1) {
        // 文章页面，调用判断方法，对符合条件的访问增加访问次数
        getVisitorIpAndJudge();
      } else if ($('.post-link').length > 1){
        // 首页 暂未使用
        // showHitCount(Counter);
      }
    });
  </script>
{% endif %}
{% endraw %}
```

### 3. default.html

把 footer.html 引入

```
{% raw %}
  {% include footer.html %}
{% endraw %}
```

### 4. post.html

在文章页面自己想展示访问次数的地方加上

```
{% raw %}
{% if site.leancloud.enable %}
<div>
  <span id="{{ page.url }}" class="leancloud_visitors" data-flag-title="{{ page.title }}">
    Visited:
    <a href="#"><span class="leancloud-visitors-count"></span>次</a>
  </span>
</div>
{% endif %}
{% endraw %}
```

### 5. 效果

![效果.png](/images/blog_img/20171106/效果.png)

---

# 最后

本文参考：

[http://blog.csdn.net/u013553529/article/details/63357382](http://blog.csdn.net/u013553529/article/details/63357382)

[http://www.cloudchou.com/android/post-981.html](http://www.cloudchou.com/android/post-981.html)
