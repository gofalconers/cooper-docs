---
title: "Robotframe与持续集成"
date: 2018-06-25T10:49:02+08:00
draft: false
subtitle: "构思一种自动化用例管理与项目集成方案"
bigimg: [{src: "/img/robot_advanced.jpg", desc: "robot进阶使用"}]
tags: ["tester", "interface", "docker"]
---
在日常测试工作中，我们使用`robotframework`写好自动化用例后，我们是在本地执行自动化用例。这需要我们人工介入，对于持续集成，可以通过`jenkins`的`robotframework`插件来实现自动跑用例。但是，对于自动化环境，还需要我们手动安装完毕后，才能进行测试，以下记录当前通过`docker`来实现一种端到端的自动化测试方案。
<!--more-->
# 自动化测试的现状
在一般的项目迭代过程中，开发写好代码后，本地打包，测试通过后，会将代码何入到对应`git`版本中，之后，在`jenkins`的项目中配置对应的版本号，进行一次构建，最终会将代码更新到对应的环境中。之后，通知相关测试人员进行测试，当我们项目迭代几次后，我们会发现我们一直在重复之前的操作，如果中间出现一次流程错误，那么我们需要定位该错误的影响范围，可能会破坏环境配置导致下一次构建失败。上述场景的问题核心在于，环境是固定，用虚拟机来创建，没有对环境进行隔离。那么，我们就可以通过容器`docker`来实现环境隔离，同时十分方便进行环境的移植，在任何一台支持`docker`的机器上快速安装相同配置的环境，从而将我们从环境的安装工作中释放出来，做一些更有价值的工作。
# 自动化测试的方案
整个方案的核心是`jenkins`，用来编排整个流程，实现源码编译打包、编译镜像、环境安装、用例自动化执行、邮件通知等。
### jenkins安装
对于`jenkins`的安装，采用容器的形式运行，所以我们需要准备`jenkins`的`Dockerfile`，目前采用单节点，所以该`jenkins`需要准备`robotframework`和`docker`环境，`Dockerfile`如下
```shell
FROM openjdk:8u121-jdk-alpine
MAINTAINER panzhengming@qq.com
RUN mkdir /opt/app -p 
WORKDIR /opt/app
RUN wget http://mirrors.shu.edu.cn/jenkins/war-stable/2.121.1/jenkins.war -O jenkins.war
RUN sed -i 's#dl-cdn.alpinelinux.org#mirrors.aliyun.com#g' /etc/apk/repositories && apk update
# 根据https://github.com/liumiaocn/easypack/blob/master/containers/alpine/jenkins/Dockerfile来下载alpine系统依赖
# 可以在jenkins官网上查看对应版本的alpine系统依赖包https://www.archlinux.org/packages/community/any/jenkins/
RUN apk add --no-cache docker py-pip git openssh-client curl unzip bash ttf-dejavu coreutils
RUN mkdir /root/.pip -p
ADD pip.conf /root/.pip/pip.conf # pip.conf配置阿里云地址，加速pytho的包安装速度
RUN pip install robotframework robotframework-requests robotframework-databaselibrary robotframework-httplibrary
VOLUME [ "/root/.jenkins" ]
EXPOSE 8080 50000
CMD [ "java", "-jar", "jenkins.war", "-Djava.awt.headless=true" ]
```
然后在一台机器上通过`docker build -t panzhengming/jenkins .`来构建我们自己的`jenkins`镜像，如果还需要其他的一些工具，可以修改上述的`Dockefile`来安装。再准备一个`docker-compose.yml`来运行该镜像，如下
```shell
version: '2'

services:
  jenkins-master:
    image: dhub.juxinli.com/ci/jenkins
    ports:
      - 48080:8080
      - 65000:50000
    volumes:
      - ./jenkins:/root/.jenkins
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
```
这样，我们就准备好了自动化测试的核心部分，接下来是只需要配置源码编译步骤以及一些用例、邮件插件即可。
### 源码编译
对于不采用`docker`的形式来编译的项目，一般是准备一台虚拟机，安装`maven`编译工具进行编译。而对于`docker`来说，可以通过拉取一个`maven`镜像来进行编译生产生产物，以下是`java`项目的`Dockerfile`
```shell
FROM maven:3.5.3-jdk-8 as build
RUN mkdir /opt/manado_api
WORKDIR /opt/manado_api
COPY . /opt/manado_api
RUN mvn package -Dmaven.test.skip=true -P docker

FROM java:8u111-alpine
MAINTAINER panzhengming@example.com
RUN mkdir /opt/manado_api -p
WORKDIR /opt/manado_api
ARG version
EXPOSE 8080
COPY --from=build /opt/manado_api/target/{version}.jar /opt/manado_api
CMD [ "java", "-jar", "${version}.jar" ]
```
大概流程有两步，第一步是拉取一个`maven`镜像，将源码拷贝至`maven`镜像，通过`mvn`指定编译`docker`的配置文件，这个命令结束后，会在项目中生成对应的`jar`包；然后，在拉取一个`jdk`环境的镜像，将刚才的生成物，也就是`jar`包，拷贝至`jdk`镜像中，通过`java`命令启动项目。

> 对于其他项目，如 `golang`、`nodejs`等编程语言都是相同，首先拉取对应的编译环境镜像编译，编译好之后，拉取运行环境的镜像运行。

这样，我们在`jenkins`中配置一个项目，只需要在构建中一个`docker`编译镜像命令来生成镜像
```shell
docker build -t example.com/panzheng/test:${gitlabBranch#*Release} .
```

> 上面命令是通过 `gitlab`进行触发构建的，需要在 `gitlab`配置一个 `webhook`，参考 [gitlab的 webhook](https://www.cnblogs.com/kevingrace/p/6479813.html)

### jenkins插件
对于`jenkins`插件，自行在插件列表中搜索`robotframe`以及`email`插件安装，对于其使用教程，参考 [jenkins的robotframework安装教程](https://www.cnblogs.com/zz27zz/p/7366575.html)。

### 环境安装
当我们生成了对应的版本镜像后，我们就可以通过一些编排工具来完成环境自动安装、升级回滚等，目前有Swarm、Mesos以及Kubernetes，自行根据需求选择特定的工具。

### 深度思考
以上，我们通过使用`docker`、`jenkins`实现了一种自动化流程，而在实际实施过程中，需要与不同岗位的人进行沟通，需要对不同的人进行解释原理，对于没有`docker`使用的项目中，一般不推荐使用。设想一下，当开发不维护`Dockerfile`，在开发过程中更改了一些配置，这样就会造成整个流程的阻塞，出现级联故障。当我们完成`CI`后，需要考虑`CD`的流程，这个待以后再讨论。

> 对于 `robotframework`用例管理，不在本次的讨论范围内，对于大规模用例的执行也是一个需求点，需要完善本次的自动化方案。

# 推荐书籍

- [持续交付-发布可靠软件的系统方法](http://vdisk.weibo.com/s/uwMdY7-7n-mkh)
- [布道之道—引领团队拥抱技术创新](http://vdisk.weibo.com/s/zmzzSciGaVmNj)