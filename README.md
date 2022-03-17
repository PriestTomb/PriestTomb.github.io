Priest Tomb Blog
=====

一个审美不足的后端程序猿参考了好几个博客后七拼八凑出来的[个人博客](https://priesttomb.github.io/)

目前连个博客名字都还没想好，样式及功能持续改造中......

#### 0. 博客主体

使用了 [Jekyll](http://jekyll.com.cn/)

#### 1. 访问次数

~~使用了 LeanCloud，实现过程参考我[这篇文章](https://priesttomb.github.io/%E6%97%A5%E5%B8%B8/2017/11/06/jekyll%E4%BD%BF%E7%94%A8LeanCloud%E8%AE%B0%E5%BD%95%E6%96%87%E7%AB%A0%E7%9A%84%E8%AE%BF%E9%97%AE%E6%AC%A1%E6%95%B0/)的介绍~~

update 20190707

由于近期 LeanCloud 需要**手持身份证**做实名认证，故放弃国内产品，改用 Google 的 Firebase，设计思路依旧可以参考 LeanCloud 那篇，细节部分略有修改，有空的话可以再写一篇用 Firebase 实现的文章

update 20190901

整理了[使用 Firebase 替代 LeanCloud 的文章](https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2019/08/31/change-leancloud-to-firebase/)

#### 2. 评论功能

使用了 [gitalk](https://github.com/gitalk/gitalk)，参考项目的 README

#### 3. 站内搜索

使用了 [simple-jekyll-search.js](https://github.com/christian-fei/Simple-Jekyll-Search)，参考项目的 README 就可以了，很简单就能用了，当然，样式可以自己调整

#### 4. 文章点赞

~~因为没有所谓的登录用户用来判断访客身份，只能简单用IP做判断，和访问次数功能的实现很相似，实现思路参考我的[这篇文章](https://priesttomb.github.io/%E6%97%A5%E5%B8%B8/2017/11/23/add-new-function-about-like-this-post/)~~

update 20190707

同上述第1点中的原因，文章点赞的数据也已经转移到 Firebase，实现思路并没有过多的改动

#### 5. 流量统计

~~使用了第三方的[站长统计](https://www.umeng.com/web)~~

update 20220317

由于站长统计从 3 月 20 日起，不再支持未备案的网站，所以，自此下掉个人博客中最后一个墙内服务

#### 6. 广告

~~使用了 [Google AdSense](https://www.google.com/adsense/start/)，一方面尝试下是不是能带来一点收入，一方面用潜在的收入来激励自己写博客，广告投放的位置会不断调整，毕竟还是不要影响阅读文章【来总脸：加点广告怎么了（雾）~~

update 20191014

尝试了一段时间的广告，发现个人站的访问量本身还是偏低，广告如果没有“点击”时，仅凭“展示”是无法获得收益的，故暂时关闭广告展示
