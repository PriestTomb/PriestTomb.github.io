---
layout: post
title: 使用Nginx为Elasticsearch集群做负载均衡及安全控制
date: 2019-06-22
categories: [方案设计实践]
tags: [ELK, Nginx, Elasticsearch]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

# 写在前面

Nginx 的使用可以为小型 Elasticsearch (下文简称 ES)集群提供负载均衡，同时也能使用 `auth_basic` 为 ES 较为开放的 REST API 提供一层安全保障

如果是较为大型的 ES 集群，可以划分更详细的角色（可参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html)），包括主节点、数据节点、Ingest 节点(功能上来说就是预处理节点)、协调节点等，这里暂不细说（因为 ES 5.* / 6.* / 7.* 各个版本还是有一些出入的，所以先挖个坑看看什么时候填），其中的协调节点的作用正是负载均衡（在小型集群中可以不指定专门的角色，每个节点都是主节点、数据节点，连接哪个节点都一样）

---

# 环境说明

* Linux 2.6.32

* Nginx 1.14.0

* Elasticsearch 6.4.0

---

# 步骤

### 0. 使用 htpasswd 生成密码文件

使用命令生成可以参考 Apache 的[官方文档](https://httpd.apache.org/docs/current/programs/htpasswd.html)

因为有些服务器没有 Apache 服务，所以现在也有很多在线生成，比如开源中国的[在线工具](https://tool.oschina.net/htpasswd)里就有，输入用户名密码，选择适合自己服务器系统的加密算法，生成加密结果串，然后到服务器上新建一个密码文件，保存这个加密结果串就行了

### 1. 配置 Nginx 负载均衡

Nginx 的安装使用这里就不再赘述，在之前的[《Nginx学习总结》](https://priesttomb.github.io/web%E6%9C%8D%E5%8A%A1%E5%99%A8/2018/11/12/Nginx%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93/)中有简单的介绍编译安装步骤

编辑 nginx.conf，配置 ES 集群的各主机地址

```
# 配置上游服务器集群信息
upstream es-cluster {
  server 10.0.0.100:9200 weight=1;
  server 10.0.0.101:9200 weight=1;
  server 10.0.0.102:9200 weight=1;
  keepalive 1000;
}

# 配置代理服务器信息
server {
  listen 19200;
  server_name localhost;

  location / {
    proxy_http_version  1.1;
    proxy_set_header    Connection "";
    proxy_pass http://es-cluster;
  }

  location /nginx-status {
      stub_status on;
  }
}
```

### 2. 配置 auth_basic

同样，编辑 nginx.conf，在 location 配置中补充 auth_basic，指定步骤 0 中新建的密码文件路径

```
location / {
  auth_basic  "login";
  auth_basic_user_file /path/to/htpasswd;
}
```

---

# 引入安全控制后如何使用

按上述三步的操作，就可以实现负载均衡和安全控制，安全方面的细节还有例如配置服务器的防火墙，限制 IP 和端口等等

多了一层身份验证后，在使用的时候就需要用户名密码，比如用 elasticsearch-head 时，改为连接 Nginx 地址后，网页就会弹出输入用户名密码的窗口

这里再介绍下如何用 Postman 和 Java 客户端

### 0. Postman

![Postman使用认证.png](/images/blog_img/20190622/Postman使用认证.png)

如图所示，在使用 REST API 时，在 **Authorization** 中选择加密类型，并输入用户名密码即可

### 1. Java Rest Client

ES 的 Java 客户端现在大都是用 RestClient，不过官方有两种，一个是 `RestClient` 一个是 `RestHighLevelClient`，两者的初始化方式类似

```java
// RestClient
RestClient restClient = RestClient.builder(
        new HttpHost("localhost", 9200, "http"),
        new HttpHost("localhost", 9201, "http")).build();

// RestHighLevelClient
RestHighLevelClient client = new RestHighLevelClient(
        RestClient.builder(
                new HttpHost("localhost", 9200, "http"),
                new HttpHost("localhost", 9201, "http")));
```

而需要进行用户名密码认证时，就变成了这样：

```java
final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
credentialsProvider.setCredentials(AuthScope.ANY,
        new UsernamePasswordCredentials("user", "password"));

RestHighLevelClient client = new RestHighLevelClient(
  RestClient.builder(new HttpHost("localhost", 9200))
    .setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {
      @Override
      public HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpClientBuilder) {
          return httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
      }
    }));
```

更详细的 REST Client 使用可以参考官方的介绍文档 [Java REST Client](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.4/index.html)
