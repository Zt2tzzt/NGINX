# 虚拟主机

虚拟主机，可以在一台服务器上，部署多个站点。

很多时候，网站在起步阶段，没有多大访问量。

将多个网站，部署在一台服务器上，也不会对服务器造成太大的压力，同时可以节省服务器资源和成本。

Nginx 中的虚拟主机，是通过 server 块来实现的。

通过 server_name 来指定虚拟主机的命名。

这样，当我们访问这个域名时，就会被 server 块所匹配，然后执行这个 server 块中的配置。

## 一、虚拟主机创建

将 `nginx.conf` 文件中的 `https` 相关 `server` 块，剪切一份；

然后在 server 目录下创建一个文件 `local.conf`，在其中，粘贴复制的内容。

servers/local.conf

```nginx
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

  location /app {
    proxy_pass http://backend;
  }

  location / {
    root html;
    index index.html index.htm;
  }

  error_page 500 502 503 504 /50x.html;
  location = /50x.html {
    root html;
  }
}
```

## 二、另外一个虚拟主机的创建

初始化一个 Vue 项目。安装依赖，并进行打包。

在 servers 目录下，创建一个虚拟主机配置文件 `vue.conf`.

```nginx
server {
  listen 9003;
  server_name localhost;

  # 指定虚拟主机的根目录，也就是打包好的 vue 项目文件目录
  location / {
    root /Users/zetian/workshop/project/vue-nginx-demo/dist;
    index index.html index.htm;
  }
}
```

重载 nginx 配置

```shell
nginx -s reload
```

在浏览器中，访问 `localhost:9003`

## 三、Nginx 使用同源策略解决跨域

Nginx 可以在一个端口下，既部署静态资源，又反向代理后端服务。

这可以在虚拟主机中，通过定义不同的 `location` 来实现。

下方示例，适配了在同一端口提供静态文件服务和反向代理服务的场景：

复制代码

```nginx
server {
  listen 80;
  server_name localhost;

  location / {
    root /var/www/html; # 这里填写你的静态文件路径
    index index.html index.htm; # 这里填你的默认首页文件名
  }

  location /api/ {
    proxy_pass http://localhost:8080/; # 这里填写你的后端 API 服务地址
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

在上方示例中:

- 当用户访问 `http://yourdomain.com/` 时，Nginx 将会服务 `/var/www/html` 目录下的静态文件。
- 当用户访问 `http://yourdomain.com/api` 时，请求将会被代理到 `http://localhost:8080`。

这么做，相当于用同源策略，解决了浏览器的跨域问题。

这样配置后，记得使用 `nginx -t` 来检测你的配置是否有语法错误，然后重启 Nginx 让新的配置生效。

## 四、Nginx 使用 CORS 解决跨域

使用 Nginx 创建一个虚拟主机，在其中监听 8000 端口，将该端口接收的请求，代理到 9000 端口的 api 服务上。

```nginx
server {
  listen 8000;
  server_name localhost;

  location / {
    # root   html;
    # index  index.html index.htm;

    # 简单请求
    add_header Access-Control-Allow-Origin *;
    # 非简单请求
    add_header Access-Control-Allow-Headers "Accept, Accept-Encoding, Accept-Language, Connection, Content-Length, Content-Type, Host, Origin, Referer,User-Agent";
    add_header Access-Control-Allow-Credentials true;
    add_header Access-Control-Allow-Methods "PUT, POST, GET, DELETE, PATCH, OPTIONS";
    if ($request_method = "OPTIONS") {
      return 204;
    }

    proxy_pass http://localhost:9000; # API 服务器的源
  }
}
```

更加完整的写法：

```nginx
server {
  listen 80;
  server_name localhost;

  location / {
    add_header 'Access-Control-Allow-Origin' '*';
    add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
    add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
    add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';

    # 如果需要带有 Cookie 的跨域请求，需要设置下面这行
    add_header 'Access-Control-Allow-Credentials' 'true';

    # 如果有预检请求（preflight）也要添加如下配置
    if ($request_method = 'OPTIONS') {
      add_header 'Access-Control-Allow-Origin' '*';
      add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
      add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
      add_header 'Access-Control-Max-Age' 1728000;
      add_header 'Content-Type' 'text/plain charset=UTF-8';
      add_header 'Content-Length' 0;
      return 204;
    }

    # 添加其他反向代理配置
    proxy_pass http://localhost:9000;
  }
}
```
