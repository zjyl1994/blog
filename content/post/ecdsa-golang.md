---
title: "Golang 使用ECDSA进行数字签名"
date: 2022-07-27T22:26:00+08:00
lastmod: 2022-07-27T22:26:00+08:00
draft: false
---

> 数字签名可以检查数据是否遭到篡改

# 起因

我的工作中，开发了一个批量改写和执行 SQL 的数据处理服务。
待执行的 SQL 模板会按照迭代定期发布到服务中，决定了服务最终运行时生成的 SQL。

此处 SQL 模板和数据处理服务的关系，可以类比为固件和硬件。

服务的固件通过 HTTP 接口进行刷入。参考手机厂家的做法，
更新服务都会检查固件包的数字签名，如果固件签名不正确，是无法正确刷入并启动的。

# ECDSA

ECDSA是大名鼎鼎的椭圆曲线签名算法，以256位的私钥尺寸提供堪比RSA3072的安全性。

Golang 标准库中提供了 `crypto/ecdsa` ，提供了签名，验证和生成公私钥的功能。但是一般情况我们不会使用Go来生成和保存公私钥，更多情况是使用 OpenSSL 进行生成。

OpenSSL 生成公私钥的命令，一般我们使用`prime256v1`或`secp384r1`这两种推荐的曲线参数，这里我们选择`prime256v1`。

```bash
openssl ecparam -name prime256v1 -genkey -noout -out priv_key.pem
openssl pkey -in priv_key.pem -pubout -out pub_key.pem
```

生成的私钥`priv_key.pem`，公钥 `pub_key.pem`，都保存为x509格式的pem证书。

# 签名

数字签名，需要先对数据体进行哈希，得到信息摘要后，通过私钥进行签名。
整理后可以得到如下 Golang 函数：

```go
func Sign(r io.Reader, privKey []byte) (string, error) {
	// 加载x509格式的私钥文件
	block, _ := pem.Decode(privKey)
	if block == nil {
		return "", errors.New("privKey no pem data found")
	}
	pk, err := x509.ParseECPrivateKey(block.Bytes)
	if err != nil {
		return "", err
	}
	// 对输入进行哈希获取信息摘要
	h := sha256.New()
	_, err = io.Copy(h, r)
	if err != nil {
		return "", err
	}
	hash := h.Sum(nil)
	// ECDSA 签名
	sign, err := ecdsa.SignASN1(rand.Reader, pk, hash)
	if err != nil {
		return "", err
	}
	return base64.StdEncoding.EncodeToString(sign), nil
}
```

# 签名验证

验证时也需要先摘要后进行验证，注意，签名验证使用公钥，此公钥可以公开存放在服务内。整理后可以得到如下 Golang 函数：

```go
func Verify(r io.Reader, pubKey []byte, sign string) (bool, error) {
	// 加载 x509 格式的公钥
	block, _ := pem.Decode(pubKey)
	if block == nil {
		return false, errors.New("pubKey no pem data found")
	}
	genericPublicKey, err := x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		return false, err
	}
	pk := genericPublicKey.(*ecdsa.PublicKey)
	// 对输入进行哈希获取信息摘要
	h := sha256.New()
	_, err = io.Copy(h, r)
	if err != nil {
		return false, err
	}
	hash := h.Sum(nil)
	// ECDSA 验证
	bSign, err := base64.StdEncoding.DecodeString(sign)
	if err != nil {
		return false, err
	}
	return ecdsa.VerifyASN1(pk, hash, bSign), nil
}

```

# 使用

一般使用中，我选择通过 1.16 新增的 embed 将 x509 格式的证书包裹在二进制中，考虑到我的使用场景是纯网络服务，并不需要担心二进制在本地被破解，也就不需要对公钥部分进行过多的混淆。

在我的使用场景中，通过专用工具打包固件，再通过`Sign`函数生成签名。
将固件通过 HTTP 接口POST刷入，签名通过 HTTP Header 传入服务。


服务拿到固件后进行验证签名，签名不通过则返回 HTTP 403 Forbidden。
如果签名通过，则执行后续的解包刷固件功能。

---

这次的博文没有什么技术含量，更多是对工作中常用的工具函数进行总结。

后续我会单独搞一个 wiki 站存放这些东西。