---
title: "打造自己的 Seedbox"
date: 2025-03-26T23:45:00+08:00
lastmod: 2025-03-26T23:45:00+08:00
draft: false
---

# 背景

曾经看番的好去处B站越来越烂了，导致完全没兴趣续费大会员。所以我尝试回到过去，重新捡起来 BT 下载。

很庆幸，2025年动漫花园和nyaa都还在，而且一直有更新。为此我还组装了一部小的全闪下载机进行 BT 下载。

但是很快我就发现一个问题，速度非常不理想，首先我的宽带是一个N层NAT的私人宽带，不仅无法连接很多Tracker，而且无法贡献上传。
下完就跑可不是什么好行为，这算是 BT 吸血，所以我决定自建一个BT Seedbox，在公网上下载顺便保种，然后用 HTTP 拖回本地进行观看。

# VPS 选择

BT不同于PT，有一个最大的问题就是蜜罐钓鱼，当你连上某些蜜罐以后，版权方就会给你的主机商发DMCA函。
一般这个时候主机商就会封掉你的机器保平安，根据主机商的ToS，你可能被清退或者Ban机。

一般来说此时就需要DMCA Free的主机商了，其中 BuyVM 的卢森堡机房是很多人都在推荐的，卢森堡是欧洲非常小的大公国，独立主权。根据 BuyVM 的 ToS，处理DMCA需要卢森堡当地线下出警。（基本上约等于是不会封的了）

不过这个机房在6个月左右会进行一次迁移，根据主机商在 LET 的说法，新的搬迁目的地会是瑞士。
据他所称，搬去瑞士后服务不变，拭目以待吧。

# 搭建 Seedbox
## 基本环境
首先需要一台 BuyVM 的卢森堡主机，通常需要抢购。3.5刀一个月，1C1G 20G的硬盘，G口无限流量。北京时间每天晚上9-10点补货。

同时，你也需要去抢购一下对应区域的存储块，1.25刀/256GB/月。也是每天晚上9-10点补货。（存储块无法跨区挂载，也无法在没有机器的情况下提前购买）1T以下的小型存储块非常抢手，一般半小时不到就没了，所以关注几个 BuyVM 补货频道，收到通知第一时刻起来抢很容易成功。

我现在的环境是3.5刀的主机+2.5刀的存储块，每月花费6刀，按当前汇率年付约500人民币。

购买好以后，需要在 Stallion 后台绑定存储块。然后进行分区，我所使用的挂载点是 `/data`。

考虑到内存使用量可能会较高，所以添加了2GB的Swap文件，不过从后面来看，基本没有使用到。

这一部分网上教程很多，跟着教程走就好了。

## 安装 BT 客户端
我使用的是 qBittorrent-Enhanced-Edition 作为 BT 客户端，他能防止迅雷吸血等多种 BT 问题。

首先需要安装 qBittorrent-Enhanced-Edition 客户端，他是一个静态编译的二进制文件，不需要任何依赖。

```bash
wget https://github.com/c0re100/qBittorrent-Enhanced-Edition/releases/download/release-5.0.4.10/qbittorrent-enhanced-nox_x86_64-linux-musl_static.zip
unzip qbittorrent-enhanced-nox_x86_64-linux-musl_static.zip
sudo mv ./qbittorrent-nox /usr/local/bin/qbittorrent-nox
sudo chmod +x /usr/local/bin/qbittorrent-nox
```

创建对应的用户和数据文件夹

```bash
sudo useradd -m qbit
sudo mkdir -p /data/qbittorrent
sudo chown qbit:qbit /data/qbittorrent
```

然后需要配置启动它的 systemd 文件,创建`/etc/systemd/system/qbittorrent.service`文件并粘贴以下内容。

```plaintext
[Unit]
Description=qBittorrent-nox service
Documentation=man:qbittorrent-nox(1)
Wants=network-online.target
After=local-fs.target network-online.target nss-lookup.target

[Service]
Type=simple
PrivateTmp=false
User=qbit
ExecStart=/usr/local/bin/qbittorrent-nox
TimeoutStopSec=1800

[Install]
WantedBy=multi-user.target
```

保存后，使用以下命令启动 qBittorrent 服务
```bash
sudo systemctl daemon-reload
sudo systemctl enable qbittorrent.service
sudo systemctl start qbittorrent.service
```

此时运行`systemctl status qbittorrent.service`可以查看服务状态。

可以看到 qbee 的 web 端口已经在 localhost:8080 启动了，
会有一行 Admin Password 记录了本次部署的初始密码。
但是此时我们还无法访问，因为我们还没有配置反向代理。

## 配置反向代理
Caddy 作为新时代的 HTTP 服务器，已经进入 Ubuntu 的软件源了，可以从 `sudo apt install caddy` 直接进行安装。
当然你会玩 Nginx 的话，也可以自己配置。

首先需要创建一个 `/etc/caddy/Caddyfile` 文件，内容如下：
```plaintext
qbittorrent.example.com {
    reverse_proxy http://localhost:8080
}

files.example.com {
    file_server {
        root /data/qbittorrent
        browse
    }
}
```
然后保存后，使用以下命令启动 Caddy 服务
```bash
sudo systemctl enable caddy.service
sudo systemctl start caddy.service
```

## 配置 BT 客户端
访问 `qbittorrent.example.com` 后，会提示你输入用户名和密码，默认的用户名是 `admin`，密码就是你在 `systemctl status qbittorrent.service` 中看到的 Admin Password。
然后点击界面上的“蓝色齿轮”进入 qBittorrent 的设置，`Behavior` -> `Language` -> `User Interface Language`，找到“简体中文”然后点`Save`按钮就切换到中文了。

再次点击“蓝色齿轮”进入 qBittorrent 的设置，配置以下内容：
`下载` -> `默认保存路径` 填写 `/data/qbittorrent`。
`WebUI` -> `验证` -> `用户名`和`密码` 改成你想要的值。
然后点击最下面的`保存`按钮。

此时，就都配置完了，你可以使用 qBittorrent 来愉快的下载 BT 了。

## 访问下载的文件
访问 `files.example.com` 就可以查看到下载好的文件。

卢森堡到大陆的网络质量很差，无法支撑大部分下载内容的在线流畅播放，所以最好还是用其他手段拖回本地 NAS。

这种就可以使用Aria2或者其他离线挂机软件以接近国际端口满速往回拖了。

# 后记

在我写这篇文章的时候，我的 Seedbox 已经运转超过24小时了。

理论上来说，搞这个是比叔叔那边充钱要贵的，但是 BT 嘛，玩的开心最重要。

也许正在 BT 下载的你，Peer 列表里和你交换数据的某一台卢森堡主机就是我的 :P