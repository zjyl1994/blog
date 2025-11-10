---
title: "Docker 配置代理"
date: 2025-10-06T11:50:00+08:00
---

> 国内的网络环境大家都懂，没有科学工具的可以不用看了。

我的开发环境Win11+VirtualBox中跑的Fedora Server，后台无头启动，所以常规的代理配置方法不奏效。

VirtualBox虚拟机配置的桥接网络和HostOnly，使用HostOnly网卡与宿主机通信，
此时就能固定宿主机IP为192.168.56.1了。
假设此时主机上开了局域网代理在7890端口，所以我们得到了一个稳定的http/https代理：`http://192.168.56.1:7890`

首先修改Docker Daemon的配置文件，添加`/etc/systemd/system/docker.service.d/http-proxy.conf`，内容如下：

```
[Service]
Environment="HTTP_PROXY=http://192.168.56.1:7890"
Environment="HTTPS_PROXY=http://192.168.56.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,192.168.56.0/24"
```

然后重启Docker Daemon：

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

上面的代理只能加速`docker pull/push`的速度
在使用`docker build`的时候，构建阶段也需要配置代理，否则会异常缓慢。

构建时需要添加网络参数`--network=host`，并且指定`--build-arg`参数挂代理。
最后能得到完整的构建命令：
`docker build --network=host --build-arg http_proxy=http://192.168.56.1:7890 --build-arg https_proxy=http://192.168.56.1:7890 -t your_image_name:latest .`

日常使用可以在`~/.bashrc`添加`alias`，方便使用：
```
alias docker-build='docker build --network=host --build-arg http_proxy=http://192.168.56.1:7890 --build-arg https_proxy=http://192.168.56.1:7890'
```

这样，直接使用`docker-build -t your_image_name:latest .`即可。
