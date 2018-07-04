---
title: "Drone 进阶使用"
date: 2018-06-25T11:18:10+08:00
draft: false
subtitle: "带你深入认识drone的接口逻辑"
bigimg: [{src: "/img/drone_advance.jpg", desc: "Drone API"}]
tags: ["cicd", "interface", "docker", "golang"]
---
在之前的[Drone, 基于容器的CI/CD](http://localhost:1313/post/drone/)一文中，我们介绍了`drone`的概念，以及安装使用教程。`drone`是基于`docker`的分布式持续集成工具，采用`golang`开发，在[插件市场](http://plugins.drone.io/)中有十分丰富的插件可供选择。在实际使用过程中，我们可以需要将`drone`集成到已有的持续集成工具中，接下来，我们通过接口的官方文档了解接口逻辑，进行接口业务测试。

<!--more-->
# 准备工作
为了熟悉我们对`drone`的接口调用关系，我们安装[charles](https://www.charlesproxy.com/)抓包工具，将浏览器代理设置为`charles`代理进行抓包，会十分清晰地看到后台接口调用流程。

> [Charles的在线破解工具访问](https://www.zzzmode.com/mytools/charles/)，参考博客[如何设置代理教程](https://blog.csdn.net/qq_29674635/article/details/78837210)

![avatar](http://wx3.sinaimg.cn/mw690/0060lm7Tly1fswqmfuc3sj31hc0pxjtx.jpg)
这样，我们只要在流程浏览器操作，结合[drone接口文档](http://docs.drone.io/api-overview/)就很容易理解其接口逻辑。

# 接口逻辑
首先用户需要在`/login/form`接口登陆，当用户登陆成功后，会成功一个`token`，这个`token`在后续的所有服务相关接口中都需要使用，基于`OAUTH`认证。通过`/api/user`接口获取当前用户的信息

```json
{
	"id": 4,
	"login": "pan",
	"email": "dsfsdf@qq.com",
	"avatar_url": "https://secure.gravatar.com/avatar/983357cbf0bdf04674850e6705e0be57?d=identicon",
	"active": false,
	"synced": 1530609606
}
```
然后，通过可以`/api/user/feed?latest=true`接口获取到在`drone`本地保存的所有被激活的仓库
```json
[
	{
		"owner": "pan",
		"name": "demo",
		"full_name": "pan/demo"
	},
	{
		"owner": "panzhengming",
		"name": "demo",
		"full_name": "panzhengming/demo"
	}
]
```
当然，这是指所有公开的项目，对于其他用户的私有项目来说，当前用户是无法看到的，应该是根据`gitea`创建项目设置是否私有来判断的。用户通过`/api/user/repos?all=true&flush=true`来获取到所有可获取到的仓库信息
```json
[
	{
		"id": 3,
		"owner": "pan",
		"name": "demo",
		"full_name": "pan/demo",
		"avatar_url": "https://secure.gravatar.com/avatar/983357cbf0bdf04674850e6705e0be57?d=identicon",
		"link_url": "http://192.168.200.20:3000/pan/demo",
		"scm": "git",
		"clone_url": "http://192.168.200.20:3000/pan/demo.git",
		"default_branch": "master",
		"timeout": 60,
		"visibility": "public",
		"private": false,
		"trusted": false,
		"gated": false,
		"active": true,
		"allow_pr": true,
		"allow_push": true,
		"allow_deploys": false,
		"allow_tags": false,
		"last_build": 0,
		"config_file": ".drone.yml"
	},
	{
		"id": 1,
		"owner": "panzhengming",
		"name": "demo",
		"full_name": "panzhengming/demo",
		"avatar_url": "https://secure.gravatar.com/avatar/35de170fc7836ea645e1a7d7b307ff6e?d=identicon",
		"link_url": "http://192.168.200.20:3000/panzhengming/demo",
		"scm": "git",
		"clone_url": "http://192.168.200.20:3000/panzhengming/demo.git",
		"default_branch": "master",
		"timeout": 100,
		"visibility": "public",
		"private": false,
		"trusted": false,
		"gated": false,
		"active": true,
		"allow_pr": true,
		"allow_push": true,
		"allow_deploys": false,
		"allow_tags": false,
		"last_build": 0,
		"config_file": ".drone.yml"
	}
]
```
其中`flush`字段默认值是`false`，当我们设置为`true`后，自动从`gitea`中获取当前用户创建的项目列表，包括最新创建的项目。之后，通过`/api/repos/pan/demo`获取到单个项目的详情，这是一个`REST`接口
```json
{
	"id": 3,
	"owner": "pan",
	"name": "demo",
	"full_name": "pan/demo",
	"avatar_url": "https://secure.gravatar.com/avatar/983357cbf0bdf04674850e6705e0be57?d=identicon",
	"link_url": "http://192.168.200.20:3000/pan/demo",
	"scm": "git",
	"clone_url": "http://192.168.200.20:3000/pan/demo.git",
	"default_branch": "master",
	"timeout": 60,
	"visibility": "public",
	"private": false,
	"trusted": false,
	"gated": false,
	"active": true,
	"allow_pr": true,
	"allow_push": true,
	"allow_deploys": false,
	"allow_tags": false,
	"last_build": 0,
	"config_file": ".drone.yml"
}
```
当获取到项目的列表后，就可以通过该接口的`POST`形式将项目激活，如`/api/repos/pan/demo1`
```json
{
	"active": true
}
```
当然，我们也可以将项目禁止注销自动构建功能。当我们设置自动构建时，通过构建的接口来启动项目`/api/repos/pan/demo/builds/1`
```json
{
	"id": 3,
	"number": 3,
	"parent": 1,
	"event": "push",
	"status": "pending",
	"error": "",
	"enqueued_at": 1530612480,
	"created_at": 1530612480,
	"started_at": 0,
	"finished_at": 0,
	"deploy_to": "",
	"commit": "b1dc6e05b1a22ea18bc4f32e02cf30106077247c",
	"branch": "master",
	"ref": "refs/heads/master",
	"refspec": "",
	"remote": "",
	"title": "",
	"message": "init project\n",
	"timestamp": 1530612162,
	"sender": "pan",
	"author": "pan",
	"author_avatar": "https://secure.gravatar.com/avatar/983357cbf0bdf04674850e6705e0be57?d=identicon",
	"author_email": "dsfsdf@qq.com",
	"link_url": "http://192.168.200.20:3000/",
	"signed": false,
	"verified": true,
	"reviewed_by": "",
	"reviewed_at": 0,
	"procs": [
		{
			"id": 13,
			"build_id": 3,
			"pid": 1,
			"ppid": 0,
			"pgid": 1,
			"name": "",
			"state": "pending",
			"exit_code": 0
		},
		{
			"id": 14,
			"build_id": 3,
			"pid": 2,
			"ppid": 1,
			"pgid": 2,
			"name": "clone",
			"state": "pending",
			"exit_code": 0
		},
		{
			"id": 15,
			"build_id": 3,
			"pid": 3,
			"ppid": 1,
			"pgid": 3,
			"name": "build",
			"state": "pending",
			"exit_code": 0
		},
		{
			"id": 16,
			"build_id": 3,
			"pid": 4,
			"ppid": 1,
			"pgid": 4,
			"name": "publish",
			"state": "pending",
			"exit_code": 0
		},
		{
			"id": 17,
			"build_id": 3,
			"pid": 5,
			"ppid": 1,
			"pgid": 5,
			"name": "wechat",
			"state": "pending",
			"exit_code": 0
		},
		{
			"id": 18,
			"build_id": 3,
			"pid": 6,
			"ppid": 1,
			"pgid": 6,
			"name": "email",
			"state": "pending",
			"exit_code": 0
		}
	]
}
```
项目构建后，我们可以获取项目构建日志，先要通过`/api/repos/pan/demo/builds`获取该项目下的所有构建信息
```json
[
	{
		"id": 3,
		"number": 3,
		"parent": 1,
		"event": "push",
		"status": "failure",
		"error": "",
		"enqueued_at": 1530612480,
		"created_at": 1530612480,
		"started_at": 1530612480,
		"finished_at": 1530612487,
		"deploy_to": "",
		"commit": "b1dc6e05b1a22ea18bc4f32e02cf30106077247c",
		"branch": "master",
		"ref": "refs/heads/master",
		"refspec": "",
		"remote": "",
		"title": "",
		"message": "init project\n",
		"timestamp": 1530612162,
		"sender": "pan",
		"author": "pan",
		"author_avatar": "https://secure.gravatar.com/avatar/983357cbf0bdf04674850e6705e0be57?d=identicon",
		"author_email": "dsfsdf@qq.com",
		"link_url": "http://192.168.200.20:3000/",
		"signed": false,
		"verified": true,
		"reviewed_by": "",
		"reviewed_at": 0
	},
	{
		"id": 2,
		"number": 2,
		"parent": 1,
		"event": "push",
		"status": "failure",
		"error": "",
		"enqueued_at": 1530612464,
		"created_at": 1530612464,
		"started_at": 1530612464,
		"finished_at": 1530612471,
		"deploy_to": "",
		"commit": "b1dc6e05b1a22ea18bc4f32e02cf30106077247c",
		"branch": "master",
		"ref": "refs/heads/master",
		"refspec": "",
		"remote": "",
		"title": "",
		"message": "init project\n",
		"timestamp": 1530612162,
		"sender": "pan",
		"author": "pan",
		"author_avatar": "https://secure.gravatar.com/avatar/983357cbf0bdf04674850e6705e0be57?d=identicon",
		"author_email": "dsfsdf@qq.com",
		"link_url": "http://192.168.200.20:3000/",
		"signed": false,
		"verified": true,
		"reviewed_by": "",
		"reviewed_at": 0
	},
	{
		"id": 1,
		"number": 1,
		"parent": 0,
		"event": "push",
		"status": "failure",
		"error": "",
		"enqueued_at": 1530612162,
		"created_at": 1530612162,
		"started_at": 1530612162,
		"finished_at": 1530612175,
		"deploy_to": "",
		"commit": "b1dc6e05b1a22ea18bc4f32e02cf30106077247c",
		"branch": "master",
		"ref": "refs/heads/master",
		"refspec": "",
		"remote": "",
		"title": "",
		"message": "init project\n",
		"timestamp": 1530612162,
		"sender": "pan",
		"author": "pan",
		"author_avatar": "https://secure.gravatar.com/avatar/983357cbf0bdf04674850e6705e0be57?d=identicon",
		"author_email": "dsfsdf@qq.com",
		"link_url": "http://192.168.200.20:3000/",
		"signed": false,
		"verified": true,
		"reviewed_by": "",
		"reviewed_at": 0
	}
]
```
然后，通过`/api/repos/pan/demo/logs/2/2`获取到第二次构建的日志
```json
[
	{
		"proc": "clone",
		"pos": 0,
		"out": "+ git init\n"
	},
	{
		"proc": "clone",
		"pos": 1,
		"out": "Initialized empty Git repository in /drone/src/192.168.200.20/pan/demo/.git/\n"
	},
	{
		"proc": "clone",
		"pos": 2,
		"out": "+ git remote add origin http://192.168.200.20:3000/pan/demo.git\n"
	},
	{
		"proc": "clone",
		"pos": 3,
		"out": "+ git fetch --no-tags origin +refs/heads/master:\n"
	},
	{
		"proc": "clone",
		"pos": 4,
		"out": "From http://192.168.200.20:3000/pan/demo\n"
	},
	{
		"proc": "clone",
		"pos": 5,
		"out": " * branch            master     -> FETCH_HEAD\n"
	},
	{
		"proc": "clone",
		"pos": 6,
		"out": " * [new branch]      master     -> origin/master\n"
	},
	{
		"proc": "clone",
		"pos": 7,
		"out": "+ git reset --hard -q b1dc6e05b1a22ea18bc4f32e02cf30106077247c\n"
	},
	{
		"proc": "clone",
		"pos": 8,
		"out": "+ git submodule update --init --recursive\n"
	}
]
```
以上是`drone`的构建主要接口的流程，我们可以根据自己的需要进行二次开发，集成我们的`PASS`平台上。对于其他的接口以及整个业务流程待整理一个产品方案文档，进行源码修改。

# 书籍推荐

- [持续交付-发布可靠软件的系统方法](http://vdisk.weibo.com/s/uwMdY7-7n-mkh)
- [微服务设计](http://www.downcc.com/soft/301326.html)