# HTTPS 配置

## 一、HTTP 和 HTTPS 

HTTPS 协议是 HTTP 协议的安全版本。

它通过对传输数据的加密，来保证数据的安全性。

HTTP 协议的默认端口，是 80；

HTTPS 协议的默认端口，是 443.

HTTPS 协议，需要使用到 SSL 证书，在各主流云平台上都可以申请到。

SSL 证书申请成功后，会得到「证书文件」和「密钥文件」。

## 二、自签名 ssl 证书

也可用 openssl，来生成一个自签名的证书（该证书没有得到 CA 机构的认证，浏览器会提示不安全）。

使用如下三个命令：

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

执行命令时，会要求输入以下信息：比如国家，省份，城市，公司，邮箱等等。

填写完成后，就会生成两个关键文件：

- 私钥文件：private.key
- 证书文件：cacert.pem

这两个文件，要放在服务器上，然后在 nginx.conf 中进行配置。

## 三、配置 Nginx

```nginx
http {
  server {
    listen 443 ssl;
    server_name localhost;
    # 证书文件名
    ssl_certificate /opt/homebrew/etc/nginx/cacert.pem;
    # 证书私钥文件名
    ssl_certificate_key /opt/homebrew/etc/nginx/private.key;
    # ssl 验证配置
    ssl_session_timeout 5m; # 缓存有效期
    # 安全链接可选的加密协议
    ssl_protocols SSLv2 SSLv3 TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    # 配置加密套件/加密算法，写法遵循 openssl 标准
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    # 使用服务器端端首选算法
    ssl_prefer_server_ciphers on;
  }
}
```

listen 后面，加上 443 端口和 ssl；

server_name 后面，一般是网站域名。

后面的加密算法配置，一般都是固定的。

重载 nginx 配置

```shell
nginx -s reload
```

在浏览器中，访问 `https://localhost/`

---

一般还会在 nginx 中，配置 http 请求重定向到 https 上。

```nginx
http {
	server {
    listen 80;
    server_name localhost; # 域名
    return 301 https://$server_name$request_uri;
  }
}
```

301 是 Http 响应状态吗，表示请求资源的 URL 已经修改，响应中会给出新的 URL

重载 nginx 配置

```shell
nginx -s reload
```

