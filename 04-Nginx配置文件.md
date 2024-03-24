# Nginx 配置文件

## 一、worker 进程

worker 进程的数量，可以通过配置文件的修改。

在 nginx.conf 文件中，找到如下代码快：

```nginx
worker_processes 1;
```

更改 worker 进程数：

```nginx
worker_processes 10;
```

保存文件。

并输入如下命令，查看配置文件是否保存成功。

```shell
nginx -t
```

nginx 配置文件修改后，需要重启服务。

```shell
nginx -s reload
```

一般来说，worker 进程的数量，保持与服务器 CPU 内核的数量相同，是比较合适的。

在 nginx.conf 中，修改如下配置：

```nginx
worker_processes auto;
```

重启 nginx

```shell
nginx -s reload
```

## 二、代码快分析

nginx 配置文件，主要由三个全局代码快组成。

### 1.全局块

最外层是全局块，

其中主要是修改 worker 进程的代码块、指定运行服务的用户...配置。

```nginx
#user  nobody;
worker_processes 1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;
```

### 2.events 块

events 代码块，主要是服务器和客户端之间网络连接的一些配置。

比如：制定每个 worker 进程，同时接收多少个网络连接、网络 IO 模型等等

```nginx
events {
  worker_connections 1024;
}
```

### 3.http 块

http 代码块，是 nginx.conf 文件中，修改最频繁的部分。

比如：虚拟主机、反向代理，负载均衡等等配置的修改。

```nginx
http {
  include mime.types;
  default_type application/octet-stream;

  #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
  #                  '$status $body_bytes_sent "$http_referer" '
  #                  '"$http_user_agent" "$http_x_forwarded_for"';

  #access_log  logs/access.log  main;
  sendfile on;
  #tcp_nopush     on;

  #keepalive_timeout  0;
  keepalive_timeout 65;

  #gzip  on;

  server {
    listen 80;
    server_name localhost;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;
    location / {
      root html;
      index index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
      root html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}
    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
  }


  # another virtual host using mix of IP-, name-, and port-based configuration
  #
  #server {
  #    listen       8000;
  #    listen       somename:8080;
  #    server_name  somename  alias  another.alias;
  #    location / {
  #        root   html;
  #        index  index.html index.htm;
  #    }
  #}
  # HTTPS server
  #
  #server {
  #    listen       443 ssl;
  #    server_name  localhost;
  #    ssl_certificate      cert.pem;
  #    ssl_certificate_key  cert.key;
  #    ssl_session_cache    shared:SSL:1m;
  #    ssl_session_timeout  5m;
  #    ssl_ciphers  HIGH:!aNULL:!MD5;
  #    ssl_prefer_server_ciphers  on;
  #    location / {
  #        root   html;
  #        index  index.html index.htm;
  #    }
  #}
  include servers/*;
}
```

http 块中，又可以包含多个 server 块；也叫“虚拟主机”，

最后一行的 ` include servers/*;` 表示把 servers 目录下的所有配置文件，都加载进来。

这样每个虚拟主机的配置，都可以放在单独的文件里面，让主配置文件，更加简洁清晰。

include 指令，还可以包含其他配置文件，

比如上方的 ` include mime.types;`，就是把 mime.types 这个文件，加载进来。

查看 mime.type 文件，可以看到其中定义了很多 MIME 的类型。

```nginx
types {
    text/html                                        html htm shtml;
    text/css                                         css;
    text/xml                                         xml;
    image/gif                                        gif;
    image/jpeg                                       jpeg jpg;
    application/javascript                           js;
    application/atom+xml                             atom;
    application/rss+xml                              rss;
    ...
}
```

这样，Nginx 就可以根据文件的后缀名，来判断文件的类型，然后根据不同的文件类型，做不同的处理。

比如：html 文件，就会被当作文本文件来处理；jpg 文件，就会被当作二进制文件来处理。
