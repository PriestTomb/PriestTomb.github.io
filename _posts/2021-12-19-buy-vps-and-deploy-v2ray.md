---
layout: post
title: 购买 VPS 主机部署 V2Ray 并配置 CloudFlare 拯救被墙的 IP
date: 2021-12-19
categories:
- 技术
tags: [v2ray, vps, cloudflare, fuckgfw]
status: publish
type: post
published: true
---

# 0 写在前面

在之前的[《2021-09-06 随笔》](https://priesttomb.github.io/%E6%97%A5%E5%B8%B8/2021/09/06/essay-20210906/)中有讲到，在用了两年机场之后，考虑种种原因，现在还是打算自己买主机部署服务。

10 月份的时候，抽了几天时间，参考了一些网上的教程，先领了谷歌云（下文简称 GCP）的 300 刀赠金，买了一台便宜的主机，部署了 V2Ray 服务。

接着从 [freenom](https://www.freenom.com/) 申请了一个免费域名，准备在 CloudFlare 配置一下 CDN 转发，不过那段时间 freenom 有问题，导致 CloudFlare 无法添加（现在已经恢复了）。网上看了一下确实有这个问题，而且短期内貌似无法解决，于是转向 [GoDaddy](https://www.godaddy.com/)，花了 15 块买一个先用着。

之后这两个月还算风平浪静，除了每天到晚上的时候确实网速略微堪忧，白天使用 Google 和 YouTube 还是很正常的。

直到。。。12 月月初继续看 GCP 的账单时，猛然发现 300 刀的赠金原来只有 3 个月有效期，而下个月就要到期了！衡量了一下 GCP 每个月 9 刀左右的消费（流量另算这点确实是个不小的问题），准备再看看其他更主流的“更适合梯子的” VPS 产品，降低点开销。

于是有了本文，一方面随手记录下整个过程，一方面把自己在用的 V2Ray 服务的配置也记录一下，省得以后再有重新部署的时候还要翻看别人的教程。

注：

本文不是严肃的教程，本人对这些技术没有深入的研究，纯粹是参考了别人的教程及一部分官方的文档来搭建了 V2Ray + Nginx + CloudFlare 这一套，不保证文中的操作/设置都是最佳的（因为我没刻意关注有哪些必须要做的优化），有需要改进的地方欢迎评论指出/讨论，谢谢。

---

# 1 挑一个 VPS

在简单的搜索之后，暂定了两款待选产品：[DigitalOcean](https://www.digitalocean.com/) 和 [Vultr](https://www.vultr.com/)，看中的都是最便宜的 VPS 一个月只要 5 刀，免费的流量也完全够我日常使用。

## 1.1 DigitalOcean 开幕暴击

打开首页，赫然发现有个活动，新注册的账号给一个有效期 60 天的 100 刀赠金，三下五除二注册了一个新号，注册时还顺便预充了 5 刀，然后，跳转到控制台，然后就傻眼了。

![do账号被锁.png](/images/blog_img/20211219/do账号被锁.png)

按提示，打开工单，想看看到底是什么原因，可实际是没有工单，也没有站内信、提醒之类的，去 Google 了一下才发现，DigitalOcean 为了防止薅羊毛，限制了只能使用真实 IP 注册，而我刚才是挂着代理操作的，输的信用账单地址也是生成的假地址。

还看到一些帖子在说要去跟客服扯皮，才能把账号解锁，一下子没了耐心，懒得费心去折腾，就开了一个工单，让他们退还刚才充的 5 刀，我也懒得用了（第二天收到回信，钱也退了）。

## 1.2 Vultr 不香嘛

打开首页，直接注册，注册完后，在控制台绑了一个付款的信用卡，先充 10 刀在账户里（这次账单地址也懒得生成一个假的了，直接用中文填了一个近似真实的）。

在选择云主机参数时，在网上先找了个测试 Vultr 各机房速度的 ping 脚本，看看我这里的网络连哪个机房会好一些，后面发现北美的机房甚至比东亚的机房还稳定，于是果断抛弃了东亚机房。主机的系统还是选了和之前 GCP 一样的 Ubuntu 18.04。

最后主机的规格选择最便宜的每个月 5 刀的配置，1 CPU + 1 G 内存 + 1 T 流量。后面其他的配置就添加了一个 SSH 密钥，其他默认都没选，直接下单部署。

部署成功后，打开 MobaXterm 准备登上去部署服务，想起来先测一下 IP，ping 了一下，呵呵，果然也是被屏蔽的。。刚才加密钥有点想当然了。没办法还是只能在网页上直接打开控制台操作了，Vultr 的 web 版控制台比 GCP 的延迟高很多，用起来很是不爽。

---

<h3>混入防转防爬防抄袭声明：本文<a href="https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2021/12/19/buy-vps-and-deploy-v2ray/">《购买 VPS 主机部署 V2Ray 并配置 CloudFlare 拯救被墙的 IP》</a>首发且仅发布于<a href="https://priesttomb.github.io/">没有名字的博客</a></h3>

---

# 2 VPS 的一些额外设置

## 2.1 时间校准

```
# 启用 NTP 服务
sudo timedatectl set-ntp true
# 将时区设为“亚洲/上海”
sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# 将硬件时钟调整到与当前系统时间一致
sudo hwclock --systohc
```

## 2.2 更新软件包

```
sudo apt update && sudo apt upgrade && sudo apt autoremove
sudo apt install curl openssl
```

## 2.3 开启 BBR

```
sudo bash -c "echo 'net.core.default_qdisc=fq' >> /etc/sysctl.conf"
sudo bash -c "echo 'net.ipv4.tcp_congestion_control=bbr' >> /etc/sysctl.conf"
# 重载生效
sudo sysctl --system

# 检查
sudo sysctl net.core.default_qdisc   # 结果是：net.core.default_qdisc = fq
sudo sysctl net.ipv4.tcp_congestion_control  # 结果是：net.ipv4.tcp_congestion_control = bbr
lsmod | grep bbr  # 可以看到 tcp_bbr
```

---

# 3 安装/部署 V2Ray

## 3.1 安装

参考 [V2Fly](https://www.v2fly.org/guide/start.html) 项目的文档：

```
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-dat-release.sh)
```

## 3.2 配置

`sudo vi /usr/local/etc/v2ray/config.json` 配置

```
{
    "log":{
        "loglevel":"warning"
    },
    "routing":{
        "domainStrategy":"AsIs",
        "rules":[
            {
                "type":"field",
                "ip":[
                    "geoip:private"
                ],
                "outboundTag":"block"
            }
        ]
    },
    "inbounds":[
        {
            "listen":"127.0.0.1",
            "port":18964,
            "protocol":"vmess",
            "settings":{
                "clients":[
                    {
                        "id":"3b8f8061-5d1d-412a-b6a7-ba970fe237e6",
                        "alterId":64
                    }
                ]
            },
            "streamSettings":{
                "network":"ws",
                "wsSettings":{
                    "path":"/ray"
                }
            }
        }
    ],
    "outbounds":[
        {
            "protocol":"freedom",
            "tag":"direct"
        },
        {
            "protocol":"blackhole",
            "tag":"block"
        }
    ]
}
```

与其他地方有关联的配置项：

* port：与 Nginx 里配置的转发端口相关

* wsSettings.path：与 Nginx 里配置的匹配 URL 相关

* clients.id：与 V2Ray 客户端的配置相关

---

# 4 安装 Nginx

```
sudo apt install nginx
```

配置在后面，CF 设置了 HTTPS 和生成证书后再一并配置。

---

# 5 设置 CloudFlare

## 5.1 由 CF 接管 DNS 解析

CF 添加站点，然后在域名现在的托管商管理页面，修改 NameServer 为 CF 提供的服务器地址，截图以 freenom 为例。

![修改NameServer.png](/images/blog_img/20211219/修改NameServer.png)

然后在 DNS 菜单中，配置域名要解析到的真实 IP，即 VPS 的 IP。

![CF接管DNS.png](/images/blog_img/20211219/CF接管DNS.png)

## 5.2 生成证书

在 SSL/TLS -> 源服务器 菜单中，创建证书。

![创建证书.png](/images/blog_img/20211219/创建证书.png)

创建后，将页面中生成的证书和私钥复制，备份，扔到服务器上。

创建证书文件和私钥文件。

```
vi /etc/ssl/cert.pem
vi /etc/ssl/key.pem
```

## 5.3 进一步安全配置

在 SSL/TLS -> 源服务器菜单中，打开“经过身份验证的源服务器拉取”

![打开经过身份验证的源服务器拉取.png](/images/blog_img/20211219/打开经过身份验证的源服务器拉取.png)

同时将 CF 的证书（证书见[官方文档](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/set-up)）扔到服务器上。

```
vi /etc/ssl/cloudflare.crt
```

回到 SSL/TLS -> 概述，将模式改为“完全”。

![严格加密模式.png](/images/blog_img/20211219/严格加密模式.png)

在 SSL/TLS -> 边缘证书，将最低 TLS 版本改为 1.2。

![最低TLS1.2.png](/images/blog_img/20211219/最低TLS1.2.png)

---

# 6 配置 Nginx

在上述 CF 一堆证书设置的基础上，配置 Nginx，`vi /etc/nginx/conf.d/my.conf`

```
server {
  listen 80 default_server;
  listen [::]:80 default_server;
  charset utf-8;
  server_name YOUR_HOST_NAME;

  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  charset utf-8;
  server_name YOUR_HOST_NAME;

  ssl_certificate /etc/ssl/cert.pem;
  ssl_certificate_key /etc/ssl/key.pem;
  ssl_client_certificate /etc/ssl/cloudflare.crt;
  ssl_verify_client on;
  ssl_session_timeout 1d;
  ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
  ssl_session_tickets off;

  # intermediate configuration
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_prefer_server_ciphers off;

  # HSTS (ngx_http_headers_module is required) (63072000 seconds)
  add_header Strict-Transport-Security "max-age=63072000" always;

  root /var/www/html;
  index index.html index.htm;

  location / {
    try_files $uri $uri/ =404;
  }

  location /ray {
    if ($http_upgrade != "websocket") {
      return 404;
    }
    proxy_redirect off;
    proxy_pass http://127.0.0.1:18964;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;

    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

这里的配置要注意的有：

* server_name：要匹配自己的域名

* proxy_pass：指向前面 V2Ray 配置中的端口

* ssl相关的配置：要和前面在服务器上保存的文件匹配

除此之外，为了让服务看起来像个正常的网站，/var/www/html/index.html 可以随便写入一些 HTML 修饰一下。

---

# 7 启动服务

设置自启，并启动服务：

```
sudo systemctl enable v2ray nginx --now
```

---

# 8 CloudFlare 配置反代（可以试试）

这个是在网上瞎逛的时候搜到的一个做法，这玩意儿到底能不能以及什么原理就没详细了解了，反正参考了一些教程配置试了一下。

在菜单 Workers 中使用免费的服务，点创建后，服务名称可以随意，反正是拿来配在 V2Ray 客户端里的，启动器也可以随意，点创建后可以再改代码。

![创建worker服务.png](/images/blog_img/20211219/创建worker服务.png)

```
addEventListener(
    "fetch", event => {
        let url = new URL(event.request.url);
        url.hostname = "YOUR_HOST_NAME";
        url.protocol = "https";
        let request = new Request(url, event.request);
        event.respondWith(
            fetch(request)
        )
    }
)
```

把 `url.hostname` 改成真实的域名地址就可以。

之后，在 V2Ray 客户端里，服务器地址和伪装域名都填这个新的地址就可以了。使用这个所谓的反代跟真实域名，在我自己日常的使用里暂时是没发现什么区别。

---

# 9 最后

部署的步骤整体参考了博文 [V2Ray (WebSocket + TLS + Web + Cloudflare) 手动配置详细说明](https://ericclose.github.io/V2Ray-TLS-WebSocket-Nginx-with-Cloudflare.html)，正如开头所讲，这些东西理解的不多，照葫芦画瓢居多，有什么配置是不对的，或者可以优化的，欢迎评论讨论。
