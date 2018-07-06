---
title: "k8s dashborad"
date: 2018-06-28T15:36:14+08:00
draft: false
subtitle: "带你安装kubernetes的web使用面板"
bigimg: [{src: "/img/dashboard.jpg", desc: "dashboard   "}]
tags: ["kubernetes", "web"]
---
[dashboard](https://github.com/kubernetes/dashboard)是`kubernetes`官方的一个web项目，通过在`web`界面上展示集群的相关信息，以及直接在`web`界面上操作集群资源，降低`kubernetes`的使用复杂，让更多人更容易地使用`kubernetes`。以下记录在集群安装`dashboard`，以及处理相关报错的过程。对于`kubernetes`的安装参考[Kubernetes, 容器调度编排框架](Kubernetes, 容器调度编排框架)一文。

<!--more-->

# dashboard安装
官方提供有安装的`kubernetes-dashboard.yaml`，我们将文件下载到本地目录，自行准备好安装文件所需要的`google`镜像，然后将安装文件的镜像地址修改为本地的私有镜像仓库地址，接下来只需要执行`kubelet create -f kubernetes-dashboard.yaml`即可，文件内容如下
```shell
[root@master2 dashboard]# cat kubernetes-dashboard.yaml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Configuration to deploy release version of the Dashboard UI compatible with
# Kubernetes 1.8.
#
# Example usage: kubectl create -f <this_file>

# ------------------- Dashboard Secret ------------------- #

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque

---
# ------------------- Dashboard Service Account ------------------- #

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Role & Role Binding ------------------- #

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Deployment ------------------- #

kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: example.com/google_containers/kubernetes-dashboard-amd64:v1.8.3 # 修改成私有镜像仓库地址
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
          - --heapster-host=http://heapster
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      # Comment the following tolerations if Dashboard must not be deployed on master
      #tolerations:
      #- key: node-role.kubernetes.io/master
      #  effect: NoSchedule

---
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
[root@master2 dashboard]# kubectl create -f kubernetes-dashboard.yaml
```
执行完毕，通过`kubelet get pods -n kube-system`会发现`dashboard`容器已经运行在一个节点中。接下来，我们需要创建一个可以登陆的用户，默认创建一个管理员用户，准备以下两个文件
```shell
[root@master2 user]# cat admin-role.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
[root@master2 user]# cat admin-user.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
[root@master2 user]# kubectl create -f admin-role.yml 
[root@master2 user]# kubectl creatge -f admin-user.yml 
```
上面两个文件的内容是创建一个`admin-user`的用户，赋予该用户管理员的权限。在该用户下我们可以执行集群的任何操作，包括查看资源、创建、删除等。
# kubernetes监控
`dashboard`容器不带有监控集群资源的功能，所以引入了`heapster`、`influxdb`、`grafana`三个组件，负责收取集群资源以及展示。从[官方项目](https://github.com/kubernetes/heapster/tree/master/deploy/kube-config/influxdb)下面下载这三个组件的安装文件到本地，如下
```shell
[root@master2 monitor]# l
total 12
-rw-r--r-- 1 root root 2300 May 30 16:56 grafana.yaml
-rw-r--r-- 1 root root 1385 Jul  4 15:11 heapster.yaml
-rw-r--r-- 1 root root  984 May 30 17:00 influxdb.yaml

```
需要修改下官方的`heapster.yaml`，将`heapster`用户赋予管理员权限，添加内容如下
```shell
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
```
同理，修改这三个组件的镜像为本地镜像仓库地址，默认是从`google`镜像仓库中心下载，这是在国内无法直接下载的。直接执行`kubectl create -f .`安装三个组件，执行完毕后，等待一段时间我们就会在`kube-system`命令空间看到`dashboard`运行的服务容器
```shell
[root@master2 monitor]# kubectl get pods -o wide -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE       IP                NODE
coredns-65dcdb4cf-g7fm9                 1/1       Running   0          2h        10.244.0.22       master1
heapster-5cd864cd6-zxvqb                1/1       Running   0          1h        10.244.3.6        node4
kube-apiserver-master1                  1/1       Running   0          2h        192.168.200.22    master1
kube-apiserver-master2                  1/1       Running   0          2h        192.168.200.21    master2
kube-controller-manager-master1         1/1       Running   0          2h        192.168.200.22    master1
kube-controller-manager-master2         1/1       Running   0          2h        192.168.200.21    master2
kube-router-bvb25                       1/1       Running   0          2h        192.168.100.111   node5
kube-router-hthj8                       1/1       Running   0          2h        192.168.100.150   node4
kube-router-rbqv7                       1/1       Running   0          2h        192.168.200.21    master2
kube-router-vx8rc                       1/1       Running   0          2h        192.168.200.22    master1
kube-router-wbjrx                       1/1       Running   0          1h        192.168.200.157   node6
kube-router-xjk5x                       1/1       Running   0          2h        192.168.200.142   node3
kube-router-zv62t                       1/1       Running   0          1h        192.168.201.2     node2
kube-scheduler-master1                  1/1       Running   0          2h        192.168.200.22    master1
kube-scheduler-master2                  1/1       Running   0          2h        192.168.200.21    master2
kubernetes-dashboard-86dd476cc8-cfdx9   1/1       Running   0          1h        10.244.3.7        node4
monitoring-grafana-79fd5987c8-wcrnr     1/1       Running   0          1h        10.244.3.5        node4
monitoring-influxdb-854d649fd4-xk9cq    1/1       Running   0          1h        10.244.4.3        node5
```
# 登陆dashboard ui
在上面的步骤中我们已经成功安装好`dashboard`所有组件，对于`web`界面的访问地址我们可以通过两种方式访问，代理访问和`NodePort`访问。
### 代理访问
在`maser`机器上输入`kubelet proxy`，设置本地代理与`apiserver`通信，然后我们就可以访问地址`http://localhost:8001/ui`，需要注意的是这只能访问本地地址，我们需要修改代理的`IP`地址为`master`的外网网卡地址，如下
```shell
kubectl proxy --address='192.168.137.100' --port=8086 --accept-hosts='^*$'
```
在浏览器中输入`https://192.168.137.100`就会看到登陆界面，但我们发现这种方式比较繁琐，下面介绍通过`NodePort`方式访问。
### NodePort访问
在`master`机器上通过`kubelet get svc -n kube-system`来查看集群的`services`，找到`kubernetes-dashboard`，输入`kubectl edit svc kubernetes-dashboard -n kube-system`将`ClusterIP`修改成`NodePort`，最终结果如下
```shell
[root@master2 monitor]# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
heapster               ClusterIP   10.101.162.230   <none>        80/TCP          1h
kube-dns               ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP   2h
kubernetes-dashboard   NodePort    10.108.65.220    <none>        443:30202/TCP   1h
monitoring-grafana     ClusterIP   10.102.231.144   <none>        80/TCP          1h
monitoring-influxdb    ClusterIP   10.100.200.50    <none>        8086/TCP        1h
```
打开浏览器输入`https://$NodeIP:30202`来登陆`UI`界面，在出现登陆方式中选择令牌登陆方式，然后在`master`机器上通过以下命令获取登陆的`token`
```shell
[root@master2 monitor]#   
Name:         admin-user-token-wrwx8
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=9331fa6e-7f57-11e8-a4f7-525400bcf250

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXdyd3g4Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI5MzMxZmE2ZS03ZjU3LTExZTgtYTRmNy01MjU0MDBiY2YyNTAiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.QA0eMTRUGVimi12-9Zy-bCVdR2BSjhsB1IQCMyNHgvy7O3nLkQP4SW9x2n-U5CXQhfY401a9IyYoOTRy9X3Yf8fWVfHEipQtR1n-wJrE0Hfa8FCJlFMv7UPJ8MEqTKlu87QMtbvLNR4o4Q8KXwAC4fnOTq-KbN4rNR920CdamanP39vdBznVrLfRoq06k6fiwqpObqgePLBDt3w51vxKadEs5ZJ9zcSWFPJaa0lJLX2TNnlN8uyB9zLGiqxqSri8_41N5_9j9cX53JNv3ODK2Y861h-GHb8bZuyE9NTqG3LdbCYtIuCYnUDQmh-55TWFhVLihRD0d3-3BdLirNzxiw
```
输入该`token`点击登陆后，就会跳转`web`界面
![avater](http://wx2.sinaimg.cn/mw690/0060lm7Tly1fsxxvpzp98j31hb0nd40y.jpg)

# 