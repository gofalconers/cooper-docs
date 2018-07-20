---
title: "Docker_advanced"
date: 2018-06-25T11:18:39+08:00
draft: false
subtitle:
bigimg: [{src: "",desc: ""}]
tags: []
---

<!--more-->
删除不运行的容器 docker container prune


在一台机器上突然出现docker无法重启，日志如下：
```shell
Jul 20 16:33:36 master1 dockerd: time="2018-07-20T16:33:36.475635135+08:00" level=info msg="libcontainerd: started new docker-containerd process" pid=29640
Jul 20 16:33:36 master1 dockerd: time="2018-07-20T16:33:36+08:00" level=info msg="starting containerd" module=containerd revision=cfd04396dc68220d1cecbe686a6cc3aa5ce3667c version=v1.0.2
Jul 20 16:33:36 master1 dockerd: time="2018-07-20T16:33:36+08:00" level=info msg="loading plugin "io.containerd.content.v1.content"..." module=containerd type=io.containerd.content.v1
Jul 20 16:33:36 master1 dockerd: time="2018-07-20T16:33:36+08:00" level=info msg="loading plugin "io.containerd.snapshotter.v1.btrfs"..." module=containerd type=io.containerd.snapshotter.v1
Jul 20 16:34:11 master1 dockerd: time="2018-07-20T16:34:11.490842351+08:00" level=warning msg="daemon didn't stop within 15 secs, killing it" module=libcontainerd pid=29640
Jul 20 16:34:11 master1 dockerd: Failed to connect to containerd: failed to dial "/var/run/docker/containerd/docker-containerd.sock": dial unix:///var/run/docker/containerd/docker-containerd.sock: timeout
Jul 20 16:34:11 master1 systemd: docker.service: main process exited, code=exited, status=1/FAILURE
Jul 20 16:34:11 master1 systemd: Failed to start Docker Application Container Engine.
Jul 20 16:34:11 master1 systemd: Unit docker.service entered failed state.
Jul 20 16:34:11 master1 systemd: docker.service failed.
Jul 20 16:34:11 master1 systemd: docker.service holdoff time over, scheduling restart.
Jul 20 16:34:11 master1 systemd: Starting Docker Application Container Engine...
Jul 20 16:34:11 master1 dockerd: time="2018-07-20T16:34:11.728783264+08:00" level=info msg="libcontainerd: started new docker-containerd process" pid=29690
Jul 20 16:34:11 master1 dockerd: time="2018-07-20T16:34:11+08:00" level=info msg="starting containerd" module=containerd revision=cfd04396dc68220d1cecbe686a6cc3aa5ce3667c version=v1.0.2
Jul 20 16:34:11 master1 dockerd: time="2018-07-20T16:34:11+08:00" level=info msg="loading plugin "io.containerd.content.v1.content"..." module=containerd type=io.containerd.content.v1
Jul 20 16:34:11 master1 dockerd: time="2018-07-20T16:34:11+08:00" level=info msg="loading plugin "io.containerd.snapshotter.v1.btrfs"..." module=containerd type=io.containerd.snapshotter.v1
```
删除/var/run/docker后，重新启动docker成功。