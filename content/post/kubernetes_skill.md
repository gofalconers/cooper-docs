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
3、因为节点主机不支持overlay2的存储类型
```shell
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=info msg="loading plugin "io.containerd.snapshotter.v1.btrfs"..." module=containerd type=io.containerd.snapshotter.v1
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=warning msg="failed to load plugin io.containerd.snapshotter.v1.btrfs" error="path /var/lib/docker/containerd/daemon/io.containerd.snapshotter.v1.btrfs must
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=info msg="loading plugin "io.containerd.snapshotter.v1.overlayfs"..." module=containerd type=io.containerd.snapshotter.v1
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=warning msg="failed to load plugin io.containerd.snapshotter.v1.overlayfs" error="/var/lib/docker/containerd/daemon/io.containerd.snapshotter.v1.overlayfs do
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=info msg="loading plugin "io.containerd.metadata.v1.bolt"..." module=containerd type=io.containerd.metadata.v1
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=warning msg="could not use snapshotter btrfs in metadata plugin" error="path /var/lib/docker/containerd/daemon/io.containerd.snapshotter.v1.btrfs must be a b
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=warning msg="could not use snapshotter overlayfs in metadata plugin" error="/var/lib/docker/containerd/daemon/io.containerd.snapshotter.v1.overlayfs does not
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=info msg="loading plugin "io.containerd.differ.v1.walking"..." module=containerd type=io.containerd.differ.v1
```
4、节点主机自动关机，在master机器上发现没有几秒内进行转移服务容器，待确定是在哪配置的时间项
5、在pv的服务存储卷节点机器上停止docker，待确认存储卷内容是否会进行转移，实现分布式存储；确定调度时间以及etcd性能是否有影响调度