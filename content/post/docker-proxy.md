---
title: "Docker build 挂梯子"
date: 2023-09-21T22:27:00+08:00
lastmod: 2023-09-21T22:27:00+08:00
draft: false
tags: ["Docker"]
categories: ["Docker"]

---

> 国内的网络环境，懂得都懂。

Docker build的时候常常需要在镜像内使用包管理器安装一些对应的包，比如`gcc`/`g++`等等，由于独特的网络环境，不得不使用一些魔法手段。

常用的方法是通过`ENV`在dockerfile里直接修改`HTTP_PROXY`,`HTTPS_PROXY`,`GOPROXY`等变量为国内特有的地址，
但是对于想开放给世界范围使用的开源项目，镜像写死这些东西会对非CN地区用家造成一些困绕。

在不修改Dockerfile本身的情况下，可以使用`build-arg`在构建阶段注入环境变量，通过`--network host`将构建阶段的网路环境与本机打通。
这样就可以借助本地的魔法加速包管理器拉取速度了。

一个简单地例子：
`docker build --network host --build-arg http_proxy=http://127.0.0.1:7890 --build-arg https_proxy=http://127.0.0.1:7890 .`

当打包好镜像后，也可以使用这种方法完成需要拉取外网内容初始化部分的。
`docker run --network host -e http_proxy=http://127.0.0.1:7890 -e https_proxy=http://127.0.0.1:7890  ac9e557bd88d`

