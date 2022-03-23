---
layout: post
title: Logstash 插件(JRuby)-文件操作
date: 2022-03-23
categories:
- 技术
tags: [ELK, Logstash, Ruby]
status: publish
type: post
published: true
---

# 1 写在前面

在写 Logstash 插件的过程中，借鉴和学习到一些 JRuby 的书写技巧，随手记录一下，仅供参考。


---

<h3>混入防转防爬防抄袭声明：本文<a href="https://priesttomb.github.io/%E6%8A%80%E6%9C%AF/2022/03/23/logstash-plugin-jruby-operate-file/">《Logstash 插件(JRuby)-文件操作》</a>首发且仅发布于<a href="https://priesttomb.github.io/">没有名字的博客</a></h3>

---

# 2 文件操作

写过需要在内存中统计数据的插件，为了防止服务重启或宕机导致丢数据，也实现了一个比较简陋的持久化，就是定时将内存中的数据写入本地文件，如果遇到宕机重启，则在插件初始化时从本地文件中读取之前的统计数据。

### 2.1 新建/打开文件

判断文件是否已存在，如果存在，直接打开，如果不存在，则新建。

```ruby
if File.exist?(@file_path)
  persist_file = File.open(@file_path, "w+")
else
  persist_file = File.new(@file_path, "w+")
end
```

第二个参数为文件的模式，常用如下：

`r`：只读，从文件开头开始，默认模式。

`r+`：读写，从文件开头开始。

`w`：只写，将现有文件的长度截断为 0 或新建一个文件进行写入。

`w+`：读写，将现有文件的长度截断为 0 或新建一个文件进行读写。

`a`：只写，每次写入都在文件末尾追加，如果文件不存在，则新建。

`a+`：读写，每次写入都在文件末尾追加，如果文件不存在，则新建。

### 2.3 读文件

方法挺多的，如前面所讲，我需要在 Logstash 启动，初始化插件时，从文件中读出之前保存的信息，还原内存中的统计结果，所以可以直接这么写：

```ruby
str = File.open(@file_path, &:readline)
```

使用`&`指定一个代码块，这里使用的是 IO 的 readline 方法，就可以打开文件的同时，直接获取一行内容，并且不需要手动关闭文件。

其他的读文件内容的写法还有：

```ruby
# 读取所有行，到一个数组中
IO.readlines(@file_path)

# 或直接遍历
File.foreach(@file_path).with_index do |line, line_no|
   puts "#{line_no}: #{line}"
end
```

### 2.4 写文件

```ruby
persist_file.write(str)
```
