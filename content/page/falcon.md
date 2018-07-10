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
项目1： 猎鹰行动
描述：这是基于kubernetes的数据中心计划，实现一个容器管理平台，主要功能有个人云主机、编译打包、部署、自动化测试、自动部署、监控，实现DevOps流程，同时对于所有流程节点都是可视化的，有指标可以进行度量。平台是Self-hosting，也就是所有组件都是在kubernetes中运行进行托管，自己托管自己。主要模块有

- LD权限
- OA管理
- IM交流
- SQ扫描
- CI集成
- TS测试
- CD交付

整个平台的组成软件有kubernetes、ldap、gitea、drone、redmine、harbor、Prometheus、robotframework、docker、hugo、minio、webhook、rocket.chat、hubot、docker-mailserver、API Blueprint、Review Board、SonarQube、statusok，其中核心部分是kubernetes，通过ldap的权限控制来划分用户权限，整个项目采用REST风格来编写接口，通过drone和gitea来实现自动编译打包以及安装，harbor充当镜像仓库。当处理完毕后，调用robotframework组件来自动跑用例，只有用例全部通过后，才会交给下一步的测试工作，当编写新功能接口测试用例后，redmine记录bug问题，等问题解决完毕后，重新执行上述工作来全量回归。这样就表示测试通过，将升级文档交付给运维来升级。通过Prometheus来实现基础设施以及微服务状态的监控，同时搭配statusok来探测服务可用性，更深层次，当业务涉及的地点十分广，通过搭建分布式业务探测系统来检查服务的可用性，可以搭配在树莓派上面，不需要过多设备。

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

项目二：数字AI家庭
描述：这是关于一个AI的故事，与智能家居的深度结合。家庭有个智能管家，对于家庭内所有服务进行托管，包括购物消费、水
- EM邮件
- PM管理
- PJ代码
电气费、家庭灯光、智能设备、数字影视、家庭种植等进行管理。