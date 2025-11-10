---
title: "Cap 的 Go 后端"
date: 2025-09-11T07:38:00+08:00
---

# Cap

Cap是个基于PoW工作量证明的验证码体系，用来给每一次请求添加一定的CPU成本，防止恶意请求。真正的客户只要计算一次，消耗一次成本。恶意尝试需要计算成千上万次，大大增加成本，使其放弃尝试。

这个验证码的主要适用场景是保护登录表单，防止暴力破解在你的接口上跑字典。让攻击者的每次请求都有更高的CPU成本。如果你需要智能防爬，可以考虑使用Anubis这款WAF。

<!--more-->

原版的 [Cap](https://capjs.js.org/) 基于Javascript实现，提供了独立的验证服务来方便和其他程序进行集成。但是对于我这个Redis依赖都不愿意引入的人来说，是非常难受的。所以我搞了个Go后端，方便直接集成到现有的系统里。

# 那么如何使用呢？

1. 首先你需要安装我写的包 `go get github.com/zjyl1994/cap-go`。
1. 实现IStorage，提供token数据的保存接口。
1. 注册POST接口，`/cap/challenge`，对接`cap.CreateChallenge`的输出。
1. 注册POST接口，`/cap/redeem`，对接`cap.RedeemChallenge`的输入输出。
1. 对接前端验证组件，这里使用官方的`@cap.js/widget`组件对接，记得`data-cap-api-endpoint`为你注册的验证端口地址。
1. 用户操作前端组件验证通过后，会给一个token，你可以使用`cap.ValidateToken`来验证token是否有效。

# 示例

这里有我写的简单示例，可以直接运行。

https://github.com/zjyl1994/cap-go/tree/master/examples

当然，这里为了简化实现，没有正确实现过期动作，推荐真正使用的时候引入freecache，提供空间固定的kv存储。

下面给一个我在生产环境使用的IStorage：

```go
import (
	"time"

	"github.com/coocood/freecache"
)

type freeCacheStorage struct {
	cache *freecache.Cache
}

// NewFreeCacheStorage 创建一个新的存储实例
// capacity 单位是字节，例如 100 * 1024 * 1024 表示 100MB
func NewFreeCacheStorage(capacity int) *freeCacheStorage {
	return &freeCacheStorage{
		cache: freecache.NewCache(capacity),
	}
}

func (f *freeCacheStorage) Get(key string) string {
	value, err := f.cache.Get([]byte(key))
	if err != nil {
		return ""
	}
	return string(value)
}

func (f *freeCacheStorage) Set(key, data string, expire time.Time) {
	var ttl int
	if expire.IsZero() {
		ttl = 0 // 不过期
	} else {
		ttl = int(time.Until(expire).Seconds())
		if ttl < 1 {
			ttl = 1 // freecache 最小为 1 秒
		}
	}
	f.cache.Set([]byte(key), []byte(data), ttl)
}

func (f *freeCacheStorage) Del(key string) {
	f.cache.Del([]byte(key))
}
```
如果你有持久化需求，也可以实现该接口直接把数据写入数据库。
不过对于登陆表单这个场景来说，没有必要。所以单纯的内存存储就够了。
