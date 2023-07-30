---
layout: post
title: Spring Boot+Spark+Elasticsearch数据分析（一）
date: 2023-07-30
categories:
- 技术
tags: [SpringBoot, Spark, ELK, Elasticsearch]
status: publish
type: post
published: true
---

# 1 业务所需

公司有个 Spring Boot 项目，用 Elasticsearch 存储了大量的归档数据，现在需要对数据做一些偏离线的大数据分析。

---

# 2 方案筛选

对数据分析的了解仅停留在 N 年前用 Hadoop 做了个毕设，于是在网上冲浪很久，找了一些已知的方案，结合公司项目的情况，梳理了个大概。

**(1) HDFS + Spark + Hive**

优点：

* 适合做离线处理，支持类似 SQL 的语法，方便使用

缺点：

* 部署运维繁琐（统一用 CDH 版本也有各自的弊端），与 Spring Boot 结合依旧繁琐（配置文件层面）
* Spark 需要有一定的 scala 语言作为基础（当前的简单分析场景无需过虑）


**(2) Flink**

偏实时分析

**(3) Druid**

偏实时分析

优点：

* 数据插入效率高，数据具有时间属性，按时间分区存储及查询

**(4) Elasticsearch**

优点：

* 项目目前有简单使用 ES，有一定的基础
* 大厂/社区有大量的实战经验及优化方案
* Java 语言开发，对 Java 开发团队友好度更高

缺点：

* ES 是一套检索系统，只用来做统计分析不算发挥其特长
* 语法上手有难度（当前查询场景简单，暂无此问题）
* 超大型集群运维难（当前数据量无需过虑）
* Java 语言开发，对服务器资源消耗大（结合第一个缺点）

**(5) ClickHouse**

偏实时分析

优点：

* 数据存储 + 数据分析
* 性能高，支持简单的类 SQL 语法
* 数据压缩，节省磁盘空间

缺点：

* 分布式运维繁琐，一些分布式操作需要手动处理（当前的数据量无需过虑）
* 数据副本不提高查询性能（对比 ES，ES 索引多副本可以提交检索性能）
* 依赖 ZK，且数据量越大，对 ZK 集群要求越高
* C++ 语言开发，对 Java 开发团队友好度较低

因为项目中的实际使用更偏离线分析，加上看到 Elasticsearch 官方提供了 [ES-Hadoop](https://www.elastic.co/cn/what-is/elasticsearch-hadoop) 来帮助用户在 Hadoop 生态和 Elasticsearch 之间做无缝数据转移，于是另辟蹊径，把 ES 当作数据源，将数据读到 Spark 进行分布式运算。

---

<h3>混入防转防爬防抄袭声明：本文<a href="https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2023/07/30/spring-boot-spark-es-1/">《Spring Boot+Spark+Elasticsearch数据分析（一）》</a>首发且仅发布于<a href="https://priesttomb.github.io/">没有名字的博客</a></h3>

---

# 3 Spark 集群部署

决定用 Spark 做数据分析后，首先的问题，就是怎么把环境部署起来。在初期搜资料的过程中属实很迷茫，有些文章中好像在说，想用 Spark 就必须要搭 Hadoop，后面才明白，指的是利用 HDFS 来存储数据，而我的方案已经用了 ES 做数据源，自然不必部署 Hadoop 集群。

公司的服务都使用 Docker 部署，在网上找到了 bde2020 [封装好的镜像](https://hub.docker.com/u/bde2020)，拉下来一个 3.1.1 版本的，之后推送到了公司本地服务器上，然后参考了很多部署的示例，也算是艰难拼凑了一个部署脚本（暂时不包括各种优化）。

## 3.1 Master 节点部署

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: spark-master-1
  name: spark-master-1
spec:
  type: NodePort
  ports:
    - name: "master8080"
      port: 8080
      targetPort: 8080
      nodePort: 30088
  selector:
    app: spark-master-1
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: spark-master-1
  name: spark-master-1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spark-master-1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: spark-master-1
    spec:
      containers:
        - env:
            - name: SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED
              value: "no"
            - name: SPARK_MODE
              value: master
            - name: SPARK_RPC_AUTHENTICATION_ENABLED
              value: "no"
            - name: SPARK_RPC_ENCRYPTION_ENABLED
              value: "no"
            - name: SPARK_SSL_ENABLED
              value: "no"
            - name: TZ
              value: "Asia/Shanghai"
          image: path/to/spark-master:3.1.1
          name: spark-master-1
          ports:
            - containerPort: 8080
            - containerPort: 7077
          resources: {}
          volumeMounts:
            - mountPath: /home/myuser/spark/lib
              name: outter-lib
      hostAliases:
        - ip: "192.168.1.7"
          hostnames:
          - "host7"
        - ip: "192.168.1.8"
          hostnames:
          - "host8"
      volumes:
        - name: outter-lib
          hostPath:
            path: /home/myuser/spark/lib
status: {}
```

几点需要说明的内容：

**a. 重点：Master 节点的容器名称不能直接使用`spark-master`**

简单来说，k8s 会将容器名称转成下划线格式，对其进行一些环境变量的默认配置，而 `SPARK_MASTER_XXX` 与 Spark 有冲突，会报类似这样的错：

```java
java.lang.NumberFormatException: For input string: "tcp://192.168.1.5:8080"
```

详见[这个Github Issue](https://github.com/big-data-europe/docker-spark/issues/128)。

b. 映射出 8080 端口是为了看 Master 的控制台 UI，不需要可以不配置。

c. 物理机如果有配置 host，容器中也需要对真实 IP 做一下 host 配置。

d. 关于 outter-lib，是后面结合 Spring Boot 使用时需要的，后面的文章中会详细解释（Master 节点中应该不需要配置这个）。

## 3.2 Worker 节点部署

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: spark-worker
  name: spark-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spark-worker
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: spark-worker
    spec:
      containers:
        - env:
            - name: SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED
              value: "no"
            - name: SPARK_MASTER
              value: spark://spark-master:7077
            - name: SPARK_MODE
              value: worker
            - name: SPARK_RPC_AUTHENTICATION_ENABLED
              value: "no"
            - name: SPARK_RPC_ENCRYPTION_ENABLED
              value: "no"
            - name: SPARK_SSL_ENABLED
              value: "no"
            - name: SPARK_WORKER_CORES
              value: "1"
            - name: SPARK_WORKER_MEMORY
              value: 1G
            - name: TZ
              value: "Asia/Shanghai"
          image: path/to/spark-worker:3.1.1
          name: spark-worker
          resources: {}
          volumeMounts:
            - mountPath: /home/myuser/spark/lib
              name: outter-lib
      restartPolicy: Always
      hostAliases:
        - ip: "192.168.1.7"
          hostnames:
          - "host7"
        - ip: "192.168.1.8"
          hostnames:
          - "host8"
      volumes:
        - name: outter-lib
          hostPath:
            path: /home/myuser/spark/lib
status: {}
```

Worker 节点的部署配置没有什么幺蛾子，只有一个环境变量 `SPARK_MASTER` 必须配置成 `spark://spark-master:7077` 还是比较神奇的，因为各容器之间也没有配置一个叫 spark-master 的域名，而 Master 节点的容器名称也不能直接使用 spark-master，但启动后就会自动找到，不知道是不是类似 ES 中 Zen discovery 的自动发现机制。
