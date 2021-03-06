---
title: "区块链是什么，我来告诉你"
date: 2018-05-02T21:26:00+08:00
lastmod: 2018-05-03T00:07:00+08:00
draft: false
tags: ["区块链"]
categories: ["区块链"]
---

# 区块链是什么

区块链，当今最火热的技术热点。为什么这么火呢，三个字：“比特币”。

首先，我们要知道区块链是什么，根据[维基百科的描述](https://zh.wikipedia.org/wiki/%E5%8C%BA%E5%9D%97%E9%93%BE)来看，是这样的：

> **区块链**（英语：blockchain 或 block chain）是用分布式数据库识别、传播和记载信息的智能化对等网络，也称为价值互联网。

用简单的话来说，区块链就是一个全球一起写的一本书。这本书有可能是账本，也可能是别的什么东西。

区块链有以下几个优势：

1. 不可修改，数据写进区块就不能更改了。
2. 单向性，你只能在最后一个块后面加新的块，而不能在之前区块中间加新块。
3. 众生平等，任何人只要有合适的客户端就可以入网，根本没有限制。谁能出新块完全取决于共识算法。
4. 去中心化，每个人都是客户端，每个人也都是服务端，每个人都拿着所有人的账本，所以这是完全的去中心化。

区块链的去中心化是真正的去中心化，没有管理员，唯一一种有效的攻击手段就是拿到51%以上的算力，这在现在这么多人参与的情况下几乎是不可能的。而且，这种攻击仅仅能拿来控制新出块的内容，并不能改动之前的老块。

<!--more-->

# 区块

想了解区块链的本质，我们需要知道什么是区块。

一个区块分为区块头和数据两部分。

## 区块头

区块头包含当前块的一些描述数据，一般来说有以下内容

1. 上一区块的整体HASH
2. 当前区块的生成时间
3. 当前区块中数据的HASH
4. 负责让大家猜的随机数NONCE

HASH就是哈希算法，是一种数据摘要，MD5/SHA1之类的就算是HASH的一种。一个合格的HASH算法能做到只要修改原始数据一比特，哈希得到的结果都会面目全非，这就是密码学中的雪崩效应，具体不多讲感兴趣的可以搜一下雪崩效应。而且，两份不同数据得到同样哈希结果也是几乎不可能的，这是哈希的抗碰撞性。而且，哈希是不可逆的（谁发明了这种任意数据压缩成256bit还能解压回去的技术，今年的图灵奖就是你的了。）

## 区块数据

区块数据就是承载区块链的实际应用了。如果你在里面记录历史事件，这就是历史链；如果你用来记录交易账目，这就是金融链；如果你用来记录溯源信息，这就是溯源链。能做什么全看你的想象力了。

# 链

区块链之所以是区块链，主要是这条链。

区块头中包含上一块的HASH，意味着你的区块不能提前生成，只有知道上一个区块的HASH才行。

而且你也不能修改链上的某个块，因为你修改了区块数据，HASH会因为雪崩效应变得面目全非。后面的链就不知道前面的链是哪个了，这条链也就断了。在实际应用中，生效的永远是最长的那条链。也就是说，想要修改数据，你需要同时修改该块和以后的所有数据，还得让全世界都承认你的修改（让你的链比别人都长），这简直就是不可能的事情。

# 共识

为什么，你的块提交了别人就要收下并信任呢？这就涉及到了节点之间的共识问题。当前主流的就是比特币的工作量证明。原理简单来说可以理解为：求一个特别难的密码学题目。其中最适合的就是计算哈希。区块链用的是SHA256算法，可以把任意数据计算成256bit。举个简单的例子，给定一个HASH值，要求区块头计算的HASH小于这个值，这个区块才会生效。你要做的就是猜出来NONCE（实际上真实的链有一套自己的难度计算体系控制出块速率，动态调节HASH的门槛）。

因为哈希的单向性，任何人都可以验证你的答案对不对。但是你没法反推，只能一点一点的改变数据去计算HASH做碰撞（这也是大家诟病的地方，因为除了作为工作量证明以外，算出来的数真的没有什么用处）。这就是数字时代的挖矿。

你的结果传给全网，他们计算后发现答案正确，就把你的这一块加入了他的区块链里。这个操作是全世界同步的。为什么你不能修改链就在这，你做不到那么短时间内计算好N多块的哈希和答案，也就没法让其他人也按照你的要求修改链了。

中本聪设计了一套动态调整出块难度的算法，使得出块时间维持在10分钟左右。就是通过调节HASH门槛来实现的。

当两个人同时拿到了正确答案怎么办？主要还是看谁快。你早拿到答案早广播，他们早采纳。谁先到6个块以上就采取谁的链，以最长的为准，这就是共识算法。

# 后记

事实上，区块链的应用场景十分有限。他出块速度慢，VISA的全球支付网络每秒钟都要处理几万笔交易，用区块链每块几MB完全不够用的，等10分钟出一个块，一个块存不了几笔交易，那就非常耽误事。你等着你的数据同步到全世界最少也得等1小时再去确认是不是真正的写链了，这对金融行业也是不太现实的事。而且计算哈希很巧妙，从数学上证明了安全，但是算力白白浪费也是一种浪费。

目前区块链我觉得做的最好的应用场景还是电子货币，银行银行之间，银行客户之间，客户客户之间拿来交换数据，让密码学保证数据安全性和不可篡改性。当然，要做好这些还是需要修改链的出块速度和块大小，否则永远不能愉快的商用。

想象一下，现在比特币上用的区块链拿来做VISA体系的话，你刷卡之后交易得排队等着确认，然后全世界每10分钟确认几万笔，你买一件衣服可能确认交易要等2个月，你会用嘛？