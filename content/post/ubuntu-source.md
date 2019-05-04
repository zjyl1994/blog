---
title: "Ubuntu 18.04 必做操作"
date: 2018-10-12T07:00:00+08:00
lastmod: 2018-10-12T07:00:00+08:00
draft: false
tags: ["Ubuntu","笔记"]
categories: ["开发笔记"]

---

> 新开一个栏目，专门记录开发中常见的命令和操作

新装完Ubuntu Server之后要做几个操作，才能让他更好用:

1. 更新软件源
1. 安装必备软件
1. 设置系统代理

<!--more-->

首先我们要做的事是更新软件源，默认软件源在美国，拜天朝国情所赐，你科学上网依然很慢。

这里选用中科大的源，USTC的源国内各个地方速度都不错。

```bash
sudo vi /etc/apt/sources.list
```

打开的文件按I进入插入模式，在最前面加上如下内容:

```text
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
```

然后<kbd>ESC</kbd>:wq保存，再输入 `sudo apt-get update` 使修改的源生效。

下一步是安装常用软件，开发机上git和基础的编译工具得装，要不然缺一些什么以后麻烦。
```bash
sudo apt-get install -y git build-essential
```

设置系统级代理很简单，只需要设置两个环境变量：

http_proxy 和 https_proxy，为了保险 HTTP_PROXY 和 HTTPS_PROXY 也最好设置上。

格式如下：

```bash
export http_proxy=http://127.0.0.1.15000
export https_proxy=http://127.0.0.1.15000
export HTTP_PROXY=http://127.0.0.1.15000
export HTTPS_PROXY=http://127.0.0.1.15000
```

经过这一番设置基本上 Ubuntu 就可用了，根据需要安装 Python 、 Go 或者 PHP 环境使用即可。