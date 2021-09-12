---
layout: post
title: Luyten 反编译报错“Invalid BootstrapMethods attribute entry 2 additional arguments required for method”
date: 2021-09-12
categories:
- 技术
tags: [java, decompiler, luyten]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

# 1 反编译失败

Java 反编译常用的都是 [JD-GUI](http://java-decompiler.github.io/) 或者 [Luyten](https://github.com/deathmarine/Luyten)，由于 JDK 的版本一路狂涨，JD-GUI 逐渐就不太适用高版本的 JDK 编译的 class 反编译了，Luyten 因为一直在持续更新中，所以稍高一些的 JDK 版本还能支持，不过最近还是遇到了反编译失败的情况。

手里有个 Java 程序的 class 文件（非 jar 文件），想反编译一下研究研究，需求就是对这一大堆的 class 文件能直接全部反编译，类似 JD-GUI 或者 Luyten 那样直接导出一个 java 文件的压缩包。

于是对整个文件夹打了一个 zip 包扔 Luyten 里，打开 zip 包倒是没有报错，不过查看某些具体的文件时就不行了，直接异常：

```java
java.lang.IllegalStateException: Invalid BootstrapMethods attribute entry: 2 additional arguments required for method java/lang/invoke/StringConcatFactory.makeConcatWithConstants, but only 1 specified.
	at com.strobel.assembler.ir.Error.invalidBootstrapMethodEntry(Error.java:244)
	at com.strobel.assembler.ir.MetadataReader.readAttributeCore(MetadataReader.java:280)
	at com.strobel.assembler.metadata.ClassFileReader.readAttributeCore(ClassFileReader.java:261)
	at com.strobel.assembler.ir.MetadataReader.inflateAttributes(MetadataReader.java:439)
	at com.strobel.assembler.metadata.ClassFileReader.visitAttributes(ClassFileReader.java:1134)
	at com.strobel.assembler.metadata.ClassFileReader.readClass(ClassFileReader.java:439)
	at com.strobel.assembler.metadata.ClassFileReader.readClass(ClassFileReader.java:377)
	at com.strobel.assembler.metadata.MetadataSystem.resolveType(MetadataSystem.java:129)
	at com.strobel.assembler.metadata.MetadataSystem.resolveCore(MetadataSystem.java:81)
	at com.strobel.assembler.metadata.MetadataResolver.resolve(MetadataResolver.java:104)
	at com.strobel.assembler.metadata.CoreMetadataFactory$UnresolvedType.resolve(CoreMetadataFactory.java:616)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.convertType(AstBuilder.java:294)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.convertType(AstBuilder.java:173)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.convertType(AstBuilder.java:169)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.createParameters(AstBuilder.java:181)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.createMethod(AstBuilder.java:662)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.addTypeMembers(AstBuilder.java:552)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.createTypeCore(AstBuilder.java:519)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.createTypeNoCache(AstBuilder.java:161)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.createType(AstBuilder.java:150)
	at com.strobel.decompiler.languages.java.ast.AstBuilder.addType(AstBuilder.java:125)
	at com.strobel.decompiler.languages.java.JavaLanguage.buildAst(JavaLanguage.java:71)
	at com.strobel.decompiler.languages.java.JavaLanguage.decompileType(JavaLanguage.java:59)
	at us.deathmarine.luyten.DecompilerLinkProvider.generateContent(DecompilerLinkProvider.java:97)
	at us.deathmarine.luyten.OpenFile.decompileWithNavigationLinks(OpenFile.java:494)
	at us.deathmarine.luyten.OpenFile.decompile(OpenFile.java:467)
	at us.deathmarine.luyten.Model.extractClassToTextPane(Model.java:420)
	at us.deathmarine.luyten.Model.openEntryByTreePath(Model.java:339)
	at us.deathmarine.luyten.Model$TreeListener$1.run(Model.java:266)

```

最初以为是本机的 JDK 版本太低了（装的还是 1.8 的版本），查了一下程序中的配置，用的是 OpenJDK-15 的版本，于是也去下了这个版本配置一下环境变量，再次尝试用 Luyten 反编译，依旧报这个异常。

搜索了一下发现 Luyten 项目的 issue 里有人提过，说是 Luyten 依赖的 Procyon 版本太低导致的。

---

# 2 尝试

先从 Luyten 发布的版本中下了一个最新的 [Luyten v0.5.4 Rebuilt](https://github.com/deathmarine/Luyten/releases/tag/v0.5.4_Rebuilt_with_Latest_depenencies)，说是支持到 Procyon 0.5.36，本以为这下该可以了，没想到还是报异常，这就让人头疼了。

去看了下 [Procyon](https://github.com/mstrobel/procyon) 项目，发现只有一个最新的[预览版本 0.6](https://github.com/mstrobel/procyon/releases/tag/0.6-prerelease) 可以下载，下下来之后打算看看用这个 procyon-decompiler 可不可以直接用命令行把整个文件夹给反编译，或者能不能支持一个 zip 文件，结果——并不支持。Procyon 居然只能反编单个文件或 jar 文件！

---

# 3 另辟蹊径

想起来 Luyten 有单 jar 包格式的发布，于是下下来，尝试启动时手动引入 procyon-decompiler 最新的 jar 包（启动类 `us.deathmarine.luyten.Luyten` 在 jar 包中的 MANIFEST.MF 文件找）：

```shell
java -cp procyon-decompiler-0.6-prerelease.jar;luyten-0.5.4.jar us.deathmarine.luyten.Luyten
```

首先，启动没有报错，其次，之前会报错的 class 文件可以反编译成功了！

---

# 4 还有其他工具

在 Procyon 项目的 README 中看到，他们有推荐另外两个依赖于 Procyon 的可视化反编译软件。

### 4.1 d4j

这个 [d4j](http://www.secureteam.net/d4j) 是依赖 JavaFX 的，但官网下下来的只是一个 jar 包，直接执行还提示缺少 JavaFX 运行组件，因为没搞过 JavaFX，随便研究了一下，也没看懂怎么启动这个 d4j 项目，遂放弃。

### 4.2 Bytecode Viewer

这个 [Bytecode Viewer](https://github.com/Konloch/bytecode-viewer) 就友好一点，下下来最新的 release，也是个单独的 jar 包，直接 `java -jar` 就可以启动。

![bytecodeviewer-mark.png](/images/blog_img/20210912/bytecodeviewer-mark.png)

这个软件同样支持直接导入 class 文件的 zip 格式压缩包，而非仅仅是 jar 包，同时还有 apk、dex 等。最大的一个优点我觉得其实是在选择反编译导出时，**可以选择不同的 decompiler**：

![diff-decompiler-mark.png](/images/blog_img/20210912/diff-decompiler-mark.png)

尝试了当前版本中默认依赖的 Procyon，果然还是会报错（指我自己测试的 class 文件），导致导出失败，不过其他的几个倒是可以用，可以说是除 Luyten 外的另一个很不错的 Java 反编译 GUI 选择了。
