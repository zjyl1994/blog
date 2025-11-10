---
title: "Clockworkpi APT 镜像源"
date: 2021-10-26T20:20:00+08:00
---

> Clockworkpi 的 APT 源放在 Github 上，由于众所周知的原因，国内访问稀烂。

2020年12月我预购了DevTerm，A04版本还没有发货，但是A06版本已经有群友拿到了，
面临的一大问题就是官方的APT源在Github上，由于众所周知的缘故，国内访问十分艰难。
就算你侥幸连上了，大概率也是不稳定的。所以我就搞了一个小小的镜像源。

使用这个镜像源十分简单。只需要你执行以下命令。

`sudo sed -i 's/raw.githubusercontent.com/su.zjyl1994.com/g' /etc/apt/sources.list.d/clockworkpi.list`

原理就是将 `/etc/apt/sources.list.d/clockworkpi.list` 中的 `raw.githubusercontent.com` 替换成 `su.zjyl1994.com`。

执行完毕后 `sudo apt update`，更新软件源，是不是加载速度变快很多了呢？

此镜像部署在我的服务器上，并且在 Cloudflare 上缓存一小时。基本可以理解为和官方同步的。

<!--more-->

以下内容是技术向的，如果你只是想来更换软件源的可以不往下看了。

核心逻辑如下：

```go
func aptReverseProxy(c *gin.Context) {
    githubRawContent, _ := url.Parse("https://raw.githubusercontent.com")
	proxy := httputil.NewSingleHostReverseProxy(githubRawContent)
	proxy.Director = func(req *http.Request) {
		req.Header = c.Request.Header
		req.Host = githubRawContent.Host
		req.URL.Scheme = githubRawContent.Scheme
		req.URL.Host = githubRawContent.Host
		req.URL.Path = c.Request.URL.Path
	}
	proxy.ModifyResponse = func(resp *http.Response) error {
		resp.Header.Set("Cache-Control", "public, max-age=3600")
		return nil
	}
	proxy.ServeHTTP(c.Writer, c.Request)
}

```

Go 中自带了一个单主机的反代组件，只需要把 Path 修改好，嫁接到 Gin 上就 OK 了。

Director 负责修改向后传递的http请求，只需要把Host、Scheme、Path都改成Github上的就行了。

ModifyResponse 负责修改传给客户端的响应，通过增加 `Cache-Control` 让 Cloudflare 的 CDN 网络缓存一小时请求内容。

你没有看错，就这么简单。

完整源码在[这里](https://github.com/zjyl1994/suapis/blob/master/service/clockworkpi/proxy.go)。可以直接点进去看。

