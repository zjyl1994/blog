---
title: "Alpine with CGO"
date: 2019-04-08T22:17:00+08:00
lastmod: 2019-04-08T22:17:00+08:00
draft: false
tags: ["Alpine", "Golang", "CGO"]
categories: ["Golang"]

---

> 今天折腾这个一天，记录下来免得未来踩坑

# Alpine with CGO

Golang 很棒，静态编译十分方便。但是，它也不是 100% 静态编译的，因为它需要依赖glibc （ 标准C运行库 ）。

而 Docker 最常用的 Apline 镜像，使用的是 musl 库，并不能愉快的运行 Go 程序

网上大部分教程都是教你，`CGO_ENABLED=0 go build -a -installsuffix cgo`，使用纯 Go 编译，不用 CGO 链接 glibc ，问题就可以解决了。

但是最麻烦的问题是，你需要引入 C/C++ 库的时候，你并不能禁用 CGO 。

幸好 Alpine 有包管理器，所以我们可以很愉快的安装所需要的库。（我之前尝试手动安装glic和libstdc++，很难弄）

<!--more-->

默认的 Golang 官方编译环境中，没有我们所需的 GCC 和 G++ 编译环境，所以要先安装 build-base 配置编译环境。

运行镜像我们也使用 Alpine ，由于 Alpine 极为精简，并没有常用的时区、证书等，会导致不可预料的错误。所以我们需要安装这些东西：


包名            | 用处 
----------------|-----------------------------------------------
ca-certificates | \[可选\] CA 证书，方便使用 TLS
tzdata          | \[可选\] 时区配置，方便 GORM 等需要处理时间的场景
libc6-compat    | \[必选\] C 标准库
libgcc          | \[必选\] GCC 相关库，CGO 编译的程序会依赖 
libstdc++       | \[必选\] C++ 标准库


完整版 Dockerfile 如下：

（使用这个 Dockerfile 时需要将 `github.com/zjyl1994/app` 替换为自己的包路径）

```Dockerfile
FROM golang:1.12.2-alpine3.9 AS builder
RUN apk --no-cache add build-base
COPY . /code
RUN mkdir -p /usr/local/go/src/github.com/zjyl1994 && \
    ln -s /code /usr/local/go/src/github.com/zjyl1994/app && \
    cd /usr/local/go/src/github.com/zjyl1994/app && \
    CGO_ENABLED=1 go build -a

FROM alpine:latest
RUN apk --no-cache add tzdata ca-certificates libc6-compat libgcc libstdc++
COPY --from=builder /usr/local/go/src/github.com/zjyl1994/app/app /app/app
CMD ["/app/app"]
```