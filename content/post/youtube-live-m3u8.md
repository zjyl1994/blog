---
title: "Youtube直播转IPTV源"
date: 2020-05-03T23:05:19+08:00
lastmod: 2020-05-03T23:05:19+08:00
draft: false
---

> Youtube 是个大宝库！

# 前言

Youtube 直播上有很多好东西，比如各大电视台喜欢在油管上直播自己的新闻频道。

比如海量台湾电视新闻台都在油管上搞了直播，我整理了一下这些。

```txt
#三立LIVE新聞HD直播
https://www.youtube.com/watch?v=4ZVUmEUFwaY
#TVBS新聞 55 頻道
https://www.youtube.com/watch?v=Hu1FkdAOws0
#東森財經新聞 57
https://www.youtube.com/watch?v=dphWo0r27Z4
#東森新聞 51 頻道
https://www.youtube.com/watch?v=RaIJ767Bj_M
#CTI中天新聞HD直播
https://www.youtube.com/watch?v=wUPPkSANpyo
#中視新聞台 LIVE
https://www.youtube.com/watch?v=3OPNkiqD48g
#民視新聞直播
https://www.youtube.com/watch?v=XxJKnDLYZz4
#華視新聞HD
https://www.youtube.com/watch?v=TL8mmew3jb8
```

<!--more-->

# 解析

我曾经想过用 Golang 自己写一套 Youtube 解析算法，但是考虑到我可能没有精力随时维护解析算法，所以还是要用现成的第三方库来弄。

Github 有个非常出名的第三方 Youtube 工具，[youtube-dl](https://github.com/ytdl-org/youtube-dl)。能在开源世界上拿到 65.6k 的 star 非常不容易，
也侧面证明了这个工具的靠谱程度。通过查看 `youtube-dl` 的帮助，可以知道 `youtube-dl -f best -g {url}` 能解析到 Youtube 的 M3U8。

在 Golang 中调用程序并获取输出内容，可以这么做：

```go
_, err := exec.LookPath("youtube-dl")
if err != nil {
    return "", err
} else {
    ctx, cancelFunc := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancelFunc()
    cmd := exec.CommandContext(ctx, "youtube-dl", "-f","best","-g","${URL}")
    out, err := cmd.CombinedOutput()
    return strings.TrimSpace(string(out)), err
}
```

首先需要检查要调用的程序有没有，免得出问题。`exec.LookPath` 会检查能否找到程序，在 `err == nil` 时，证明可以调用到程序。
使用 `exec.Command` 指定调用的命令行，`CombineOutput` 会运行并输出程序运行内容。需要考虑的是，运行程序可能会卡住，尤其是
国内的神奇网络，如果没有扶墙，可能会无限卡下去。所以，必须增加超时。通过指定超时 `context.WithTimeout(context.Background(), 10*time.Second)`，
设定 10 秒的延迟。所以这个时候需要把 `exec.Command` 换成 `exec.CommandContext`，超时之后会自动取消。

# 提供服务

Kodi 之类的 IPTV 客户端，都只能使用 M3U8 地址。所以需要一个 Go 程序提供 Youtube 直播地址转换到 M3U8 功能。用 Gin 可以非常容易的写一个 Web 服务。

```go
func liveHandler(c *gin.Context) {
	liveURL := c.Query("url")
	if liveURL == "" {
		c.AbortWithStatus(http.StatusNotFound)
		return
	}
	liveM3U8, err := getYoutubeLiveM3U8(liveURL)
	if err != nil {
		log.Println(err)
		c.AbortWithStatus(http.StatusInternalServerError)
		return
	}
	if cfg.ProxyStream {
		liveProxyM3U8 := cfg.BaseURL + "/p/live.m3u8?url=" + url.QueryEscape(liveM3U8)
		c.Redirect(http.StatusTemporaryRedirect, liveProxyM3U8)
	} else {
		c.Redirect(http.StatusTemporaryRedirect, liveM3U8)
	}
}
```

## 代理直播流

正常的油管 M3U8 在电视之类没扶墙的环境是不能播放的，所以代理是很有必要的。如何代理可以参看 RTHK 那篇博文。

这种代理就是简单的读取，转发。只是 M3U8 里面的连接也跟着替换一下，注意必须保持扩展名 `.m3u8` `.ts` ，
要不然 VLC 之类的会无法播放。

## 优化加缓存

实际使用的时候，会发现一个很大的问题。比如 Kodi 换台的时候，首先会访问设定的地址，然后调用一次 `youtube-dl`，
解析个6-7秒返回结果，再加上 Kodi 需要去真实的直播源加载数据，每次换台需要卡顿 8-10s 左右。

考虑到频道列表是固定的，所以可以提前使用 `youtube-dl` 加载一下。要缓存数据，最好的方法是使用 redis 之类的工具。
不过为了这个简单的缓存引入 redis 实在不值当，所以直接使用内置类型 map 会很方便，高并发环境下需要使用 `sync.Map` 防止冲突 panic。

具体实现可以参考 [https://github.com/zjyl1994/livetv/blob/master/m3u8cache.go](https://github.com/zjyl1994/livetv/blob/master/m3u8cache.go) 文件。

### 缓存规则

在进行 Youtube 地址转换获取 M3U8 的时候，先行访问快取，如果命中快取就直接用快取中获取。获取不到再去用真实的 `youtube-dl` 获取连接。

### 冷启动

当程序启动的时候，快取中没有任何数据，所以程序启动时需要逐个获取连接放入快取。

```go
func loadChannelCache() {
	channels, err := channelParser(cfg.ChannelFile)
	if err != nil {
		log.Println(err)
		return
	}
	for _, v := range channels {
		log.Println("caching", v.URL)
		liveURL, err := realGetYoutubeLiveM3U8(v.URL)
		if err != nil {
			log.Println(err)
			return
		}
		urlCache.Store(v.URL, liveURL)
		log.Println(v.URL, "cached")
	}
}
```

### 热更新

Youtube 生成的 M3U8 中有过期时间，一般是6小时。如果想要保持随时可看的状态，就得经常更新。
编写一个更新函数，从频道定义文件里获取 Youtube 链接，然后更新一下所有频道的 M3U8。
然后使用 `sync.Map.Range` 遍历所有的链接，清理掉过期的链接，只要2-3小时抓取一次即可。

```go
func updateURLCache() {
	channels, err := channelParser(cfg.ChannelFile)
	if err != nil {
		log.Println(err)
		return
	}
	for _, v := range channels {
		log.Println("caching", v.URL)
		liveURL, err := realGetYoutubeLiveM3U8(v.URL)
		if err != nil {
			log.Println(err)
		} else {
			urlCache.Store(v.URL, liveURL)
			log.Println(v.URL, "cached")
		}
	}
	urlCache.Range(func(k, v interface{}) bool {
		value := v.(string)
		regex := regexp.MustCompile(`/expire/(\d+)/`)
		matched := regex.FindStringSubmatch(value)
		if len(matched) < 2 {
			urlCache.Delete(k)
		}
		expireTime := time.Unix(string2Int64(matched[1]), 0)
		if time.Now().After(expireTime) {
			urlCache.Delete(k)
		}
		return true
	})
}
```

# 安装和使用

本次的程序使用多配置文件模式，具体可以参考 [https://github.com/zjyl1994/livetv/blob/master/README-zh.md](https://github.com/zjyl1994/livetv/blob/master/README-zh.md)。

这次折腾完了，就可以用 Kodi 看香港台，台湾台，等等各种直播频道，配置好之后就享受观影吧。