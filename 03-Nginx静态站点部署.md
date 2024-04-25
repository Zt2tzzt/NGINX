# Nginx 静态站点部署

## 一、配置文件位置查看

在命令行输入以下命令：

```shell
nginx -V # 大写的 V
```

输出：

```shell
zetian@ZeTiandeMacBook-Pro ~ % nginx -V
nginx version: nginx/1.25.4
built by clang 15.0.0 (clang-1500.1.0.2.5)
built with OpenSSL 3.2.1 30 Jan 2024
TLS SNI support enabled
configure arguments: --prefix=/opt/homebrew/Cellar/nginx/1.25.4 --sbin-path=/opt/homebrew/Cellar/nginx/1.25.4/bin/nginx --with-cc-opt='-I/opt/homebrew/opt/pcre2/include -I/opt/homebrew/opt/openssl@3/include' --with-ld-opt='-L/opt/homebrew/opt/pcre2/lib -L/opt/homebrew/opt/openssl@3/lib' --conf-path=/opt/homebrew/etc/nginx/nginx.conf --pid-path=/opt/homebrew/var/run/nginx.pid --lock-path=/opt/homebrew/var/run/nginx.lock --http-client-body-temp-path=/opt/homebrew/var/run/nginx/client_body_temp --http-proxy-temp-path=/opt/homebrew/var/run/nginx/proxy_temp --http-fastcgi-temp-path=/opt/homebrew/var/run/nginx/fastcgi_temp --http-uwsgi-temp-path=/opt/homebrew/var/run/nginx/uwsgi_temp --http-scgi-temp-path=/opt/homebrew/var/run/nginx/scgi_temp --http-log-path=/opt/homebrew/var/log/nginx/access.log --error-log-path=/opt/homebrew/var/log/nginx/error.log --with-compat --with-debug --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_degradation_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_v3_module --with-ipv6 --with-mail --with-mail_ssl_module --with-pcre --with-pcre-jit --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module
```

以上输出可以查看到 Nginx 的

- 安装目录，即 `--prefix=/opt/homebrew/Cellar/nginx/1.25.4`

- 配置文件目录，即 `--conf-path=/opt/homebrew/etc/nginx/nginx.conf`

使用以下命令，验证配置文件语法，查看配置文件位置：

```shell
nginx -t
```

输出：

```shell
zetian@ZeTiandeMacBook-Pro ~ % nginx -t
nginx: the configuration file /opt/homebrew/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /opt/homebrew/etc/nginx/nginx.conf test is successful
```

## 二、配置文件分析

使用 VSCode 打开配置文件。

在 nginx.conf 文件中，找到以下代码快：

```nginx
server {
  listen 80;
  server_name localhost;

  location / {
    root html;
    index index.html index.htm;
  }
}
```

`localhost /` ，表示匹配在浏览器中输入的 url 中的路径 `/`，即没有路径。

`root html`，表示 `localhost /` 匹配的 Nginx 的安装目录（根目录）下的 html 目录。

`index index.html index.htm;`，表示匹配 `html` 目录下，`index.html` 或 `index.htm` 文件

### 1.location 路径匹配

`location` 指令，不仅要匹配 url，还要匹配服务器上对应的目录，比如将上面的配置改为如下：

```nginx
server {
  listen 80;
  server_name localhost;

  location /app {
    root html;
    index index.html index.htm;
  }
}
```

再访问 `localhost/app`，nginx 会返回 404，因为 html 目录下，没有 app 目录。

在 html 目录下，创建 app 文件夹，将 index.html 文件放入其中，再请求 `localhost/app`，还是有问题。因为这么写，nginx 会去找目录下的 app 文件。正确的请求方式应该是：`localhost/app/`，或 `localhost/app/index.html`.

### 2.location 路径精确匹配

然而，我们会发现，当请求 url 是 `localhost/apple/` 时，也能正确返回。这是因为上面的配置可以匹配所有“app”前缀的路径。

如果要设置精确匹配，应该进行如下配置：

```nginx
server {
  listen 80;
  server_name localhost;

  location = /app {
    root html;
    index index.html index.htm;
  }
}
```

这样，就只能匹配 `localhost/app/` 的请求方式了。

### 3.location 正则匹配

在 loacation 和 路径之间，使用 ～，启用正则表达式。

```nginx
server {
  location ~ /videos/video[6-9].avi {
    root /var/www/localhost;
  }
}
```

以上配置，表示匹配 /var/www/localhost/videos 下 video6.avi、video7.avi、video8.avi、video9.avi 三个文件。

`~*` 表示不区分文件名的大小写，但如果要正确匹配文件，请求 url 中仍然要严格区分目录和文件名单大小写。。

### 4.location 匹配优先级

1. `=`，表示精确匹配。
2. `^~`，表示优先前缀匹配（功能和前缀匹配类似）。
3. `~` 或 `~*` ，表示正则表达式匹配。
4. 空格，表示普通匹配。

### 5.location 重定向匹配

在 location 中，进行请求的重定向：为请求 `localhost/temp` 返回服务器上 /app/index.html  文件。

```nginx
server {
  location /temp {
    return 307 /app/index.html
  }
}
```

以上重定向配置，会改变请求 url 的地址，如果要达到更隐蔽的重定向请求效果（用户一般看不出来），应该进行如下配置：

```nginx
server {
  listen 80;
  server_name localhost;
  root /var/www/location

  rewrite /temp /app/index.html
}
```

以上配置，表示为请求 `localhost/temp` 返回服务器上 /app/index.html  文件

重定向、重写，主要针对单个文件路径。如果要让 Nginx 有多个文件可以选择，需要使用 try_files

```nginx
server {
  location / {
    try_files $uri $uri/ =404
  }
}
```

以上配置表示，先尝试匹配 url 中单文件名，如果文件名不存在则匹配 ur 中单目录名下单 index.html，如果也不存在则返回 404.

配置 404 页面

```nginx
server {
  location {
    error_page 404 /404.html
  }
}
```

以上配置，表示当出现 404 时，返回 /404.html 文件。

### 6.localtion 配置图片静态资源

```nginx
server {
  location /images/ {
  	alias /opt/temp/;
    autoindex on;
  }
}
```

- `location /images/`：这是图片资源的 URL 路径，您可以根据需要进行修改。
- `root /opt/temp/`：这是图片文件夹的实际路径，确保与您创建的文件夹路径一致。
- `autoindex on;`：此选项表示自动创建索引，允许浏览文件夹中的图片。

现在，您可以通过访问以下链接来查看您上传的图片 `ip地址/images/1.jpg`

## 三、Nginx 的安装目录（根目录）分析

输入以下命令，进入 Nginx 的安装目录：

```shell
cd /opt/homebrew/Cellar/nginx/1.25.4
```

输入以下命令，查看该目录下的内容

```shell
ls -ltr
```

输出：

```shell
zetian@ZeTiandeMacBook-Pro 1.25.4 % ls -ltr
total 680
drwxr-xr-x  4 zetian  admin     128  2 15 00:03 share
-rw-r--r--  1 zetian  admin      49  2 15 00:03 README
-rw-r--r--  1 zetian  admin    1397  2 15 00:03 LICENSE
-rw-r--r--  1 zetian  admin  326027  2 15 00:03 CHANGES
drwxr-xr-x  3 zetian  admin      96  2 19 17:57 bin
-rw-r--r--  1 zetian  admin     202  2 19 17:57 homebrew.nginx.service
-rw-r--r--  1 zetian  admin     685  2 19 17:57 homebrew.mxcl.nginx.plist
lrwxr-xr-x  1 zetian  admin      16  2 19 17:57 html -> ../../../var/www
-rw-r--r--  1 zetian  admin    1266  2 19 17:57 INSTALL_RECEIPT.json
```

可以看到，确实有 html 文件夹。

输入以下命令，进入 html 目录，

```shell
cd html
```

查看该目录下的内容

```shell
ls -ltr
```

输出：

```shell
zetian@ZeTiandeMacBook-Pro html % ls -ltr
total 16
-rw-r--r--  1 zetian  admin  615  2 15 00:03 index.html
-rw-r--r--  1 zetian  admin  497  2 15 00:03 50x.html
```

可以看到，确实有 index.html 文件，该文件，就是浏览器中看到的欢迎页。

## 四、Hexo 生成静态网站

Hexo 是一个基于 Node.js 的博客框架。

全局安装 Hexo

```shell
npm install hexo-cli -g
```

初始化

```shell
hexo init blog-demo
```

安装依赖

```shell
cd blog-demo;

pnpm install
```

本地运行

```shell
hexo server / hexo s
```

生成静态资源，

```shell
hexo g
```

把 Markdown 格式的文章，转成静态页面，放到 public 文件夹下。

将生成的静态站点，部署在 Nginx 上

输入下方命令，进入 public 目录，并拷贝该目录下的文件到 `/opt/homebrew/Cellar/nginx/1.25.4/html` 目录下：

```shell
cd public

cp -rf * /opt/homebrew/var/www
```

或者使用 hexo 一键部署的命令

```shell
hexo d
```

ChatGPT:

> `hexo d` 是 Hexo 静态站点生成器的一个命令，`d` 是 `deploy` 的缩写，整个命令可以看作 `hexo deploy`。
>
> `hexo deploy` 或 `hexo d` 命令的作用是部署你的网站。在你运行这个命令前，你需要在你的 Hexo 配置文件 `_config.yml` 中设定部署的相关配置。
>
> 具体执行过程中，Hexo 会根据配置文件中的 `deploy` 选项来确定部署的目的地。部署的目的地可以是 GitHub、GitLab、Heroku 等多种类型的远程主机。
>
> 例如，如果你的 `_config.yml` 文件中的 `deploy` 选项如下所示：
>
> ```yaml
> deploy:
>   type: git
>   repository: git@github.com:username/username.github.io.git
>   branch: master
> ```
>
> 运行 `hexo deploy` 或 `hexo d` 命令时，Hexo 会将生成的静态网站内容推送到上述配置的 GitHub 仓库。
>
> 需要注意的是，在运行 `hexo deploy` 命令之前，通常需要先运行 `hexo generate` 或其简写形式 `hexo g` 来生成静态网站内容。或者你可以将两者连起来一起运行，如 `hexo g -d`，这将会生成并部署网站。
