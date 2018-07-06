---
title: "Kubernetes_volume"
date: 2018-07-04T15:53:19+08:00
draft: false
subtitle:
bigimg: [{src: "", desc: ""}]
tags: []
---

<!--more-->

Failed to get system container stats for "/system.slice/docker.service"
heapster为普通用户的话，在dashboard页面中没有对应的图形显示，查看heapster容器日志会看到报没有权限。
openebs节点上日志报错
Jul  4 15:27:11 nodejs-200 iscsid: connect to 10.100.129.18:3260 failed (No route to host)
Jul  4 15:27:15 nodejs-200 iscsid: connect to 10.100.129.18:3260 failed (No route to host)
Jul  4 15:27:18 nodejs-200 kubelet: E0704 15:27:18.064347    7274 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jul  4 15:27:19 nodejs-200 iscsid: connect to 10.100.129.18:3260 failed (No route to host)
Jul  4 15:27:23 nodejs-200 iscsid: connect to 10.100.129.18:3260 failed (No route to host)
Jul  4 15:27:27 nodejs-200 iscsid: connect to 10.100.129.18:3260 failed (No route to host)
Jul  4 15:27:31 nodejs-200 iscsid: connect to 10.100.129.18:3260 failed (No route to host)
Jul  4 15:27:35 nodejs-200 iscsid: connect to 10.100.129.18:3260 failed (No route to host)

Jul  4 15:27:38 nodejs-200 kubelet: E0704 15:27:38.064936    7274 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jul  4 15:27:39 nodejs-200 iscsid: connect to 10.100.129.18:3260 failed (No route to host)
kubelet报错
Jul  4 15:54:44 node3 kubelet: E0704 15:54:44.770766    2884 summary.go:92] Failed to get system container stats for "/system.slice/kubelet.service": failed to get cgroup stats for "/system.slice/kubelet.service": failed to get container info for "/system.slice/kubelet.service": unknown container "/system.slice/kubelet.service"
Jul  4 15:54:44 node3 kubelet: E0704 15:54:44.770816    2884 summary.go:92] Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container "/system.slice/docker.service"
Jul  4 15:54:53 node3 kubelet: E0704 15:54:53.840182    2884 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jul  4 15:54:54 node3 kubelet: E0704 15:54:54.783751    2884 summary.go:92] Failed to get system container stats for "/system.slice/kubelet.service": failed to get cgroup stats for "/system.slice/kubelet.service": failed to get container info for "/system.slice/kubelet.service": unknown container "/system.slice/kubelet.service"
Jul  4 15:54:54 node3 kubelet: E0704 15:54:54.783800    2884 summary.go:92] Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container "/system.slice/docker.service"
Jul  4 15:55:04 node3 kubelet: E0704 15:55:04.791674    2884 summary.go:92] Failed to get system container stats for "/system.slice/kubelet.service": failed to get cgroup stats for "/system.slice/kubelet.service": failed to get container info for "/system.slice/kubelet.service": unknown container "/system.slice/kubelet.service"
Jul  4 15:55:04 node3 kubelet: E0704 15:55:04.791725    2884 summary.go:92] Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container "/system.slice/docker.service"
Jul  4 15:55:13 node3 kubelet: E0704 15:55:13.840574    2884 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jul  4 15:55:14 node3 kubelet: E0704 15:55:14.804346    2884 summary.go:92] Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container "/system.slice/docker.service"
Jul  4 15:55:14 node3 kubelet: E0704 15:55:14.804397    2884 summary.go:92] Failed to get system container stats for "/system.slice/kubelet.service": failed to get cgroup stats for "/system.slice/kubelet.service": failed to get container info for "/system.slice/kubelet.service": unknown container "/system.slice/kubelet.service"


在安装`openebs`后，在一个节点使用该节点下的磁盘，但我将该节点移除，发现文件内容没有进行转移。