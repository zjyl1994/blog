---
title: "Authy 导出密钥"
date: 2019-12-07T21:40:00+08:00
lastmod: 2019-12-07T21:40:00+08:00
draft: false
---

> Authy 也可以导出密钥，而且现在很简单了

Authy 是一款挺好用的 2FA 生成器，问题在于它没有 KaiOS 版本，所以我昨天自己写了一个生成器。详情可以见上一篇文章。

Authy 官方没有给密钥导出方案（说是为了安全考虑不提供这个），网上的 js 方案都是从chrome版的authy中读取，还不能用。我花了很久找到了这个现成的方案。

 https://github.com/alexzorin/authy 

可以用 `go get github.com/alexzorin/authy` 下载这个包，然后使用 `go run $GOPATH/src/github.com/alexzorin/authy/cmd/authy-export/authy-export.go` 运行。

**注意** 不要用 Git Bash 运行这个程序，否则到输入密码部分会直接崩溃。建议用 CMD 或者 PowerShell 运行。

运行起来之后，首先要求输入的是 Authy 的电话区号，不带+号的。

然后输入手机号，它会联系 Authy 服务器，此时你的 Authy 设备上会有个登录提示，按照提示同意即可。

然后会显示输入 backup password，输入你的密码后，就会显示出你所有的 token。

输出的都是 `otp://` 开头的 2FA URI，你就可以把这些 OTP 导入其他 2FA 验证器了。

