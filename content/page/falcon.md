---
title: "DevOps"
date: 2018-06-26T10:47:01+08:00
draft: false
subtitle: "I can help you work happy"
bigimg: [{src: "/img/falcon.jpg", desc: "DevOps"}]
tags: ["devops"]
---
这是代号为`falcon`行动，主要是为了解决您正常开发过程中的基础设施、环境管理、开发工具以及生产管理工具等。
<!--more-->
项目1： 猎鹰行动，AI项目管理
描述：这是基于kubernetes的数据中心计划，实现一个容器管理平台，主要功能有个人云主机、编译打包、部署、自动化测试、自动部署、监控，实现DevOps流程，同时对于所有流程节点都是可视化的，有指标可以进行度量。平台是Self-hosting，也就是所有组件都是在kubernetes中运行进行托管，自己托管自己。主要模块有

- LD权限
- OA管理
- IM交流
- SQ扫描
- CI集成
- TS测试
- CD交付

整个平台的组成软件有kubernetes、ldap、gitea、drone、redmine、harbor、Prometheus、robotframework、docker、hugo、minio、webhook、rocket.chat、hubot、docker-mailserver、API Blueprint、Review Board、SonarQube、gruntjs、statusok，其中核心部分是kubernetes，通过ldap的权限控制来划分用户权限，整个项目采用REST风格来编写接口，通过drone和gitea来实现自动编译打包以及安装，harbor充当镜像仓库。当处理完毕后，调用robotframework组件来自动跑用例，只有用例全部通过后，才会交给下一步的测试工作，当编写新功能接口测试用例后，redmine记录bug问题，等问题解决完毕后，重新执行上述工作来全量回归。这样就表示测试通过，将升级文档交付给运维来升级。通过Prometheus来实现基础设施以及微服务状态的监控，同时搭配statusok来探测服务可用性，更深层次，当业务涉及的地点十分广，通过搭建分布式业务探测系统来检查服务的可用性，可以搭配在树莓派上面，不需要过多设备。

对于redmine项目管理安装插件有：

- [redmine_custom_reports](https://github.com/Restream/redmine_custom_reports)
- [redmine_issue_dynamic_edit](http://www.redmine.org/plugins/redmine_issue_dynamic_edit)
- [bitbucket_reference_redmine](http://www.redmine.org/plugins/bitbucket_reference_redmine)
- [redmine_dashboard](https://github.com/jgraichen/redmine_dashboard)
- [redmine_messenger](https://github.com/AlphaNodes/redmine_messenger)
- [redmine_jenkins](https://github.com/jbox-web/redmine_jenkins)
- [redmine_intouch](https://github.com/centosadmin/redmine_intouch)
- [redmine_issue_template](https://github.com/Restream/redmine_issue_template)
- [bug提交规范](https://zh.opensuse.org/openSUSE:%E6%8F%90%E4%BA%A4%E9%94%99%E8%AF%AF%E6%8A%A5%E5%91%8A)
- [notify_custom_users](https://github.com/Restream/notify_custom_users)
- [redmine-theme-gitmike](https://github.com/makotokw/redmine-theme-gitmike)

对于ldap权限管理，使用[go-ldap](https://github.com/go-ldap/ldap)开发一个ldap微服务来完成所有操作，默认提供一个管理员账号来分配所有人员的账号。这是为了方便用户的账号管理，制作一个sso单点的登陆，同时提供一个注册页面，实现一键注册的功能，只有管理员可以注册。接下来是关于具体的软件项目管理的内容，通过度量来进行管理，正如[架构即未来+现代企业可扩展的Web架构流程和组织](https://pan.baidu.com/s/1QWqEXp2msRg0qu3GAoDE7A)一书中关于的项目管理的核心，一切都是度量。项目的情况大概分为以下几个部分：

- 产品通过hugo编写需求，通过git进行管理，这样就将需求文档与版本进行关联起来，支持导出
- 开发通过gitea管理源码，通过hugo编写架构文档，API Blueprint编写接口文档，drone进行构建镜像，最后编写helm版本包
- 测试通过helm安装测试环境，robotframework编写用例，redmine提交bug，通过hugo编写测试方案，通过git进行管理，这样需求、接口、数据库文档、测试方案就统一关联
- 运维管理维护kubernetes环境，helm进行版本升级管理，Prometheus监控
- 项目通过ldap管理账号，沟通交流通过邮件以及rocket.chat，bug管理通过redmine进行维护，hubot自动化运维机器人

以上列出了主要的场景，接下来是度量分析，从不同职位的维度进行说明

- 产品发布版本的频率、文档编写、版本工时统计、收支统计
- 开发编写代码行数、构建通过率、版本bug数以及修复的时间
- 测试编写自动化用例数、提交的bug数
- 运维服务可用性，以及生产事故数，收集基础信息

对于一些流程进行规定:

- 产品文档清晰，有流程图，提供标准模板，可以修改指定
- 开发架构清晰，开发编码规范，数据库表记录，提供标准模板，可以修改指定
- 测试有测试方案，测试用例，提bug说明，提供标准模板，可以修改指定
- 运维提供运维手册、紧急事故修复流程，提供标准模板，可以修改指定
- 项目事故报告编写规划，通过hugo编写，每个项目都有一个事故清单，来进行事故还原，以及一些改进措施，由项目负责人编写，提供模板，纳入度量体系中，待确认责任归属在哪
- 对于工作外的其他事，由人事进行统一处理，纳入度量体系，选择一个OA系统进行管理

对于项目的安装，统一安装在kubernetes中，对于一些重要的应用进行HA配置、数据备份。

项目使用到的库：

- [go-ldap使用介绍](https://www.golang123.com/topic/1750)
- [logrus日志模块](https://github.com/sirupsen/logrus) 利用插件推送给日志系统
- [go-remine项目管理](github.com/mattn/go-redmine)
- [drone-go构建](github.com/drone/drone-go/drone)
- [go-gitea源码管理](github.com/go-gitea/go-sdk/gitea)

ldap使用的文章https://blog.csdn.net/dolphin_h/article/details/56349454
golang交叉编译工具https://github.com/mitchellh/gox
golang依赖管理工具https://github.com/kardianos/govendor
ssh的web终端https://github.com/liftoff/GateOne
ssh的web终端https://github.com/codetainerapp/codetainer
golang爬虫工具https://github.com/gocolly/colly
golang的worker工具https://github.com/benmanns/goworker
golang的私有网盘https://github.com/syncthing/syncthing
golang的消息推送https://github.com/nats-io/go-nats
golang的消息推送https://github.com/nsqio/nsq
golang的健康检查https://github.com/etherlabsio/healthcheck
golang的健康检查https://github.com/Talento90/go-health
golang的pdf工具https://github.com/jung-kurt/gofpdf
golang的模板工具https://github.com/valyala/quicktemplate
golang的selenium工具https://github.com/aandryashin/selenoid
golang的selenium工具https://github.com/aerokube/ggr
golang的类型检查https://github.com/asaskevich/govalidator
golang的job工具https://github.com/ajvb/kala
golang的线上接口监控https://github.com/sanathp/statusok

IM交流工具https://github.com/mattermost/
IM视频会议https://github.com/mattermost/mattermost-webrtc

项目二：数字AI家庭
描述：这是关于一个AI的故事，与智能家居的深度结合。家庭有个智能管家，对于家庭内所有服务进行托管，包括购物消费、水
- EM邮件
- PM管理
- PJ代码
电气费、家庭灯光、智能设备、数字影视、家庭种植等进行管理。

IOT开发框架
https://github.com/google/periph
https://github.com/Mainflux/mainflux
https://github.com/sensorbee/sensorbee
https://github.com/hybridgroup/gobot/
https://github.com/sensorbee/sensorbee
