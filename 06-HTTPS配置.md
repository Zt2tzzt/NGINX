# HTTPS 配置

HTTPS 协议是 HTTP 协议的安全版本。

它通过对传输数据的加密，来保证数据的安全性。

HTTP 协议的默认端口，默认是 80；

HTTPS 协议的默认端口，默认是 443.

HTTPS 协议，需要使用到 SSL 证书，在主流云平台上都可以申请到。

SSL 证书申请成功后，会得到「证书文件」和「密钥文件」。

也可用过 openssl，来生成一个自签名的证书，使用如下三个命令：

生成私钥文件（private.key）

```shell
openssl genrsa -out private.key 2048
```

根据私钥生成证书签名请求文件（Certification Signing Request，简称 CSR 文件）

```shell
openssl req -new -key private.key -out cert.csr
```

使用私钥对证书申请进行签名，从而生成证书文件（pem 文件）

```shell
openssl x509 -req -in cert.csr -out cacert.pem -signkey private.key
```

执行命令时，会要求输入以下信息：比如国家，省份，城市，公司，邮箱等。

填写完成后，就会生成两个文件：

- 一个是私钥文件：private.key
- 另一个是证书文件：cacert.pem

这两个文件，要放在服务器上，然后再 nginx.conf 中进行配置。