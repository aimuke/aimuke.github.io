---
title: "OpenSSL 公钥、私钥以及自签名证书"
tags: [https, openssl, ca, certificate, "证书", "公钥", "私钥"]
---

[原文 OpenSSL 公钥、私钥以及自签名证书](https://www.zybuluo.com/muyanfeixiang/note/392079)

# 简介

公钥私钥用来互相加解密的一对密钥，一般是采用 `RSA` 非对称算法。公钥解密的私钥能解密，私钥加密的公钥能解密。关于公私钥更多内容，网上都有就不详细介绍。

关于CA证书,是由第三方机构下发的，可以对私钥签名附加上机构信息。浏览器就可以在https协议中通过该证书信任第三分站点。当然也可以自己生成跟证书来签发子证书。

# 操作

## 生成根证书

1.首先生成私钥

```sh
openssl genrsa -out root.key 2048
```

该命含义如下：

- `genrsa` 使用RSA算法产生私钥
- `-aes256` 使用256位密钥的AES算法对私钥进行加密(以忽略)
- `-out` 输出文件的路径
- `2048` 指定私钥长度

2.生成根证书签发申请文件(csr文件)

```sh
openssl req -new -key root.key -out root.csr -subj "/C=CN/ST=ShangHai/L=ShangHai/O=Yunan International Trust Company/OU=Internet Finance/CN=YNTRUST"
```

该命令含义如下：

- `req` 执行证书签发命令
- `-new` 新证书签发请求
- `-key` 指定私钥路径
- `-out` 输出的csr文件的路径
- `-subj` 证书相关的用户信息(subject的缩写)

3.自签发根证书(cer文件)

```sh
openssl x509 -req -days 3000 -sha1 -extensions v3_ca -signkey root.key -in root.csr -out root.cer
```

该命令的含义如下：
- `x509`——生成x509格式证书
- `-req`——输入csr文件
- `-days`——证书的有效期（天）
- `-sha1`——证书摘要采用sha1算法
- `-extensions`——按照openssl.cnf文件中配置的v3_ca项添加扩展
- `-signkey`——签发证书的私钥
- `-in`——要输入的csr文件
- `-out`——输出的cer证书文件

使用的 `-extensions` 值为 `v3_ca`，`v3_ca`中指定的 `basicConstraints` 的值为`CA:TRUE`，表示该证书是颁发给CA机构的证书。

到此就完成了根证书的生成。

## 使用根证书签发服务端证书

和生成根证书的步骤类似，这里就不再介绍相同的参数了。

1.生成服务端私钥(可以给商户客户)
```sh
openssl genrsa  -out private/server-key.pem 2048
```

2.生成证书请求文件

```sh
openssl req -new -key private/server-key.pem -out private/server.csr -subj "/C=CN/ST=ShangHai/L=ShangHai/O=Yunan International Trust Company/OU=Internet Finance/CN=YNTRUST"
```

3.使用根证书签发服务端证书(此时证书只有公钥，没有私钥)
```sh
openssl x509 -req -days 3000 -sha1 -extensions v3_req -CA root_cert/root.cer -CAkey root_cert/root.key -CAserial ca.srl -CAcreateserial -in private/server.csr -out private/server.cer
```

这里有必要解释一下这几个参数：
- `-CA`——指定CA证书的路径
- `-CAkey`——指定CA证书的私钥路径
- `-CAserial`——指定证书序列号文件的路径
- `-CAcreateserial`——表示创建证书序列号文件(即上方提到的serial文件)，创建的序列号文件默认名称为 `-CA`，指定的证书名称后加上 `.srl` 后缀

> 注意：这里指定的`-extensions`的值为 `v3_req`，在OpenSSL的配置中， `v3_req` 配置的 `basicConstraints` 的值为 `CA:FALSE`

4.将上一步的证书转换为 der 格式（二进制格式，不能文本打开）

```sh
openssl x509 -outform der -in test-private/server.cer -out private/publicserver.ccertificate.der
```

# 证书格式的转换

以上生成的公私钥和证书都是 `PEM` 格式的，但很多时候不同场景中还需要用到其他格式的证书：

```sh
p12／pfx
```

`p12/pfx` 是按照 `PKCS#12` 编码的对象，它通常由 `X.509` 证书和对应的私钥组成。生成 `p12` 格式的方法如下：

```sh
openssl pkcs12 -export -in server.cer -inkey server-key.pem -out server.pfx
```

由于文件中保存了私钥，因此执行该命令，openssl会要求用户输入密码，用于保护私钥。

# 从私钥导出公钥

```sh
openssl rsa -in server-key.pem -pubout -outform PEM -out server-key-pub.pem
openssl pkcs8 -topk8 -inform PEM -outform DER -in filename -out filename -nocrypt
```

# pfx to pem

```sh
openssl pkcs12 -in file.pfx -out file.pem -nodes
```
certificate to public key
```sh
openssl x509 -pubkey -noout -in cert.pem > pubkey.pem
```
https://www.sslshopper.com/ssl-converter.html

# References

- [原文 OpenSSL 公钥、私钥以及自签名证书](https://www.zybuluo.com/muyanfeixiang/note/392079)
