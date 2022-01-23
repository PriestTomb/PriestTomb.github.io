---
layout: post
title: 本地编译 Logstash 源码并导入 Eclipse 启动
date: 2022-01-23
categories:
- 技术
tags: [ELK, Logstash, Eclipse]
status: publish
type: post
published: true
---

# 1 下载源码

从 Github 上下载 [Logstash](https://github.com/elastic/logstash) 的代码。

这里我依旧切换回 Logstash 6 的版本进行学习，比如选择了最高的 6.8。

---

# 2 本地编译

下载好的源码根目录中自带了一个 gradlew，这里可以直接执行一下 `gradlew assembleZipDistribution`，把项目编译一下打个 ZIP 包。其他的编译命令不熟，几个 `assembleXXXDistribution` 应该实现的效果都一样，先准备好运行的环境，再最终打包。

这期间主要是安装 JRuby 和下载各种 gems，最终可以在 vendor 目录中看到。

另外打开 logstash-core，可以在 lib 目录下发现一个新的目录 jars，这就是 logstash-core 模块运行所需的第三方 jar 包。

---

<h3>混入防转防爬防抄袭声明：本文<a href="https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2022/01/23/run-logstash-source-code-by-using-eclipse/">《本地编译 Logstash 源码并导入 Eclipse 启动》</a>首发且仅发布于<a href="https://priesttomb.github.io/">没有名字的博客</a></h3>

---

# 3 导入 Eclipse

因为对 Gradle 不是很熟，先尝试以 Gradle 项目的形式单独导入 logstash-core 目录时，会在编译的阶段报错：

```bash
Could not get unknown property 'classes' for root project 'logstash-core' of type org.gradle.api.Project.
```

看了下 build.gradle，报错的地方是：

```bash
task sourcesJar(type: Jar, dependsOn: classes) {
    from sourceSets.main.allSource
    classifier 'sources'
    extension 'jar'
}
```

这个任务前置依赖 classes，但又报错说不是 Gradle 的内置 API，同时在当前 logstash-core 模块中又没定义，这就有点奇怪，导致怎么编都报错。

最后果断还是抛开 Gradle，直接新建一个空的 Java 项目，比如 logstash-core-test，把 logstash-core 中的 src/main 全部复制粘贴一番，再新建一个 libs 目录，把前面提到的 lib/jars 目录中的第三方 jar 包复制过来，配置一下项目的 build path，把它当成一个普通的 Java 项目即可。

![src.main.png](/images/blog_img/20220123/src.main.png)

![libs.png](/images/blog_img/20220123/libs.png)

---

# 4 可能的报错

如果 Eclipse 的设置没怎么改过，或者给项目配置的 JRE 是旧版本，可能会发现 ProcessMonitor.java 中有个引入报错了：

```bash
Access restriction: The type 'UnixOperatingSystemMXBean' is not API (restriction on required library 'xxx\jre\lib\rt.jar')
```

这个可以使用高版本的 JRE 避免，比如我用 JDK 1.8.0_202 就是正常的。

或者可以改 Eclipse 的设置，Windows -> Preferences -> Java -> Complicer -> Errors/Warnings 里面的 Deprecated and restricted API 中的 Forbidden references(access rules) 选为 Warning 。

遇到这个报错是因为电脑上之前有个没卸载干净的 JDK 目录，这个项目导入时不知道为何默认使用了那个路径，就莫名出现了这个问题，后来发现不对，改成正常的路径，就一切正常了。

---

# 5 运行配置

Logstash 启动至少还有两个额外的配置要加。

### 5.1 环境变量

加入一个 LS_HOME，指向下载的 Logstash 源码的根目录。

![ls_home.png](/images/blog_img/20220123/ls_home.png)

### 5.2 启动参数

指定一个配置文件，否则默认会加载 LS_HOME/config/pipelines.yml，但这个文件默认又是空的，就启动不了。

![args.png](/images/blog_img/20220123/args.png)

---

# 6 启动

启动 org.logstash.Logstash.main(String...)，不出意外的话，应该可以正常运行了。
