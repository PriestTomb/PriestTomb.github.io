---
layout: post
title: CSS-当后端程序猿遭遇font-family
date: 2017-10-23
categories:
- 乱七八糟的前端
tags: [CSS]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

### 吃鲸

下午因为看不惯 post 页中的英文字体，想看看 CSS 调整下博客的字体，然后就看到了这么一行：

{% highlight css %}
  font-family: Exo,'-apple-system','Open Sans',HelveticaNeue-Light,'Helvetica Neue Light','Helvetica Neue','Hiragino Sans GB','Microsoft YaHei',Helvetica,Arial,sans-serif;
{% endhighlight %}

WHAT??? 这一长串都是啥？

还算面熟的就只有 Microsoft YaHei(微软雅黑) 和 Arial 这两个常见字体，最后的 sans-serif 还是我前两天[刚学到](https://kb.cnblogs.com/page/192018/)的：

> Serif的意思是，在字的笔划开始及结束的地方有额外的装饰，而且笔划的粗细会因直横的不同而有不同。相反的，Sans Serif则没有这些额外的装饰，笔划粗细大致差不多。如下图：
>
> ![Serif和Sans Serif的区别.jpg](/images/blog_img/20171023/diff.jpg)

分别拿 Exo、-apple-system 去搜了一下，大概懂了，这些都是字体，但...为什么要写这么多？

### 学习

用 font-family 做关键字搜了下，在知乎某个问答下面看到了很多人推荐 Ruby China 的这篇[Web 中文字体应用指南](https://ruby-china.org/topics/14005)，另外也参考学习了这篇[如何优雅的选择字体(font-family)](https://segmentfault.com/a/1190000006110417)，大概明白了这一长串的字体配置是怎么回事：

* 这么多字体是为了兼容各个系统和浏览环境，比如 Mac 下、Win 下、Linux 下等等
* 先英文字体后中文字体是因为大多数英文字体中没有中文，而大多数中文字体里虽然有英文，但不是很美观，为了“优雅”，所以优先使用我们觉得漂亮的英文字体来展示英文，再用漂亮的中文字体来展示中文
* 中文字体的写法要注意尽量用字体名称，即英文的名称
