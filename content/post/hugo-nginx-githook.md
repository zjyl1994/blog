---
title: "Hugo 博客折腾记"
date: 2019-11-06T22:42:00+08:00
lastmod: 2019-11-06T22:42:00+08:00
draft: false
tags: ["Hugo","Nginx","CentOS"]
categories: ["博客"]
---

> 五一期间折腾了一下服务器的 Docker 化，踩了好多的坑，写文记录下来。

> 别小看这篇文章，我已经折腾两天了

# 起因

本来想搞个第二博客写点非技术性的东西，经过一系列纠结，选定了 WordPress + Docker-Compose 的部署方案，本来教程都快写完了，但是 WordPress 死活拉不起来，昨天整整调试了一天，最后放弃。

后来选定了 Hugo + GitHub Web hook + Caddy 这种备选方案，因为我现在的博客就是这么搭建的。由于我不想开源，所以选了 GitHub 的私有仓库，一切都很美好。但是 Caddy 的 Git 插件有 bug，并不能处理 私有仓库这种 `git@github.com:username/example.git` 的地址，相关讨论 issue 在这里： https://github.com/abiosoft/caddy-git/issues/106 。简单来说就是标准库更新之后这个插件处理 ssh 式仓库地址的功能就炸了，虽然有人提 fix，但是开发者没有 merge。所以只能用最原始的办法了：Nginx+部署脚本+Web hook handler。

# 教程

## Nginx 安装

```bash
# 先安装 epel 源，才能安装 nginx
yum install epel-release
# 装 nginx
yum install nginx
# 启动nginx+自启
systemctl start nginx
systemctl enable nginx
```

安装后需要简单的配置，主要就是网站的 root 指定到 `/var/www` 目录中，当然你也可以指定到其他地方。

## 部署脚本

部署脚本是最重要的，去 Github 拉数据，送到 hugo 编译，把编译后的内容放到 nginx 的 web root 目录中。

我的部署脚本是这样的：

```bash
#!/usr/bin/env bash
cd story_site
git pull --recurse-submodules
hugo --cleanDestinationDir --destination=/var/www
```

写完部署脚本可以试一下，执行完毕之后应该就可以在浏览器中看到对应的博客页面了。

## 自动更新

现在的更新是每次搞完了手动调用 `deploy.sh` , 自动更新非常有必要。（因为 Caddy 版可以做到这一切）

```bash
wget https://github.com/adnanh/webhook/releases/download/2.6.10/webhook-linux-amd64.tar.gz
tar -xvf webhook*.tar.gz
mv webhook-linux-amd64/webhook /usr/local/bin
rm -rf webhook-linux-amd64*
```

新建一个hook描述文件 `webhook.json`

```json
[
  {
    "id": "redeploy-webhook",
    "execute-command": "/root/deploy.sh",
    "command-working-directory": "/root"
  }
]
```

然后运行  `webhook -hooks webhook.json -port=10086` 端口写什么都可以

然后去 GitHub 上设置 Webhook 就可以了。

Webhook 地址就是这样的 `http://ipaddress:10086/hooks/redeploy-webhook`

# 结语

折腾了两天，我本身没想到搞一个第二博客会这么麻烦，中间 debug 耗费太多时间。现在看来 PHP 的 WordPress 已经不适用于轻量级网站了，Caddy 也有 bug，等待修复把。（服务器上的 Caddy 还留着，等官方修复了）

