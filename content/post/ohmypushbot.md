---
title: "OhMyPushBot ———— Telegram 推送 Bot"
date: 2019-07-06T23:41:00+08:00
lastmod: 2019-07-06T23:41:00+08:00
draft: false
---

> 前几天做的小玩具

# OhMyPushBot

Telegram 是个有趣的聊天工具，最好玩的就是它十分开放 Bot 系统。相比于微信公众号这种层层审核还一堆限制，或者 QQ、微信 机器人这种强行扒官方协议的灰色接口，
Telegram 提供的 API 非常简单。你可以用 Bot 管理 Channel，制作好玩的 Bot，这一切只需要和 BotFather 聊聊天，就可以设置并创建一个Bot，完全不需要等待审核。

制作这个 Bot，最大的用处就是方便一些自动化脚本通知事件结束。一般的场景下，自动化脚本要发送通知，多数使用发送邮件的方案，但是发送邮件需要使用专用的工具。
而且一般的邮件需要自己检查邮件情况，早些年有过黑莓的PushMail服务，几百元一个月的服务费也不是一般人玩得起的。现在好了，Telegram 作为一个免费 IM 工具，
完善的客户端支持（Linux、Windows上都有，各大手机平台也都有），比较实时的推送速度，简直是事件通知最佳的选择。

<!--more-->

# 如何使用

我已经部署了一个实例，挂在我的服务器上做转发。访问 [OhMyPushBot](https://t.me/ohmypushbot)，先和 Bot 聊一下天。

1. 和 Bot 聊天，点击 `start` 按钮
1. 点击命令或者手动输入 `/url` 命令
1. Bot 会发送给你一个 URL，像是 https://ohmypushbot.zjyl1994.com/send?chatid=1234567890&sign=xxxxxxxxxxxxxxxxxxx
1. 给刚才的 URL 发送一个 HTTP POST 请求，请求Body中的消息会被投递到您和 Bot 的聊天窗口中
1. 消息体正文支持 Markdown 格式，请求成功后接口返回 `ok` ，否则返回错误消息

# Markdown 支持

消息体支持的 [Markdown 特性](https://core.telegram.org/bots/api#markdown-style) ，原文是英文的，简单翻译如下：

<pre>
*粗体*
_斜体_
[行内超链接](http://www.example.com/)
[行内提及用户](tg://user?id=123456789)
`行内代码块`
```代码块
预先格式化过的代码块
```
</pre>

# 如何实现

原生的 Telegram Bot API 非常好用，只需要你拿到合法的 Bot Token，调用 HTTP 格式的接口，就可以发送消息，也可以用 Webhook 接收用户发来的消息。

这个 Bot 主要使用了两个 API，`sendMessage` 和 `setWebhook`,使用方法可以参见 [Telegram Bot API 文档](https://core.telegram.org/bots/api)。

首先我先使用 `setWebhook` 挂了一个 Webhook 到 Telegram 服务器，当 Bot 收到消息的时候，Telegram 会主动调用我们的接口，解析 JSON，
可以得到当前聊天的 ChatID，调用 `sendMessage` 通过 ChatID 可以发送消息到特定的对话中。
具体实现可以参见开源代码的 [service.go](https://github.com/zjyl1994/telegram-push-bot/blob/master/service.go)。

为了防止无聊的人随机生成 ChatID 调用接口发送垃圾消息，我增加了一个简单的参数验证，接口的 sign 部分就是 ChatID 根据 BotToken 进行 HMAC-MD5 之后的结果。
如果不知道的人想伪造 sign，从密码学角度来说几乎不可能。而且 HMAC 的 Key 是 BotToken，每个Bot都不一样，所以十分的安全。

# 开源代码

为了方便各位审计我的代码，我把代码托管到了 GitHub 上，地址是：[https://github.com/zjyl1994/telegram-push-bot](https://github.com/zjyl1994/telegram-push-bot)
欢迎各位大佬和神仙拍砖。