---
title: "重构——在线网站更新手记"
date: 2021-10-06T20:04:00+08:00
---

> 趁着十一假期，我更新了一下网站的设计。

重构，为了更好的设计。

我的网站，有三到四个子站组成。最早的设计是通过 VPS 直接提供服务，
在一段时间的迭代中，我选择了 Cloudflare 进行防护和加速，虽然中国大陆地区
使用 Cloudflare 是减速行为。

现在的网站结构：

`www.zjyl1994.com` 是我的主站，主要负责导航到其他子站点。

`blog.zjyl1994.com` 是我的博客站，就是这个站点了。

`su.zjyl1994.com` 是我的后端站点，我的所有活动功能都挂在这里。

# Pages

www 和 blog 都是属于静态网站，www 是纯手写的 html 代码，
blog 的代码是使用 Hugo 生成的，对于这种静态资源站点，我选择了
Cloudflare Pages 进行托管。静态资源依托全球网络可以抵抗任何形式的攻击，
而且可以稳定的被 CDN 进行缓存，进行就近分发。

# Tunnel

Cloudflare 于今年早些时候公布了免费的 Tunnel 计划，原先是收费计划的 Argo Tunnel，
针对回源场景可以免费使用了。使用 Cloudflared 可以直接绕开 caddy、nginx 等反代工具，
让 CF 成为你的 Portal。

<!--more-->

我的服务器是 Ubuntu Server 20.04 LTS 的，所以可以直接参考官方的 [安装教程](https://pkg.cloudflare.com/) 。

```bash
echo 'deb http://pkg.cloudflare.com/ focal main' | sudo tee /etc/apt/sources.list.d/cloudflare-main.list
curl -C - https://pkg.cloudflare.com/pubkey.gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install cloudflared
```

安装过后，就可以使用 `cloudflared` 命令进行后续操作了。

输入 `cloudflared tunnel login` ，它会给出一个 URL 地址，复制到浏览器里进行授权。

输入 `cloudflared tunnel create <NAME>` 创建名为 NAME 的隧道，会给定一个 UUID 作为你的隧道标识符。

编辑 `~/.cloudflared/config.yaml`，参考如下输入：

```yaml
url: http://localhost:<PORT>
tunnel: <UUID>
credentials-file: /root/.cloudflared/<UUID>.json
```

PORT 替换成后端服务的端口号，UUID 替换成隧道标识符给的 UUID。

此时隧道配置完毕，通过 `cloudflared tunnel route dns <Name> <Endpoint>` 
在 CF 上创建指向 Endpoint 的 DNS 记录。此时，在 DNS 页面可以看到一条 Endpoint 的 CNAME 记录。

输入 `cloudflared tunnel run <Name>` 启动隧道，此时你就可以访问你的 Endpoint 网站，流量会打到你的内网服务端里。确定没问题后 Ctrl+C 停止运行，我们会用服务的形式启动隧道。

```bash
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

此时就可以稳定的访问你的内网服务了。

# Su APIs

原本的 Su API 包含了 “订阅合并”、“赛博鱼师傅” 功能，
全新重构的 Su API 在以上功能外扩展了 “文件共享库” 功能。

## 结构重构

结构重构，旧的 Su 属于传统的 Gin 程序，没有日志写盘，配置文件散落一地。

新的结构由 Gin + Zap 构成，全局状态包含 Debug 、Secret。
根据 Debug 位决定 Zap 的日志输出方式，具体可以 [参考代码](https://github.com/zjyl1994/suapis/blob/master/vars/setup.go) 。

全局 vars 包，这是我在公司项目中学到的。将公众变量抽取到一个独立的包里，方便程序全局调取，也方便统一初始化和管理。

## 订阅合并

订阅合并部分，增加了 Always Online 类似的功能解决数据源经常挂掉的问题。

实现方式就是访问订阅源进行回源的部分，成功返回之前增加了一个写盘操作，
当访问数据源失败时从磁盘加载上次获取成功的数据。
[参考代码](https://github.com/zjyl1994/suapis/blob/master/service/sssub/sub.go) 。

## 文件共享库

增加这个共享库的原因是因为我有时候需要在外面获取一些配置文件等信息，
需要在公开电脑上登陆各种网盘的话就会很麻烦。

DigitalOcean 的 Space 是一个 S3 兼容服务，每月5刀提供 250GB 的对象存储。共享 VPS 的 1TB 出站流量。

实现上就是很简单的 ListObject + Presigned 集合。 [参考代码](https://github.com/zjyl1994/suapis/tree/master/service/s3index) 。
实际实现中，增加了缓存机制减少每次调用 S3 API 的开销。
虽然 Space 的API调用不收费，但是时间层面还是比较大的开销。

## 赛博鱼师傅

赛博鱼师傅是一个 Telegram Bot 主要提供吃什么选择，旧的实现方式是读取配置文件到内存中，每次加载需要杀掉进程。
新的实现方式是独立读取单独的 `eatwhat.txt` 文档，更新文档可以实时反馈到吃什么选择器中。