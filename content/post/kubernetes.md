---
title: "Kubernetes, 容器调度编排框架"
date: 2018-06-18T15:23:41+08:00
draft: false
subtitle: "带你认识云计算kubernetes"
bigimg: [{src: "/img/kubernetes.jpg", desc: "kubernetes"}]
tags: ["linux", "centos", "docker", "network", "cicd", "kubernetes", "golang"]
---
`kubernetes`是一种开源的容器编排系统，实现容器服务的自动部署、自动伸缩以及管理。在`kubernetes`中，会将组成应用的容器组合成一个逻辑单元以更易管理和发现。`kubernetes`积累了`Google`生产环境运行工作负载15年的经验，并吸收了来自社区的最佳想法和实践。
<!--more-->
### kubernetes功能介绍

- **自动化装箱**：在不牺牲可用性的条件下，基于容器对于资源的要求和约束自动部署容器。同时，为了提高利用率和节省更多资源，将关键和最佳工作量结合在一起。
- **自愈能力**：当容器失败时，会对容器进行重启；当所部署的`Node`节点有问题时，会对容器进行重新部署和重新调度；当容器未通过监控检查时，会关闭此容器；知道容器正常运行时，才对外提供服务。
- **水平扩容**：通过简单的命令、用户界面或基于`CPU`的使用情况，能够对应用进行扩容和缩容。
- **服务发现和负载均衡**：开发者不需要额外的服务发现机制，就能够基于`Kubernetes`进行服务发现和负载均衡。
- **自动发布和回滚**：`Kubernetes`能够程序化的发布应用和相关配置。如果发布有问题，`kubernetes`将能够回归发生的变更。
- **保密和配置管理**：在不需要重新构建镜像的情况下，可以部署和更新保密和应用配置。
- **存储编排**：自动挂接存储系统，这些存储可以来自本地、公共云提供商、网络存储(如NFS、ISCSI、Gluster、Ceph、Openebs等)。

### kubernetes环境配置
对于`kubernetes`的安装，官方提供了两种方式，二进制安装和容器(`kubeadm`)安装，对于二进制的安装难度较大，以下介绍通过`kubeadm`进行集群的安装。在使用`kubeadm`之前，我们需要修改一下系统配置，因为`kubernetes`是根据主机的`hostname`进行注册，所以我们将不同机器修改成不同的`hostname`；然后，由于`kubernetes`的`apiserver`通过机器的`10255`端口通信，为了防止端口被禁止，我们关闭防火墙以及加固；最后`kubernetes`的集群网络是通过配置主机的网卡，我们要开启内核网卡转包的模式。

> 以下操作是在 `centos`系统上操作，对于其他系统需要使用对应的命令

##### 修改hostname
```shell
[root@master1 ~]# hostnamectl set-hostname master1 # 修改成你想要修改的hostname，
[root@master1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain
192.168.200.22 master1 # 修改成对应的IP，其他机器
```
##### 关闭防火墙、加固
```shell
[root@master1 ~]# systemctl disable firewalld
[root@master1 ~]# systemctl stop firewalld
[root@master1 ~]# setenforce 0 # 临时关闭selinux，下面是永久关闭selinux
[root@master1 ~]# sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux
[root@master1 ~]# sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
[root@master1 ~]# sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux
[root@master1 ~]# sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config  
[root@master1 ~]# reboot -n 修改后需要重启机器
[root@master1 ~]# /usr/sbin/sestatus # 查看加固是否关闭，出现下面结果表示已经关闭
SELinux status:                 disabled
```
##### 开启网卡traffic
```shell
[root@master1 ~]# cat /etc/sysctl.conf
net.bridge.bridge-nf-call-ip6tables = 1 # 将这行添加到/etc/sysctl.conf
net.bridge.bridge-nf-call-iptables = 1 # 将这行添加到/etc/sysctl.conf
[root@master1 ~]# sysctl -p /etc/sysctl.conf # 执行该命令后，出现有刚才添加的配置项表示成功
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
[root@master1 ~]# modprobe br_netfilter # 如果出现No such file or directory，执行该命令后重新执行上面命令
```
这样，我们就处理一台机器环境，对于集群的其他机器都是同样处理。本次安装的机器列表如下

| hostname | IP | Extened |
| --- | --- | --- |
| master1 | 192.168.200.22 | keepalived |
| master2 | 192.168.200.21 | keepalived |
| node3   | 192.168.200.142|   计算节点  |
| VIP     | 192.168.200.240| 虚IP|
其中，`master1`和`master2`通过`keepalived`实现`apiserver`的高可用，接下来我们安装`keepalived`，对于其安装，在`master1`和`master2`机器上通过`yum`来进行安装，如下
```shell
[root@master1 ~]# yum install keepalived
```
接下来，设置将`master1`机器设置为主节点，`master2`设置为备节点，只要修改下`keepalived.conf`文件即可，以下是两台机器的配置文件内容
```shell
[root@master1 ~]# cat /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.200.240:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface br0 # 修改成对应网卡名称
    virtual_router_id 61
    # 主节点权重最高 依次减少
    priority 120
    advert_int 1
    #修改为本地IP
    mcast_src_ip 192.168.200.22
    nopreempt
    authentication {
        auth_type PASS
        auth_pass sqP05dQgMSlzrxHj
    }
    unicast_peer {
        #注释掉本地IP
         192.168.200.21
         #192.168.200.22
    }
    virtual_ipaddress {
        192.168.200.240/24
    }
    track_script {
        CheckK8sMaster
    }

}
[root@master2 deployment]# cat /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.200.240:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface br0 # 修改成对应网卡名称
    virtual_router_id 61
    # 主节点权重最高 依次减少
    priority 110
    advert_int 1
    #修改为本地IP
    mcast_src_ip 192.168.200.21
    nopreempt
    authentication {
        auth_type PASS
        auth_pass sqP05dQgMSlzrxHj
    }
    unicast_peer {
        #注释掉本地IP
        #192.168.200.21
         192.168.200.22
    }
    virtual_ipaddress {
        192.168.200.240/24
    }
    track_script {
        CheckK8sMaster
    }

}
```
然后，在两台机器上分别执行`systemctl enable keepalived;systemctl start keepalived`来启动`keepalived`。在这个过程中，我们可以查看日志来确认启动过程，日志内容如下
```shell
[root@master1 ~]# systemctl status keepalived -l
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since 三 2018-06-27 16:45:01 CST; 8s ago
  Process: 20043 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 20044 (keepalived)
   Memory: 1.4M
   CGroup: /system.slice/keepalived.service
           ├─20044 /usr/sbin/keepalived -D
           ├─20045 /usr/sbin/keepalived -D
           └─20046 /usr/sbin/keepalived -D

6月 27 16:45:03 master1 Keepalived_vrrp[20046]: Sending gratuitous ARP on br0 for 192.168.200.240
6月 27 16:45:03 master1 Keepalived_vrrp[20046]: Sending gratuitous ARP on br0 for 192.168.200.240
6月 27 16:45:03 master1 Keepalived_vrrp[20046]: Sending gratuitous ARP on br0 for 192.168.200.240
6月 27 16:45:03 master1 Keepalived_vrrp[20046]: Sending gratuitous ARP on br0 for 192.168.200.240
6月 27 16:45:08 master1 Keepalived_vrrp[20046]: Sending gratuitous ARP on br0 for 192.168.200.240
6月 27 16:45:08 master1 Keepalived_vrrp[20046]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on br0 for 192.168.200.240
6月 27 16:45:08 master1 Keepalived_vrrp[20046]: Sending gratuitous ARP on br0 for 192.168.200.240
6月 27 16:45:08 master1 Keepalived_vrrp[20046]: Sending gratuitous ARP on br0 for 192.168.200.240
6月 27 16:45:08 master1 Keepalived_vrrp[20046]: Sending gratuitous ARP on br0 for 192.168.200.240
6月 27 16:45:08 master1 Keepalived_vrrp[20046]: Sending gratuitous ARP on br0 for 192.168.200.240
[root@master2 deployment]# systemctl status keepalived
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2018-05-30 13:33:57 CST; 4 weeks 0 days ago
 Main PID: 16989 (keepalived)
   CGroup: /system.slice/keepalived.service
           ├─16989 /usr/sbin/keepalived -D
           ├─16990 /usr/sbin/keepalived -D
           └─16991 /usr/sbin/keepalived -D

Jun 27 16:03:04 master2 Keepalived_vrrp[16991]: Sending gratuitous ARP on br0 for 192.168.200.240
Jun 27 16:03:09 master2 Keepalived_vrrp[16991]: Sending gratuitous ARP on br0 for 192.168.200.240
Jun 27 16:03:09 master2 Keepalived_vrrp[16991]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on br0 for 192.168.200.240
Jun 27 16:03:09 master2 Keepalived_vrrp[16991]: Sending gratuitous ARP on br0 for 192.168.200.240
Jun 27 16:03:09 master2 Keepalived_vrrp[16991]: Sending gratuitous ARP on br0 for 192.168.200.240
Jun 27 16:03:09 master2 Keepalived_vrrp[16991]: Sending gratuitous ARP on br0 for 192.168.200.240
Jun 27 16:03:09 master2 Keepalived_vrrp[16991]: Sending gratuitous ARP on br0 for 192.168.200.240
Jun 27 16:45:02 master2 Keepalived_vrrp[16991]: VRRP_Instance(VI_1) Received advert with higher priority 120, ours 110
Jun 27 16:45:02 master2 Keepalived_vrrp[16991]: VRRP_Instance(VI_1) Entering BACKUP STATE
Jun 27 16:45:02 master2 Keepalived_vrrp[16991]: VRRP_Instance(VI_1) removing protocol VIPs.
```
在上面日志中，我们可以看到在`master1`上新增了一个虚`IP`，`master2`进入备节点中。在`master1`机器上输入`ip a`会看到一个新的网卡。最后，我们关闭`master1`机器上的`keepalived`，来确认虚`IP`正确漂移到`master2`机器上，在此不做详细记录。

### kubernetes集群安装
在上面的操作中，我们完成了`kubernetes`集群安装必要的环境配置。我们现在可以通过`kubeadm`来安装集群，主要需要三个工具`kubeadm`、`kubelet`、`kubectl`，对于这三者区别，`kubeadm`和`kubelet`是安装集群工具，`kubectl`是集群命令行工具，可以操作集群资源。接下来，我们通过`yum`来进行安装，需要配置一个[阿里云k8s源](https://mirrors.aliyun.com/kubernetes/yum/)，只需在`/etc/yum.repos.d`新增一个`k8s.repo`，内容如下
```shell
[root@master1 yum.repos.d]# cat k8s.repo
[kubernetes]
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
```
然后，运行`yum install kubeadm kubelet kubectl`进行安装，这是默认安装最新版本，而本次是演示`v1.9.6`版本的安装，所以输入以下命令安装
```shell
[root@master1 yum.repos.d]# yum install kubeadm-1.9.6 kubelet-1.9.6 kubectl-1.9.6
```
等待安装结束，上面三个工具就安装到本地上了。在其他机器上都执行一样操作，这样我们就差最后一步就可以安装集群了。由于`kubernetes`采用`etcd`作为数据库存储，所以我们还需要安装`etcd`。对于`etcd`的安装有单机和集群，本次由于资源有限，通过两台机器组成集群，分别安装在`master1`和`master2`上面。
##### etcd安装
从[etcd Release](https://github.com/coreos/etcd/releases/)中选择`v3.3.2`版本，下载到本地后将二进制文件放到`/usr/bin`中。对于`etcd`集群有加密和不加密安装，本次是通过加密安装，对于加密工具，官方推荐采用`cfssl`来进行加密。下载工具到本地
```shell
[root@master1 ssl]# mkdir /opt/ssl;cd /opt/ssl/
[root@master1 ssl]# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O cfssl
[root@master1 ssl]# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O cfssljson
[root@master1 ssl]# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O cfssl-certinfo
[root@master1 ssl]# chmod +x ./* # 赋权限
[root@master1 ssl]# export PATH=.:$PATH # 将当前目录临时加到系统环境变量中
```
##### 配置`CA`文件
```shell
[root@master etcd]# cat ca-config.json
{
"signing": {
"default": {
  "expiry": "8760h"
},
"profiles": {
  "kubernetes": {
    "usages": [
        "signing",
        "key encipherment",
        "server auth",
        "client auth"
    ],
    "expiry": "8760h"
  }
}
}
}
[root@master etcd]# cat ca-csr.json
{
"CN": "kubernetes",
"key": {
"algo": "rsa",
"size": 2048
},
"names": [
{
  "C": "CN",
  "ST": "BeiJing",
  "L": "BeiJing",
  "O": "k8s",
  "OU": "System"
}
]
}
```
##### 生成CA证书和私钥
```shell
[root@master1 ssl]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
[root@master1 ssl]# ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```
##### 创建etcd证书签名请求
```shell
[root@master1 ssl]# cat etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.200.22", # 修改成自己的IP
    "192.168.200.21" # 修改成自己的IP
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```
##### 生成etcd证书和私钥
```shell
[root@master1 ssl]# cfssl gencert -ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=kubernetes etcd-csr.json | cfssljson -bare etcd
[root@master1 ssl]# ls etcd*
etcd  etcd.csr  etcd-csr.json  etcdctl  etcd-key.pem  etcd.pem
[root@master1 ssl]# mkdir -p /etc/etcd/ssl /var/lib/etcd
[root@master1 ssl]# cp etcd.pem etcd-key.pem  ca.pem /etc/etcd/ssl/ # 将制作的证书拷贝到/etc/etcd/ssl
[root@master2 ssl]# scp -r root@192.168.200.22:/etc/etcd/ssl /etc/etcd # 在master2机器上拷贝master1制作的证书到本地目录
```
当`etcd`集群证书创建成功后，我们配置`etcd.service`服务文件，如下所示
```shell
[root@master1 ssl]# cat /etc/systemd/system/etcd.service # 新建etcd.service文件，然后按照下面格式修改内容
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
Environment=NODE_NAME=master1
Environment=NODE_IP=192.168.200.22 # 改成本机IP，修改后删除注释
Environment=ETCD_NODES=master1=https://192.168.200.22:2380,master2=https://192.168.200.21:2380 # 改成集群的IP，修改后删除注释
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
        --name=${NODE_NAME} \
        --cert-file=/etc/etcd/ssl/etcd.pem \
        --key-file=/etc/etcd/ssl/etcd-key.pem \
        --peer-cert-file=/etc/etcd/ssl/etcd.pem \
        --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
        --trusted-ca-file=/etc/etcd/ssl/ca.pem \
        --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
        --initial-advertise-peer-urls=https://${NODE_IP}:2380 \
        --listen-peer-urls=https://${NODE_IP}:2380 \
        --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \
        --advertise-client-urls=https://${NODE_IP}:2379 \
        --initial-cluster-token=etcd-cluster-0 \
        --initial-cluster=${ETCD_NODES} \
        --initial-cluster-state=new \
        --data-dir=/var/lib/etcd/
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
[root@master2 yum.repos.d]# cat /etc/systemd/system/etcd.service # 改成本机IP，修改后删除注释
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
Environment=NODE_NAME=master2
Environment=NODE_IP=192.168.200.21
Environment=ETCD_NODES=master1=https://192.168.200.22:2380,master2=https://192.168.200.21:2380 # 改成集群的IP，修改后删除注释
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
        --name=${NODE_NAME} \
        --cert-file=/etc/etcd/ssl/etcd.pem \
        --key-file=/etc/etcd/ssl/etcd-key.pem \
        --peer-cert-file=/etc/etcd/ssl/etcd.pem \
        --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
        --trusted-ca-file=/etc/etcd/ssl/ca.pem \
        --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
        --initial-advertise-peer-urls=https://${NODE_IP}:2380 \
        --listen-peer-urls=https://${NODE_IP}:2380 \
        --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \
        --advertise-client-urls=https://${NODE_IP}:2379 \
        --initial-cluster-token=etcd-cluster-0 \
        --initial-cluster=${ETCD_NODES} \
        --initial-cluster-state=new \
        --data-dir=/var/lib/etcd/
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

> 对于`etcd.service`说明，参考 [centos服务创建](https://blog.csdn.net/zhangxtn/article/details/50462008)

最后，我们在`master1`和`master2`机器分别输入`systemctl daemon-reload
;system enable etcd;system start etcd`来启动`etcd`集群，在命令执行完毕后，通过`systemctl status etcd -l`查看`etcd`服务状态日志，如下表示正常
```shell
[root@master1 etcd]# systemctl status etcd -l
● etcd.service - Etcd Server
   Loaded: loaded (/etc/systemd/system/etcd.service; disabled; vendor preset: disabled)
   Active: active (running) since 三 2018-06-27 17:32:38 CST; 17s ago
     Docs: https://github.com/coreos
 Main PID: 20456 (etcd)
   Memory: 6.4M
   CGroup: /system.slice/etcd.service
           └─20456 /usr/bin/etcd --name=master1 --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem --peer-cert-file=/etc/etcd/ssl/etcd.pem --peer-key-file=/etc/etcd/ssl/etcd-key.pem --trusted-ca-file=/etc/etcd/ssl/ca.pem --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem --initial-advertise-peer-urls=https://192.168.200.22:2380 --listen-peer-urls=https://192.168.200.22:2380 --listen-client-urls=https://192.168.200.22:2379,http://127.0.0.1:2379 --advertise-client-urls=https://192.168.200.22:2379 --initial-cluster-token=etcd-cluster-0 --initial-cluster=master1=https://192.168.200.22:2380,master2=https://192.168.200.21:2380 --initial-cluster-state=new --data-dir=/var/lib/etcd/

6月 27 17:32:38 master1 etcd[20456]: raft.node: 87db4208f2c1d75 elected leader 87db4208f2c1d75 at term 6
6月 27 17:32:38 master1 etcd[20456]: published {Name:master1 ClientURLs:[https://192.168.200.22:2379]} to cluster d5e782bf1188f850
6月 27 17:32:38 master1 etcd[20456]: ready to serve client requests
6月 27 17:32:38 master1 etcd[20456]: ready to serve client requests
6月 27 17:32:38 master1 etcd[20456]: serving client requests on 192.168.200.22:2379
6月 27 17:32:38 master1 etcd[20456]: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
6月 27 17:32:38 master1 systemd[1]: Started Etcd Server.
6月 27 17:32:38 master1 etcd[20456]: setting up the initial cluster version to 3.3
6月 27 17:32:38 master1 etcd[20456]: set the initial cluster version to 3.3
6月 27 17:32:38 master1 etcd[20456]: enabled capabilities for version 3.3

[root@master2 etcd]# systemctl status etcd -l
● etcd.service - Etcd Server
   Loaded: loaded (/etc/systemd/system/etcd.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2018-06-27 17:32:38 CST; 9s ago
     Docs: https://github.com/coreos
 Main PID: 24108 (etcd)
   Memory: 7.7M
   CGroup: /system.slice/etcd.service
           └─24108 /usr/bin/etcd --name=master2 --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem --peer-cert-file=/etc/etcd/ssl/etcd.pem --peer-key-file=/etc/etcd/ssl/etcd-key.pem --trusted-ca-file=/etc/etcd/ssl/ca.pem --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem --initial-advertise-peer-urls=https://192.168.200.21:2380 --listen-peer-urls=https://192.168.200.21:2380 --listen-client-urls=https://192.168.200.21:2379,http://127.0.0.1:2379 --advertise-client-urls=https://192.168.200.21:2379 --initial-cluster-token=etcd-cluster-0 --initial-cluster=master1=https://192.168.200.22:2380,master2=https://192.168.200.21:2380 --initial-cluster-state=new --data-dir=/var/lib/etcd/

Jun 27 17:32:38 master2 etcd[24108]: 22726779ccf8c813 [logterm: 1, index: 2, vote: 0] cast MsgVote for 87db4208f2c1d75 [logterm: 1, index: 2] at term 6
Jun 27 17:32:38 master2 etcd[24108]: raft.node: 22726779ccf8c813 elected leader 87db4208f2c1d75 at term 6
Jun 27 17:32:38 master2 etcd[24108]: published {Name:master2 ClientURLs:[https://192.168.200.21:2379]} to cluster d5e782bf1188f850
Jun 27 17:32:38 master2 etcd[24108]: ready to serve client requests
Jun 27 17:32:38 master2 etcd[24108]: ready to serve client requests
Jun 27 17:32:38 master2 etcd[24108]: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
Jun 27 17:32:38 master2 etcd[24108]: serving client requests on 192.168.200.21:2379
Jun 27 17:32:38 master2 systemd[1]: Started Etcd Server.
Jun 27 17:32:38 master2 etcd[24108]: set the initial cluster version to 3.3
Jun 27 17:32:38 master2 etcd[24108]: enabled capabilities for version 3.3
```
当`etcd`集群启动正常后，我们还不能直接来安装`k8s`集群，由于`kubeadm`启动的镜像默认是从`google`镜像中心下载，国内是无法直接下载的，我们需要根据[kubeadm 镜像列表](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)下载对应镜像到本地的镜像仓库。当我们将所有镜像都下载到本地后，只需要修改`kubelet`配置文件，设置启动时从本地镜像仓库下载所需的镜像，如下
```shell
[root@master2 kubelet.service.d]# cat /etc/systemd/system/kubelet.service.d/20-pod-infra-image.conf # 在该目录新建20-pod-infra-image.conf文件
[Service]
Environment="KUBELET_EXTRA_ARGS=--pod-infra-container-image=example.com/google_containers/pause-amd64:3.0" # 将example.com修改成本地镜像仓库地址，格式对应

[root@master2 kubelet.service.d]# cat /etc/systemd/system/kubelet.service.d/20-pod-infra-image.conf # 修改kubelet的cgroup类型与docker的一致
[Service]
Environment="KUBELET_EXTRA_ARGS=--pod-infra-container-image=dhub.juxinli.com/google_containers/pause-amd64:3.0"
[root@master2 kubelet.service.d]# cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs" # 修改成docker驱动类型，通过docker info来查看
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS
```
> 集群所有主机都需要安装上面设置镜像仓库地址，对于镜像仓库的安装，参考 [vmare harbor企业镜像仓库](https://github.com/vmware/harbor)

当完成上面的操作后，我们还是不可以马上安装`k8s`集群，还需要编写`kubeadm`启动配置项文件`config.yml`，如下所示
```shell
[root@master1 kubernetes]# touch config.yml
[root@master1 kubernetes]# cat config.yml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
etcd:
  endpoints:
  - https://192.168.200.22:2379 # 修改成本机的IP，然后删除注释
  - https://192.168.200.21:2379 # 修改成本机的IP，然后删除注释
  caFile: /etc/etcd/ssl/ca.pem
  certFile: /etc/etcd/ssl/etcd.pem
  keyFile: /etc/etcd/ssl/etcd-key.pem
  dataDir: /var/lib/etcd
networking:
  podSubnet: 10.244.0.0/16 # 这是集群设定POD的子网地址，必须保证与接下来安装的网络模型中设定的子网地址一样
kubernetesVersion: 1.9.6
api:
  advertiseAddress: "192.168.200.240" # 修改成keepalived设置的虚IP，然后删除注释
token: "b99a00.a144ef80536d4344"
tokenTTL: "0s"
apiServerCertSANs:
- master1
- master2
- 192.168.200.22 # 修改成本机的IP，然后删除注释
- 192.168.200.21 # 修改成本机的IP，然后删除注释
- 192.168.200.240 # 修改成keepalived设置的虚IP，然后删除注释
featureGates:
  CoreDNS: true # 更换kube-dns为coredns
imageRepository: "example.com/google_containers" # 修改成本地镜像仓库地址，然后删除注释
```
最后的最后，在`master1`机器的`config.yml`目录下，运行`kubeadm init --config config.yml`，等待结束，会看到成功安装集群的提示，如下
```shell
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token b99a00.a144ef80536d4344 192.168.200.240:6443 --discovery-token-ca-cert-hash sha256:e740d9bae6699ee24516eaeea582abb3e01b0621bf36818def3774c0447e1bf3
```

> 当忘记保存上面的 `kubeadm join`信息，当想要新加一个节点，可以通过在 `master`机器上输入 `kubeadm token generate;kubeadm token create <generated-token> --print-join-command --ttl=0`来查看

接下来，在`master2`机器上将`master1`的`config.yml`文件和`/etc/kubernetes/pki`目录下面文件拷贝到本地的`/etc/kubernetes/pki`中，然后同样运行之前的命令`kubeadm init --config /path/to/config.yml`(替换成对应的目录)，命令执行结束后同样会看到上面的日志信息，在`node3`节点机器上运行来加入显示的`kubeadm join`一行命令，这会将`node3`机器加入到集群中。在`master1`机器上通过`kubectl`来查看集群状态，如下所示
```shell
[root@master1 kubernetes]# kubectl get node
NAME      STATUS     ROLES     AGE       VERSION
master1   NotReady   master    7m        v1.9.6
master2   NotReady   master    2m        v1.9.6
node3     NotReady   <none>    44s       v1.9.6
```

> 根据 `kubeadm init`命令显示的提示，正确配置 `kubelet`

出现`NotReady`表示集群还没安装网络模型，现在还无法使用，对于`kubernetes`网络模型，有`fannel`、`casio`、`kube-router`等，本次选择`kube-router`类型，参考[kube-router安装说明](https://github.com/cloudnativelabs/kube-router/blob/master/docs/kubeadm.md)，下载对应的安装文件后，执行以下命令
```shell
[root@master1 add-on]# kubectl apply -f kubeadm-kuberouter.yaml
[root@master1 add-on]# kubectl apply -f kubeadm-kuberouter-all-features.yaml
[root@master1 add-on]# kubectl -n kube-system delete ds kube-proxy
[root@master1 add-on]# docker run --privileged --net=host example.com/google_containers/kube-proxy-amd64:v1.9.6 kube-proxy --cleanup
```
> 对于 `kube-router`网络，内核需要支持 `ipvs`，执行 `lsmod|grep ip_vs`来查看当前机器是否支持，有输出则表示支持，没有的话执行 `modprobe  ip_vs`来手动加载  

最后的最后，我们通过`kubectl`来查看集群状态，集群节点已经全部`Ready`，如下
```shell
[root@master1 add-on]# kubectl get node
NAME      STATUS    ROLES     AGE       VERSION
master1   Ready     master    17m       v1.9.6
master2   Ready     master    13m       v1.9.6
node3     Ready     <none>    11m       v1.9.6
[root@master2 helm]# kubectl get cs # 查看集群状态
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```
接下来，我们来创建一个应用来感受`kubernetes`的使用，首先，准备一个`nginx.yml`文件，如下
```shell
[root@master2 deployment]# cat nginx.yml # 创建一个deployment类型的ngixn服务
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 10
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
[root@master2 deployment]# kubectl create -f nginx.yml
deployment "nginx-deployment" created
[root@master2 deployment]# kubectl get pods  -o wide # 查看刚才命令的结果，会看到有10pods创建
NAME                               READY     STATUS    RESTARTS   AGE       IP             NODE
nginx-deployment-9985f488c-7g44s   1/1       Running   0          30s       10.244.2.139   node3
nginx-deployment-9985f488c-8vrfj   1/1       Running   0          30s       10.244.2.142   node3
nginx-deployment-9985f488c-br9pb   1/1       Running   0          30s       10.244.2.144   node3
nginx-deployment-9985f488c-cc7xf   1/1       Running   0          30s       10.244.2.143   node3
nginx-deployment-9985f488c-gm4xs   1/1       Running   0          30s       10.244.2.136   node3
nginx-deployment-9985f488c-h5sjt   1/1       Running   0          30s       10.244.2.145   node3
nginx-deployment-9985f488c-md2cr   1/1       Running   0          30s       10.244.2.141   node3
nginx-deployment-9985f488c-tnq4m   1/1       Running   0          30s       10.244.2.140   node3
nginx-deployment-9985f488c-twrg5   1/1       Running   0          30s       10.244.2.138   node3
nginx-deployment-9985f488c-wnzn7   1/1       Running   0          30s       10.244.2.137   node3
[root@master2 deployment]# curl  10.244.2.145  # 访问一个pod的ip来确定功能正常
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
在安装过程中，对于长时间的没有反应，可以在对应机器上查看系统日志`tailf -f /var/log/messages`来确定报错原因，同时需要查阅官方文档。

> 本次安装参考了 [kubernetes1.9快速安装](https://www.kubernetes.org.cn/3536.html)一文

### kubernetes环境清理
当我们安装失败后，需要再次在机器上安装时，需要对于环境做一些清理操作，清理一些配置项来避免再次安装失败。主要有两步，一是清理`etcd`数据，二是清理`kubeadm`启动项
##### etcd清理
```shell
[root@master1 etcd]# systemctl stop etcd # 在etcd安装机器上都需要操作
[root@master1 etcd]# rm -rf /var/lib/etcd/member # 在etcd安装机器上都需要操作，之后重新启动etcd
```
##### kubeadm清理
```shell
kubeadm reset
rm -rf /root/.kube
ip link delete dummy0
ip link delete kube-dummy-if
ip link delete kube-bridge
ip link delete tunl0
iptables -F &&  iptables -X &&  iptables -F -t nat &&  iptables -X -t nat #清理路由
iptables -L -n
iptables --flush  
iptables -tnat --flush
systemctl restart docker.service # 重启docker
```

### 查看集群网络情况
安装`ipvsadm`来查看`kube-router`建立的网络连接情况
```shell
[root@node3 opt]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  node3:31109 rr
  -> 10.244.3.12:pcsync-https     Masq    1      0          0
TCP  node3:https rr persistent 10800
  -> 192.168.200.240:sun-sr-https Masq    1      0          0
TCP  node3:domain rr
  -> 10.244.0.21:domain           Masq    1      0          0
TCP  node3:http rr
  -> 10.244.3.10:us-cli           Masq    1      0          0
TCP  node3:http rr
  -> 10.244.3.11:hbci             Masq    1      0          0
TCP  node3:d-s-n rr
  -> 10.244.4.2:d-s-n             Masq    1      0          0
TCP  node3:https rr
  -> 10.244.3.12:pcsync-https     Masq    1      0          0
UDP  node3:domain rr
  -> 10.244.0.21:domain           Masq    1      0          0
```
安装`ipset`来查看请求相应情况
```shell
[root@node2 docker]# ipset --list
Name: kube-router-pod-subnets
Type: hash:net
Revision: 3
Header: family inet hashsize 1024 maxelem 65536 timeout 0
Size in memory: 17168
References: 2
Members:
10.244.0.0/24 timeout 0
10.244.2.0/24 timeout 0
10.244.1.0/24 timeout 0
10.244.3.0/24 timeout 0
10.244.4.0/24 timeout 0
10.244.5.0/24 timeout 0

Name: kube-router-node-ips
Type: hash:ip
Revision: 1
Header: family inet hashsize 1024 maxelem 65536 timeout 0
Size in memory: 16912
References: 1
Members:
192.168.201.2 timeout 0
192.168.100.150 timeout 0
192.168.200.22 timeout 0
192.168.200.142 timeout 0
192.168.100.111 timeout 0
192.168.200.21 timeout 0
```
### 移除集群节点
当节点出现故障时，我们需要将该节点移除，执行下面命令
```shell
[root@master2 bin]# kubectl drain node4 --delete-local-data --force --ignore-daemonsets
node "node4" cordoned
WARNING: Deleting pods with local storage: kubernetes-dashboard-86dd476cc8-cfdx9, monitoring-grafana-79fd5987c8-wcrnr; Ignoring DaemonSet-managed                                                                                         pods: kube-router-hthj8
pod "monitoring-grafana-79fd5987c8-wcrnr" evicted
pod "kubernetes-dashboard-86dd476cc8-cfdx9" evicted
pod "pvc-26038a45-7f5d-11e8-a4f7-525400bcf250-ctrl-545546857f-lxh8p" evicted
pod "heapster-5cd864cd6-zxvqb" evicted
node "node4" drained
```
### TODOS
- kubernetes的操作面板dashboard
- kubernetes的存储系统openebs
- kubernetes的包管理软件helm
- kubernetes的负载均衡器ingress
- kubernetes的集群监控prometheus


### 书籍推荐
- [Kubernetes权威指南第2版](http://www.3322.cc/soft/33984.html)