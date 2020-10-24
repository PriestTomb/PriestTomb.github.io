---
layout: post
title: Filebeat 无法采集日志（Permission denied）
date: 2020-04-19
categories:
- 技术
tags: [Filebeat]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

# 写在前面

在服务器上安装 Filebeat 采集业务日志时，发现启动后没有采到任何内容，排除掉不是 Filebeat 的配置错误后，发现其实是一个权限导致的问题

**---update 2020.10.24---**

经过一段时间的应用，Filebeat 直接用业务系统所属的用户部署、或者用 root 用户部署更简单一些，可以免除一些权限的问题

---

# 处理问题

业务系统部署在用户 biz 下，业务日志在 biz 的 home 路径中，这里假设为 /home/biz/logs/\*.log

而 Filebeat 使用的是另一个用户 filebeat，安装路径为 /home/filebeat/filebeat/

在 /home 目录下 `ll` 查看时，发现用户 biz 的文件夹所属用户组为 biz，访问权限为 700，也就是说其他普通用户、用户组都没有权限，这里想解决问题，简单粗暴的做法可以直接把权限更改成 777，但会有一定的安全风险，所以打算从用户组的方式入手

**以下步骤均以 root 用户进行操作**

### 0. 新建用户组

`groupadd biz_beats`

### 1. 将用户添加到用户组

`usermod -G biz_beats biz`

`usermod -G biz_beats filebeat`

### 2. 修改 biz 用户 home 目录所属用户组

`chown biz.biz_beats /home/biz/ -R`

### 3. 修改 biz 用户 home 目录所属用户组权限

`chmod 770 /home/biz/`

注意：用户组权限至少为 750，filebeat 采集日志至少需要 rx

---

# 参考

[鸟哥的 Linux 私房菜](http://cn.linux.vbird.org/linux_basic/0410accountmanager.php#users)
