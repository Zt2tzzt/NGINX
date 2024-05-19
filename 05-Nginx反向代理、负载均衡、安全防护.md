# 反向代理、负载均衡、安全防护

## 一、正向代理和反向代理

反向代理，是相对于正向代理来理解的。

### 1.正向代理

正向代理，就是代理客户端；

> 当使用 VPN 访问墙外网站时，VPN 会代理我们的客户端，去访问这些网站，然后把访问的结果，返回给我们。

这里代理服务器（VPN）代理的是客户端，这个过程，客户端是明确的，而服务端是不知道的

### 2.反向代理

反向代理，就是代理服务端。

> 当我们访问一个网站，比如 google，它有成千上万台服务器，但是对外暴露的，只有一个域名；
>
> 我们在浏览器请求这个域名，该请求，会被转发到不同的服务器上；
>
> 这样，google 就隐藏了真实服务器 IP 地址，端口等信息。

这里的代理服务器（Nginx），代理的是服务端，这个过程，客户端是不知道的，对于服务端是明确的。

Nginx 就是一个反向代理服务器。

## 三、反向代理配置

修改 nginx.conf 配置文件，在 http 块中，添加反向代理的配置。

```nginx
http {
  upstream backend {
    server 127.0.0.1:8000；
    server 127.0.0.1:8001；
    server 127.0.0.1:8002；
  }
}
```

upstream 后面可以跟上一个任意的名字（比如 `backend`），表示代理的服务器的名字。

upstream 块里面，是需要被代理的服务器的配置。

上方示例中，表示代理本地服务器 8000，8001，8002 端口上的三个服务

在 http 的 server 块中，添加一个 location 配置

将所有 `/app` 开头的请求，代理到刚刚配置的 upstream 中

```nginx
http{
  upstream backend {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
  }
  server {
    listen 80;
    server_name localhost;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;
    location /app {
      proxy_pass http://backend;
    }
  }
}
```

proxy_pass 后面的地址，要和 upstream 配置的名称一致。

这样，一个反向代理的配置，就完成了。

重载 nginx 配置

```shell
nginx -s reload
```

打开浏览器，输入 `localhost/app`，并重复刷新页面。

发现请求被轮流代理到了 `127.0.0.1:8000`，`127.0.0.1:8001`，`127.0.0.1:8002` 上的服务。

### 1.健康检查

Nginx 会定期向定义的服务器发送健康检查请求。如果服务器响应正常，则认为服务器健康；如果服务器没有响应或者响应异常，则认为服务器不健康。当服务器被标记为不健康时，Nginx 将不再将请求转发到该服务器，直到它恢复健康。

```nginx
http {
  upstream myapp1 {
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;

    # 健康检查配置。
    # 每10秒进行一次健康检查，如果连续3次健康检查失败，则认为服务器不健康；
    # 如果连续2次健康检查成功，则认为服务器恢复健康。
    check interval=10s fails=3 passes=2;
  }

  server {
    listen 80;

    location / {
      proxy_pass http://myapp1;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
  }
}
```

> `check interval=10s fails=3 passes=2;` 这样的配置语法在开源版本的 NGINX 上是不支持的。这是 `ngx_http_upstream_check_module` 模块的特有语法，而该模块不包含在 NGINX 的开源版本中，需要自行下载、编译和安装。该模块是开源免费的，具体详情请参见 [ngx_http_upstream_check_module 文档](https://github.com/yaoweibin/nginx_upstream_check_module)。

---

案例分析：

```nginx
server {
  listen 80;
  server_name api.example.com;
  
  location / {
    proxy_pass http://backend;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

- 当客户端请求 `api.example.com` 时，Nginx 会将请求转发到后端服务器集群。
- 通过设置 `proxy_set_header`，可以修改客户端请求的头部信息，确保后端服务器能够正确处理请求。

## 四、负载均衡

Nginx 默认的反向代理，是轮询代理的。

有时，后台服务器的硬件配置不同，比如第一台服务器，比后两台服务器配置好。

那么就可以使用权重 weight 来调整负载均衡的策略。

### 1.轮询 weight 权重配置

权重轮询算法是 Nginx 默认的负载均衡算法。它按顺序将请求逐一分配到不同的服务器上。通过设置服务器权重（weight）来调整不同服务器上请求的分配率。

权重越大，被分配到的请求越多。

在 nginx.conf 文件中，进行配置。

```nginx
http {
  upstream backend {
    server 127.0.0.1:8000 weight=3;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
  }
}
```

以上配置，表示第一个服务器，收到的请求，是后面两个服务器的 3 倍。

重载 nginx 的配置

```shell
nginx -s reload
```

### 2.ip_hash 配置

这个策略，会根据客户端的 IP 地址，进行哈希运算；这样可以确保同一客户端的请求始终被分配到同一台服务器，有助于保持用户的会话状态。

```nginx
http {
  upstream backend {
    ip_hash;
    server 127.0.0.1:8000 weight=3;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
  }
}
```

重载 nginx 的配置

```shell
nginx -s reload
```

打开浏览器，输入 `localhost/app`，并重复刷新页面。

可以看到，现在无论怎么刷新，请求的都是同一台服务器。

### 3.fair 配置

根据服务器的响应时间和负载来分配请求。

```nginx
upstream backend {
  fair; # 启用公平调度算法
  server backend1.example.com;
  server backend2.example.com;
  server backend3.example.com;
}
```

它结合了轮询和I P 哈希的优点，但 Nginx 默认不支持公平调度算法，需要安装额外的模块（upstream_fair）来实现。

> `Nginx`拥有丰富的开源模块，有很多还有待我们探索，除了一些定制化的需求需要自己开发，大部分的功能都有开源。大家可以在 `NGINX` 社区、`GitHub` 上搜索 "nginx module" 可以找到。虽然前端人员可能不经常直接操作 Nginx，但了解其基本概念和简单的配置操作是必要的。这样，在需要自行配置 Nginx 的情况下，前端人员能够知晓如何进行基本的设置和调整。

### 4,URL Hash 配置

根据请求的 URL 的哈希值分配请求，每个请求的 URL会被分配到指定的服务器，有助于提高缓存效率。

```nginx
upstream backend {
  hash $request_uri; # 启用URL哈希算法
  server backend1.example.com;
  server backend2.example.com;
}
# 根据请求的 URL 哈希值来决定将请求发送到 backend1 还是 backend2。
```

这种方法需要安装 Nginx 的 hash 软件包。

---

案例分析：

```nginx
http {
  upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
  }
  server {
    listen 80;
    server_name example.com;
    
    location / {
      proxy_pass http://backend;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
  }
}
```

- Nginx 将请求分发给多个后端服务器。通过 `upstream` 指令定义了一个服务器组，
- 然后在 `location` 块中使用 `proxy_pass` 指令将请求代理到这个服务器组。
- Nginx 支持多种负载均衡策略，如轮询（默认）、IP 哈希等。

## 五、安全防护

```nginx
server {
  listen 80;
  server_name example.com;
  location / {
    # 防止 SQL 注入等攻击
    rewrite ^/(.*)$ /index.php?param=$1 break;
    # 限制请求方法，只允许 GET 和 POST
    if ($request_method !~ ^(GET|POST)$ ) {
      return 444;
    }
    # 防止跨站请求伪造
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
  }
}
```

- 通过 `rewrite` 指令，可以防止一些常见的 Web 攻击，如 SQL 注入。
- 这种限制请求方法，可以减少服务器被恶意利用的风险。
- 同时，添加了一些 HTTP 头部来增强浏览器安全，如防止点击劫持和跨站脚本攻击（XSS）等。
