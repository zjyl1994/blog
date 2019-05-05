---
title: "服务器 Docker 化笔记"
date: 2019-05-05T23:50:00+08:00
lastmod: 2019-05-06T00:12:00+08:00
draft: false
tags: ["Docker"]
categories: ["Docker"]

---

> 五一期间折腾了一下服务器的 Docker 化，踩了好多的坑，写文记录下来。
> 注意，这篇文章目前尚未完成。

# 起因

起因很简单，五一去玩了一圈，结果到处都是人，所以在家猫着总得折腾点什么。
在公司写东西日常使用 Docker，目前对 Docker 的操作也非常熟悉了，所以总想着把我的服务器 Docker 化

先说一下我的基础配置，Kimsufi 的 KS-7 主机，i3 处理器，8G 内存，2T 硬盘，100M 带宽。
内存很充裕，就可以放心的折腾 Docker 环境。事实上我最后弄完几个容器，实际服务器占用也就 500M - 700M 左右。

# 重装服务器

由于我的主机已经用了一阵，所以服务器里很多已经抛弃的东西，也有很多正在用的东西。首先要备份数据到其他地方，我还有其它的服务器，所以把有用的数据 rsync 送到别的服务器上。

Kimsufi 登入管理面板，重新安装最新版的 CentOS 7。配置 SSH Key，升级最新软件包和内核。Kimsufi 安装的时候要注意，一定要勾选 `use distribution kernel` 选项。
如果没有勾选这个选项，Kimsufi 会给你安装定制版的内核，我不太清楚定制版内核会带来什么优势，但是目前来看只会造成麻烦。所以一定要使用发行版自带的内核。

<!--more-->

## 升级内核

升级内核很简单，因为有秋水逸冰大佬写的[一键 BBR 脚本](https://teddysun.com/489.html)。他的一键 BBR 脚本本质上就是给你的服务器升级到 4.12 以上版本带 BBR 的内核。所以说拿来使用非常方便。

`wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh && chmod +x bbr.sh && ./bbr.sh`

一条命令，按照提示按y，最后会重启应用最新的内核。（目前能安装的最新内核是 Linux 5.0）

**未完待续**