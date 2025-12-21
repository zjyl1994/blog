---
title: "2026 数字生活提升计划"
date: 2025-12-21T10:27:00+08:00
---

为了迎接2026年，我计划提升我的数字生活体验。
因此决定把密码管理器、云笔记迁移到境内的服务器上提升使用体验。

我有一个很重要的刚需就是Web端，公司电脑上有深信服等审计工具，所以很多需要安装客户端的产品我都不能使用。

本篇文章我不会详细介绍每个服务的用法，相关服务知名度很高，网上视频非常多，我这里只记录我自己个人觉得的最佳部署方式。

# 服务部署
服务器部署容器时，我选择使用podman+quadlet的方式，这样可以与systemd进行集成，不需要管理散落的docker compose文件。

fedora已经自带podman了，如果是ubuntu可以使用apt安装。

```bash
sudo apt install podman
```

## Vaultwarden 密码管理器
Vaultwarden是一个Bitwarden的服务器端实现，我将把我的密码管理器迁移到这个服务器上。

新建一个文件`~/.config/containers/systemd/vaultwarden.container`，内容如下：

```ini
[Unit]
Description=Vaultwarden container
After=network-online.target

[Container]
AutoUpdate=registry
Image=ghcr.io/dani-garcia/vaultwarden:latest
Exec=/start.sh
Environment=DOMAIN=https://vault.example.com
Environment=ROCKET_PORT=8080
Environment=ROCKET_ADDRESS=0.0.0.0
Environment=SIGNUPS_ALLOWED=false
Environment=INVITATIONS_ALLOWED=false
Volume=%h/vaultwarden:/data/
PublishPort=127.0.0.1:8181:8080

[Service]
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

然后执行
```bash
systemctl --user daemon-reload
systemctl --user enable vaultwarden
systemctl --user start vaultwarden
```

配置你的反代指向`http://127.0.0.1:8181`即可访问你部署的实例。

注意，首次需要设置`SIGNUPS_ALLOWED=true`注册第一个账号，注册后再关闭，避免其他人进来注册滥用。如果你真有共享需求也应该通过邀请的形式发送链接给好友，而不是开放自助注册。

## Trilium 笔记
Trilium是一个基于Web的笔记应用，我将把我的笔记迁移到这个服务器上。

在此之前我都是使用Joplin+WebDAV，但是Joplin刚需客户端同步，没法直接Web编辑，并且E2EE只对服务器端设防客户端却是明文存储，所以经过精挑细选选择了Trilium。

```ini
[Unit]
Description=Trilium Notes Container (Quadlet)
After=network.target

[Container]
Image=docker.io/triliumnext/trilium:v0.100.0
Volume=%h/trilium:/home/node/trilium-data
PublishPort=127.0.0.1:9980:8080

[Service]
Restart=on-failure
```

然后执行
```bash
systemctl --user daemon-reload
systemctl --user enable trilium
systemctl --user start trilium
```

美中不足的就是，截至写文的时候还没有rootless镜像，导致镜像目录中的文件还都是`UID:10099`的权限。不过仅仅影响备份场景，日常使用不受影响。

配置你的反代指向`http://127.0.0.1:9980`即可访问你部署的实例。首次访问需要设置主密码，务必记住。

## Backrest 备份
有了以上这些重要的自部署服务，最重要的就是备份。
考虑到我现有的服务器资源都在深圳，互相备份不是好的灾备方案，最好的选择就是异地备份。
所以直接去Backblaze开了一个美西的B2存储桶专供备份使用，B2在10GB以内是免费的，超出以后每1TB $5/月，比较划算。

**注意，开的存储桶请选择private权限，除非你想让全世界都可以下载。**

部署过程：

1. 下载[restic](https://github.com/restic/restic/releases/tag/latest)，一般vps选择linux_amd64版本。
2. 上传到vps，bunzip2解压，复制到`/usr/bin`目录下。
3. 下载[backrest](https://github.com/garethgeorge/backrest/releases/tag/latest)，一般vps选择linux_x86_64版本。
4. 上传到vps，tar xvfz 解压，执行其中的`install.sh`。

之后你就可以在 `http://localhost:9898` 访问backrest的控制面板了，配置合适的反代即可通过域名访问。

**不要使用ubuntu中的restic，版本太老backrest不支持。要自己去下载最新二进制文件安装。**

界面配置概念和restic相同，创建Repo，创建要备份的计划，触发时间，设置保留副本数即可。

由于Trilium的权限问题，所以Backrest不适合放在容器中使用，我直接在主机上用Root权限的systemd运行，这样能直接读写需要备份的数据。

# 笔记App

Trilium Web端直接访问域名即可，输入主密码就能在任何设备上记录并查阅你的笔记。

安卓平台有一个第三方安卓App [TriliumDroid](https://github.com/FliegendeWurst/TriliumDroid)，在我的三星手机上运行一切正常，在小米手机上每次都需要重新同步才能使用，不过不耽误什么。

# 附录
- [Vaultwarden](https://github.com/dani-garcia/vaultwarden)
- [Trilium](https://github.com/TriliumNext/Trilium)
- [TriliumDroid](https://github.com/FliegendeWurst/TriliumDroid)
- [Restic](https://github.com/restic/restic)
- [Backrest](https://github.com/garethgeorge/backrest)