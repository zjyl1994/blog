---
title: "服务器 Docker 化笔记"
date: 2019-05-05T23:50:00+08:00
lastmod: 2019-05-06T21:04:00+08:00
draft: false
tags: ["Docker"]
categories: ["Docker"]

---

> 五一期间折腾了一下服务器的 Docker 化，踩了好多的坑，写文记录下来。

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

# 配置 Docker 环境

## 安装 Docker CE

安装 Docker CE 并配置自启动 

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker
```

运行完上面这些命令的话，Docker 环境也应该装好了。如果你想试试 Docker 的话，可以输入 `docker run hello-world`，看一下 Docker 的 Hello World。

## 容器规划

一个好的服务器环境需要提前规划。

| 容器 | 用途 |
|-----|------|
| caddy | Web 服务器，反代所有 Web 服务 |
| portainer | 图形化的 Docker 管理面板 |
| mariadb | MySQL 服务器 (个人倾向 mariadb 分支) |
| phpmyadmin | 图形化的 MySQL 管理面板|
| registry | Docker 镜像的私有仓库 |
| blog | 我的博客服务 |

## 配置 Docker 的网络环境

要想 Docker 容器之间愉快的互联，需要设置一个 Docker 网络。(本文中用 xxcloud 作为网络名)

```bash
docker network create xxcloud
```

## 配置 Portainer

Portainer 是一个非常好用的 Docker 面板，要启动这个非常简单。

```bash
docker run -d  --restart=always -v /var/run/docker.sock:/var/run/docker.sock --name portainer --network xxcloud portainer/portainer
```

（由于我打算使用 Caddy 做反代，所以此处不暴露端口给外部，如果需要的话，自己添加 -p 参数映射外网端口）

非常神奇的是，Docker 容器中的面板可以操作 Docker 自身，只需要映射 docker.sock 文件到容器中就行了。

## 配置 Caddy

### 安装 Caddy

Caddy 是一个非常好用的 Web 服务器，支持自动 ACME 申请 Let's Encrypt 的 TLS 证书。很多东西都需要强制 HTTPS，如果使用传统的 Nginx，你需要去申请，然后拿到证书回来安装证书，每隔 90 天再手动去换证书。如果你不想手动换证书，市面能买到的证书也差不多需要1年一换。

使用 Caddy 就很简单了，给一个邮箱名字然后它自己会去处理这些麻烦事。

```bash
mkdir -p /home/data/caddy
docker run -d -p 80:80 -p 443:443 --restart=always --name caddy  --network xxcloud -e ACME_AGREE=true -v /home/data/caddy/caddyfile:/etc/Caddyfile -v /home/data/caddy/srv:/srv -v /home/data/caddy/certs:/root/.caddy abiosoft/caddy
```

Caddy 需要映射 80 和 443 到外网，因为它负责反代所有服务，作为整个的入口。

映射功能说明：

| 实体机上的路径 | 用途 |
|---------------|------|
| /home/data/caddy/caddyfile | 配置文件 Caddyfile |
| /home/data/caddy/srv | 存放网站文件 |
| /home/data/caddy/certs | 存放 Caddy 获取到的证书文件 |

80 和 443 都需要自己在 Firewalld 里打开端口，具体如何可以自己搜索。

### 配置 Caddy

修改 Caddyfile：(注意，你需要把 example.com 换成你自己的域名，具体Caddy写法可以参考Caddy教程)

```Caddyfile
portainer.example.com {
	tls email@example.com
	gzip
	proxy / http://portainer:9000
}
```

## 安装 MariaDB

日常开发中 MySQL 是非常重要的一个东西，我个人倾向于 MariaDB。MySQL 被 Oracle 收购以后，MySQL 的原作者在最后一个自己的版本上起了新的分支，开发了MariaDB。MariaDB 和 MySQL 完全兼容，所有的教程都能用，而且也不用怕 Oracle 搞事。

```bash
mkdir -p /home/data/mysql
docker run --name mariadb  --restart=always --network xxcloud -v /home/data/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=REPLACE_IT_WITH_YOU_ROOT_PASSWORD -d mariadb:latest
```

运行完上面的命令，MariaDB 就拉起来了。MariaDB 的数据会存在 /home/data/mysql 里。

**注意：**  MYSQL_ROOT_PASSWORD 这个密码一定要换成自己的，而且，这个仅限于第一次运行时设置。如果你的 /home/data/mysql 里有现成的 MySQL 数据，那么容器会使用现成数据里的密码。

## PHPMYADMIN

### 安装 PHPMYADMIN

```bash
mkdir -p /home/data/phpmyadmin/themes
docker run --name phpmyadmin  --restart=always --network xxcloud -e PMA_HOST=mariadb -e PMA_ABSOLUTE_URI=https://phpmyadmin.example.com -d -v /home/data/phpmyadmin/config.user.inc.php:/etc/phpmyadmin/config.user.inc.php -v /home/data/phpmyadmin/themes:/var/www/html/themes phpmyadmin/phpmyadmin
```

官方教程十分的坑，并没有说明白实际的运行位置。官方文档说主题需要映射 `/www/themes`，实际上这个主题文件在 `/var/www/html/themes`，我费了好大劲才进入容器查到位置。（它使用的是sh，也不在/bin里，只能来回试）

映射功能说明：

| 实体机上的路径 | 用途 |
|---------------|------|
| /home/data/phpmyadmin/config.user.inc.php | 配置文件 |
| /home/data/phpmyadmin/themes | 放主题 |

### 配置 PHPMYADMIN

#### 持久化高级设定

phpMyAdmin 的高级设定想要持久化，需要在 MySQL 里建立数据库。新建一个叫 phpmyadmin 的数据库，使用 phpMyAdmin 源码包中的 `sql/create_tables.sql` 初始化数据表。

编辑 `/home/data/phpmyadmin/config.user.inc.php` :

```php
<?php
$cfg['blowfish_secret'] = '/Cwzlc:SHQ9}sxGN/0;iuOX1:H3QSbpy';
$i = 0;
$i++;
$cfg['Servers'][$i]['auth_type'] = 'cookie';
$cfg['Servers'][$i]['host'] = 'mariadb';
$cfg['Servers'][$i]['compress'] = false;
$cfg['Servers'][$i]['AllowNoPassword'] = false;
$cfg['Servers'][$i]['controlhost'] = '';
$cfg['Servers'][$i]['controlport'] = '';
$cfg['Servers'][$i]['controluser'] = 'phpmyadmin';
$cfg['Servers'][$i]['controlpass'] = 'REPLACE_IT_WITH_YOU_ROOT_PASSWORD';
$cfg['Servers'][$i]['pmadb'] = 'phpmyadmin';
$cfg['Servers'][$i]['bookmarktable'] = 'pma__bookmark';
$cfg['Servers'][$i]['relation'] = 'pma__relation';
$cfg['Servers'][$i]['table_info'] = 'pma__table_info';
$cfg['Servers'][$i]['table_coords'] = 'pma__table_coords';
$cfg['Servers'][$i]['pdf_pages'] = 'pma__pdf_pages';
$cfg['Servers'][$i]['column_info'] = 'pma__column_info';
$cfg['Servers'][$i]['history'] = 'pma__history';
$cfg['Servers'][$i]['table_uiprefs'] = 'pma__table_uiprefs';
$cfg['Servers'][$i]['tracking'] = 'pma__tracking';
$cfg['Servers'][$i]['userconfig'] = 'pma__userconfig';
$cfg['Servers'][$i]['recent'] = 'pma__recent';
$cfg['Servers'][$i]['favorite'] = 'pma__favorite';
$cfg['Servers'][$i]['users'] = 'pma__users';
$cfg['Servers'][$i]['usergroups'] = 'pma__usergroups';
$cfg['Servers'][$i]['navigationhiding'] = 'pma__navigationhiding';
$cfg['Servers'][$i]['savedsearches'] = 'pma__savedsearches';
$cfg['Servers'][$i]['central_columns'] = 'pma__central_columns';
$cfg['Servers'][$i]['designer_settings'] = 'pma__designer_settings';
$cfg['Servers'][$i]['export_templates'] = 'pma__export_templates';
$cfg['DefaultLang'] = 'zh';
```

其中 `controluser` 是你给 phpMyAdmin 存储数据时开的数据库账户，
`controlpass` 是账户的密码，
`pmadb` 是数据库名。

要注意的是 `blowfish_secret` 需要你生成一个自己的密语，如果你不知道如何生成，可以点[这里](https://phpsolved.com/phpmyadmin-blowfish-secret-generator/)，每次刷新你都会得到一个新的随机密语。

#### 配置主题
PHPMYADMIN 的主题可以在 [https://www.phpmyadmin.net/themes/](https://www.phpmyadmin.net/themes/) 这里下载，下载之后可以在 PHPMYADMIN 首页主题处切换。

**注意：** 下载后的主题需要解压后变成文件夹放在themes文件夹里。由于有 php 文件，所以最好使用官网上的主题，安全性比较高。

### 通过 Caddy 映射外网访问

编辑前文提到的caddyfile，增加如下内容

```Caddyfile
phpmyadmin.example.com {
	tls email@example.com
	gzip
	proxy / http://phpmyadmin:80
}
```

## 配置私有 Docker 仓库

### 安装 Docker Registry

安装私有 Docker Registry 很简单，它本身也是一个镜像。

我希望的 Docker Registry 是 HTTPS + 密码保护 的安全私有仓库，所以说首先需要生成密码文件。

```bash
mkdir -p /home/data/auth
docker run --entrypoint htpasswd registry:2 -Bbn testuser testpassword > /home/data/auth/registry.htpasswd
```

你需要替换上述命令中的 `testuser` 和 `testpassword` 为你自己的用户名和密码。

然后用命令 `docker container stop registry` 停止刚才的 registry 容器

现在，可以拉起真正使用的 registry 了。

```bash
mkdir -p /home/data/registry
docker run -d --restart=always --name registry --network xxcloud -v /home/data/registry:/var/lib/registry -v /home/data/auth/registry.htpasswd:/auth/htpasswd -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd registry:2
```

实体机上的 `/home/data/registry`，就是存储仓库数据的地方，建议放在空间大的目录下。

### 配置 Caddy

编辑 Caddyfile

```Caddyfile
registry.example.com {
	tls email@example.com
	proxy / http://registry:5000 {
		transparent
	}
}
```

**特别注意：** 反代 registry 的时候一定要增加 `transparent` 这个属性，要不然 push 镜像的时候会无限 retry。（我在这上面坑了好久）


## 配置博客

配置博客可以单独写一篇文章，就不在这写了

# 后记
很久没有写过这么长的文章了，我是一个金鱼记忆的人 （毕竟是咸鱼），如果不记下来肯定会忘记。

写这篇博客一个是为了记录我遇到的问题踩到的坑，如果能帮到后来人更好。

服务器 Docker 化之后整洁很多，数据都在 `/home/data` 里，一目了然，尝鲜新的东西只需要docker拉下来，也不用配置更多更麻烦的东西。而且自制的一些东西只要本地 Docker 能跑起来，push 到线上环境100%也能跑起来，这就是 Docker 的优势。总的来说利大于弊。

而且，我在配置完所有内容后，服务器实际占用空间也就500MB - 700MB。当然 Docker 还是需要大内存的，1G 以下的服务器就不要这么折腾了。