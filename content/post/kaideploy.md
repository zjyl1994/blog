---
title: "KaiDeploy 开发笔记"
date: 2020-03-22T23:31:00+08:00
lastmod: 2020-03-22T23:31:00+08:00
draft: false
tags: ["KaiOS"]
categories: ["KaiOS"]

---

> 折腾 KaiOS 的我又搞了一个没什么人会用到的小工具

# 引子
最近折腾完 KaiAuth，就在 KaiOS 的交流群里和他们吹水。总能发现小白玩家连基本的操作都不懂却被迫折腾 adb，然后和 Palemoon 之类的过时火狐战斗，才能把应用装上。（有大把的人连 adb 都找不到）

Kai OS 脱胎与 Firefox OS，应用要用当年 Firefox 的 Web IDE 才能 sideload 到手机。由于 Mozilla 已经在 2016 年停止开发 Firefox OS， 新版本 Firefox 逐渐拿掉了自己不需要的 Web IDE，这却给我们这些 Kai OS用户造成蛮大的困扰。为了能随时 sideload 新的 app 进手机，必须保留最后一代有 Web IDE 的浏览器。这很不方便，一不小心就自动更新了。

<!--more-->

# 折腾

## 破解协议

首先，Web IDE 是用 ADB 导出的调试端口进行通讯的，标准端口就是`localhost:6000`，这是一个很大的突破点。

为了抓到中间的通讯内容，我们需要一个转发器，并且能记录所有中间流过的数据。我选择了 sokit ，毕竟这种东西没必要自己写。

打开 sokit 的转发功能，打开记录到日志，这时候形成了一个 `Web IDE <-tcp-> 转发器 <-tcp-> ADB <-USB电缆-> 手机` 的连接隧道。

打开 Web IDE ，我们会抓到非常多的数据包，不利于解析。所以我选择了一个更取巧的办法。

Kai OS 曾经有过一款部署工具，叫做 `kdeploy`，不知道为什么被官方删除了。[Lux Ferre](https://gitlab.com/suborg) 大佬开发了一个替代品，https://gitlab.com/suborg/gdeploy。可以直接通过命令行抓包推送软件包到手机。

现在，通讯隧道变成了  `gdeploy <-tcp-> 转发器 <-tcp-> ADB <-USB电缆-> 手机` 。完成一次通讯就可以抓到协议报文了。

## 协议分析

报文太长了，我就不贴在这里了，有兴趣的可以自己抓一下。我这里直接说一下结论，如果你有一定的基础看懂那个报文很简单。

协议体是这样的：`json长度:json体` (来回的信息都是这样的，最简单的例子就是这样的 `2:{}`)

首先，打开 TCP 端口，手机会主动上报一条消息，内容是操作系统支持的调试功能。这个对部署应用没有什么意义，可以忽略。

然后，`gdeploy` 发送：

```json
31:{"to":"root","type":"listTabs"}
```

得到手机响应：

```json
452:{"from":"root","selected":0,"tabs":[],"memprofActor":"server1.conn9.memprofActor1","preferenceActor":"server1.conn9.preferenceActor2","actorRegistryActor":"server1.conn9.actorRegistryActor3","settingsActor":"server1.conn9.settingsActor4","webappsActor":"server1.conn9.webappsActor5","deviceActor":"server1.conn9.deviceActor6","directorRegistryActor":"server1.conn9.directorRegistryActor7","heapSnapshotFileActor":"server1.conn9.heapSnapshotFileActor8"}
```

结合上下文，我们发现对于应用部署最有用的就是 `webappsActor` 这个里面的值，记录下来。

电脑端发送以下报文：

```json
59:{"to":"server1.conn9.webappsActor5","type":"uploadPackage"}
```

其中，to 表明要发送给的 `actor`, type 必须为 `uploadPackage` 。

手机端响应：

```json
86:{"actor":"server1.conn9.packageUploadJSONActor9","from":"server1.conn9.webappsActor5"}
```

记下来 `actor` 中的值，后续上传会用到。

电脑端下行以下内容：

```json
43865:{"to":"server1.conn9.packageUploadJSONActor9","type":"chunk","chunk":"PK....."}
```

通过后面的几条报文，我们可以知道，`to` 是刚才拿到的 `actor`， `chunk` 是分片的 zip 包。（chunk 中的PK头可以很容易的知道这是一个 zip 包）

发送后手机端会响应以下：

```json
79:{"written":20480,"_size":102400,"from":"server1.conn9.packageUploadJSONActor9"}
```

其中 `written` 是本次写入的数据量，`_size`是整个 `uploadActor` 接收到的数据量。

多发几次把完整的zip包发过去就可以了

电脑端发送以下 json 通知手机上传完成:

```json
60:{"to":"server1.conn9.packageUploadJSONActor9","type":"done"}
```

手机端会回复一个没什么用的 ack 信息：

```json
48:{"from":"server1.conn9.packageUploadJSONActor9"}
```

电脑端下达安装指令给 `webappActor`, 这里的 `appId` 我查看了 `gdeploy` 里的实现，一层一层的扒代码进去得到就是个 `uuid.v1`，所以随便生成一个就好了。

```json
149:{"appId":"eb4942c0-6b89-11ea-b686-7fba009c94d8","upload":"server1.conn9.packageUploadJSONActor9","to":"server1.conn9.webappsActor5","type":"install"}
```

手机会返回以下响应：

```json
135:{"appId":"kaiauth.zjyl1994.com","path":"/data/local/tmp/b2g/eb4942c0-6b89-11ea-b686-7fba009c94d8","from":"server1.conn9.webappsActor5"}
```

如果你想在安装完成后和 Web IDE 一样自动打开应用，可以记住 `appId` 。

电脑端下行以下指令清除用过的 `uploadActor`

```json
62:{"to":"server1.conn9.packageUploadJSONActor9","type":"remove"}
```

**选做部分：**

以下内容是通过 Web IDE 安装流程抓取到的：

自动打开应用，`manifestURL` 的格式是这样的 `app://${appId}/manifest.webapp`，发给 `webappsActor`

```json
111:{"to":"server1.conn0.webappsActor5","type":"launch","manifestURL":"app://kaiauth.zjyl1994.com/manifest.webapp"}
```

手机端响应：

```json
114:{"from":"server1.conn0.webappsActor5","type":"appOpen","manifestURL":"app://kaiauth.zjyl1994.com/manifest.webapp"}
```

此时手机上会自动弹出你刚才安装的App界面。

## 实现踩坑

考虑到需要免除运行库，所以 `Golang` 就是最简单的解决方案了。

 ### 打包 Zip

网上能找到的 `Golang` 打包 zip 代码或多或少都有一些问题, 多数问题出现在他会把绝对路径压制进 zip。所以我修改了一个版本的 zip 打包方法，考虑到没必要留存zip包，所以说直接内存操作又快又方便，毕竟 Kai OS 应用也没见过超过 10 MB 的。

```go
func zipToMem(source string) (data []byte, err error) {
	buf := new(bytes.Buffer)
	archive := zip.NewWriter(buf)
	source, err = filepath.Abs(source)
	if err != nil {
		return nil, err
	}
	info, err := os.Stat(source)
	if err != nil {
		return nil, err
	}
	if !info.IsDir() {
		return nil, errors.New("source not dir")
	}
	filepath.Walk(source, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			return err
		}
		if info.IsDir() && info.Name() == ".git" {
			return filepath.SkipDir
		}
		header, err := zip.FileInfoHeader(info)
		if err != nil {
			return err
		}
		header.Name = strings.TrimPrefix(path, source)
		if info.IsDir() && header.Name == "" {
			return nil
		}
		header.Name = filepath.ToSlash(header.Name)
		if info.IsDir() {
			header.Name += "/"
		} else {
			header.Method = zip.Deflate
		}
		header.Name = strings.TrimPrefix(header.Name, "/")
		if *verboseFlag {
			fmt.Println(header.Name)
		}
		writer, err := archive.CreateHeader(header)
		if err != nil {
			return err
		}
		if info.IsDir() {
			return nil
		}
		file, err := os.Open(path)
		if err != nil {
			return err
		}
		defer file.Close()
		_, err = io.Copy(writer, file)
		return err
	})
	archive.Close()
	return buf.Bytes(), nil
}
```

简单来说就是通过 `filepath.Walk` 遍历文件夹里的文件，然后对路径做处理，考虑到 Windows 和 \*nix 系列的路径分隔符不一样，所以统一用 `filepath.ToSlash` 转换到 `/`，然后方便处理。

最后的结果就是打开 zip 包，所有文件都是相对路径模式存在。这样就正确了！

如果你的zip处理不正确，不是相对路径的，Kai OS 那边只会提示安装失败，并不会给你详细原因。

### 协议解析和组装

```go
func readJSON(r *bufio.Reader) (*simplejson.Json, error) {
	strLen, err := r.ReadString(delimByte)
	if err != nil {
		return nil, err
	}
	strLen = strings.TrimSuffix(strLen, ":")
	bJSON := make([]byte, atoi(strLen))
	_, err = io.ReadFull(r, bJSON)
	if err != nil {
		return nil, err
	}
	return simplejson.NewJson(bJSON)
}

func writeJSON(w io.Writer, json *simplejson.Json) error {
	bJSON, err := json.MarshalJSON()
	if err != nil {
		return err
	}
	buf := bytes.NewBufferString(itoa(len(bJSON)))
	err = buf.WriteByte(delimByte)
	if err != nil {
		return err
	}
	_, err = buf.Write(bJSON)
	if err != nil {
		return err
	}
	_, err = w.Write(buf.Bytes())
	return err
}
```

为了方便解析，所以输入输出都是 `simplejson` 对象。组装协议很简单，读取协议使用了 `bufio.ReadString` 实现了截取冒号前字符串的功能，要注意：截取到的字符串是包含冒号的，需要处理掉。

不得不说这个协议设计的还是很简单明了的，如果可以的话甚至可以用 `telnet` 进行操作，基本不需要二进制操作。一般来说我设计协议都是固定字节的包头，决定后面要读取多长的数据。这种文本协议解析速度肯定没有二进制协议快，但是胜在简单明了。

### chunk转换

本次最大的问题就在于，直接使用 Go 的 `json.Marshal` 处理字节数组，会处理成 Base64 ，如果协议是我设计的我肯定不会在 json 里直接写二进制，但是手机端的处理程序是固定的，只能针对性的修改。

通过查阅 https://www.json.org/json-zh.html ，可以知道 JSON 的字符串需要转义一定的内容。

![JSON string](https://www.json.org/img/string.png)

考虑到 ASCII 中有不少控制字符，最好的办法就是只显示可见字符。

可见字符的 ASCII 范围是 32~126，不可见字符用 `\u00FF` 表示。（FF是字节的十六进制值）

```go
func jsonEncodeBytes(byteArray []byte) *json.RawMessage {
	var sb strings.Builder
	sb.WriteString(`"`)
	for _, b := range byteArray {
		switch b {
		case 8:
			sb.WriteString(`\b`)
		case 9:
			sb.WriteString(`\t`)
		case 10:
			sb.WriteString(`\n`)
		case 12:
			sb.WriteString(`\f`)
		case 13:
			sb.WriteString(`\r`)
		case 34:
			sb.WriteString(`\"`)
		case 92:
			sb.WriteString(`\\`)
		default:
			if b >= 32 && b <= 126 {
				sb.WriteByte(b)
			} else {
				sb.WriteString(`\u00`)
				sb.WriteByte(hextable[b>>4])
				sb.WriteByte(hextable[b&0x0f])
			}
		}
	}
	sb.WriteString(`"`)
	result := json.RawMessage(sb.String())
	return &result
}
```

这样就能得到手机端需要的 Json 里面跑二进制流了。

# 一些废话

Kai OS 还是很有折腾潜力的，3月14日的时候 Mozilla 已经和 Kaiostech 达成协议。由 Mozilla 给 Kai OS 升级 Gecko 内核，
新闻在这里 https://www.kaiostech.com/press/kaios-technologies-and-mozilla-partner-to-enable-a-healthy-mobile-internet-for-everyone/。

重点在于，升级后的 Kai OS 速度会得到一系列更强的 Buff，毕竟现在的 Kai OS Gecko 还停留在 2016 的 48 版本。2017 年大把的特性更新还上了量子火狐，现在 Kai OS 一点没吃到。就是不知道未来的 HMD 能不能给 Nokia 2720 Flip 升级新内核了，估计很难。

![KaiOS-Mozilla](https://www.kaiostech.com/wp-content/uploads/KaiOS-Mozilla-Features.jpg)

最后，完整的代码在这里: https://github.com/zjyl1994/kaideploy

我已经借助 bat 批处理文件和 WinRAR 自解压做好了一个 自带 ADB 的 OmniSD 一键安装器，个人测试还是很方便使用的。