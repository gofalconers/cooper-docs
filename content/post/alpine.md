---
title: "Alpine"
date: 2018-06-25T15:39:58+08:00
draft: false
subtitle:
bigimg: [{src: "", desc: ""}]
tags: []
---

<!--more-->

### alpine使用技巧

换源 sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
limits.h: No such file or directory 需要安装musl-dev和gcc编译环境
execvp: No such file or directory 需要安装build-base，包含g++、make
搭建alpine私有源 https://my.oschina.net/funwun/blog/710877