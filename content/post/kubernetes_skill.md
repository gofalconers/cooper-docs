---
title: "Kubernetes_skill"
date: 2018-07-06T16:52:51+08:00
draft: false
subtitle:
bigimg: [{src: "", desc: ""}]
tags: []
---

<!--more-->
记录kubernetes的日常使用以及应用场景构思。


1、kubernetes是通过pod中容器的hostname来注册，当我们手动或者程序修改了hostname，应该就会出现服务异常；对于如何预防，通过gosu来设置程序运行时的用户限制使用root权限，防止出现安全事件。
2、centos系统内核为3.10.0-229.7.2.el7.x86_64版本，会导致节点服务器重启，丢失kubelet相关信息，查看另一台内核同样为3.10.0-229.7.2.el7.x86_64的机器同样出现重新现象导致节点kubelet相关信息丢失，内核为3.10.0-693.el7.x86_64则没有出现该现象。怀疑是kube-router功能不支持低版本内核。待确认原因