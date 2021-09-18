---
title: "nginx + docker + letsencrpty 部署网站"
date: 2021-09-17T20:00:39+09:00
slug: "deploy-website-with-nginx-docker-letsencrypt"
dropCap: false
---

前端后端都稍微写过，也有一个基于 hugo 的静态博客，不过在部署网站方面都是靠别人或者工具自动化，真要自己从头部署一个网站还真是两眼一抹黑，抓瞎。所以记录一下部署一个 https 网站的折腾过程，给有需要的人。

##### 目标

- 在 ubuntu 系统上启动 nginx 容器
- 通过 Let's encrypt 得到 HTTPS 功能

##### 需要准备的东西

- 域名
- 服务器

##### 1. 环境安装

- 按照[文档](https://docs.docker.com/engine/install/ubuntu/#install-using-the-convenience-script)，安装 Docker
- 按照[文档](https://docs.docker.com/compose/install/)， 安装 Docker Compose

##### 2. 配置 nginx，开启HTTP通信

先准备好对应的文件，文件结构如下
```sh
└── example
    ├── html
    │   └── index.html                   # 主页
    ├── nginx
    │   ├── default.conf                 # 网站相关配置
    │   ├── nginx.conf                   # nginx配置
    │   ├── Dockerfile                   # 制作nginx容器的文件
    │   ├── letsencrypt                  # let's encrypt相关的文件放在这
    │   └── logs            
    │       ├── access.log               
    │       ├── certbot.log              # 证书更新的log
    │       └── error.log 
    └── docker-compose.yml               # 一键启动nginx的脚本
```

`Dockerfile`中的内容为
```docker
FROM nginx:latest                                             
RUN apt-get update && \            
    apt-get install -y certbot python-certbot-nginx && \   # 安装certbot
    apt-get clean
ADD nginx.conf /etc/nginx/               # 将我们定义好的配置加入nginx容器中
ADD default.conf /etc/nginx/conf.d/
```

`nginx.conf`中的内容为
```nginx
# nginx.conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    tcp_nopush     on;
    keepalive_timeout  65;
    include /etc/nginx/conf.d/*.conf;
}
```

`default.conf`中的内容为
```nginx
# default.conf
server {
    server_name  localhost;
    listen 80;
    listen [::]:80;           # ipv6对应

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
```

`docker-compose.yml`中的内容为
```yml
services:
  proxy:
    build: ./nginx
    tty: true
    container_name: proxy
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    ports:     # 映射80和443端口
      - "80:80"
      - "443:443"
    volumes:      # 将本地的文件映射到容器中， 注意左边为相对路径
      - './nginx/logs:/var/log/nginx' 
      - './nginx/letsencrypt:/etc/letsencrypt'
      - './html:/usr/share/nginx/html'
```

最后在`example`文件夹下运行`sudo docker-compose up -d`，nginx容器就搭起来了。浏览器访问`http://<hostname>`就能够看到`index.html`里面的内容了。也可以通过`docker ps`确认现在在运行的容器。
```sh
$ sudo docker ps
CONTAINER ID   IMAGE        COMMAND                  CREATED        STATUS         PORTS                                                                      NAMES
9bbc29f18441   cool_proxy   "/docker-entrypoint.…"   43 hours ago   Up 8 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   proxy
```

##### 3. 开启 HTTPS

通过上面的流程，我们的网站已经可以 HTTP 访问了。如果想要开启 HTTPS 访问，就必须要有私钥和证书。[Let's Encrypt](https://letsencrypt.org/zh-cn/) 提供免费的证书，但是有效期为 3 个月，需要定期更新。这个并不是太大的问题，只要通过 cron 加个定时任务就行。

运行`sudo docker exec -it proxy bash`进入到我们刚刚建立的 nginx 容器中，运行`certbot`获得证书（中间会让你填写域名相关信息）。证书会保存在`/etc/letsencrypt/live/<hostname>`文件夹下。

接下来修改`default.conf`，加入 HTTPS 相关配置（注意替换成自己的域名）。
```nginx
server {
    server_name  localhost;
    listen 80;
    listen [::]:80;
    return 301 https://$host$request_uri; 
}

server {
    server_name  localhost;
    listen       443 ssl http2;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384";
    ssl_prefer_server_ciphers   on;
    ssl_certificate     /etc/letsencrypt/live/<hostname>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<hostname>/privkey.pem;

    # Improve HTTPS performance with session resumption
  	ssl_session_cache shared:SSL:10m;
  	ssl_session_timeout 10m;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
```

然后`example`文件夹下运行`docker-compose restart`将刚刚的更改重载，浏览器访问`https://<hostname>`，如果能打开就代表 HTTPS 已经开启。

##### 4. 自动更新证书

因为 let's encrypt 证书是 3 个月一过期，所以我们需要在 cron 里面加一个定时任务，比如每个月更新一次证书。
运行`sudo crontab -e`打开定时任务配置文件，在最后一行添加以下代码。
```
@monthly { date; docker exec proxy certbot renew; } >> ~/example/nginx/logs/certbot.log 2>&1
```

为了确定命令是否可以被正确运行，在命令行运行
```sh
$ { date; sudo docker exec proxy certbot renew; } >> ~/example/nginx/logs/certbot.log 2>&1
```
然后运行`cat ~/example/nginx/logs/certbot.log`查看 certbot 证书更新日志，有以下 log 代表可以运行。
```
Thu 16 Sep 2021 05:48:59 PM UTC
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/<hostname>.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Cert not yet due for renewal

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

The following certs are not due for renewal yet:
  /etc/letsencrypt/live/<hostname>/fullchain.pem expires on 2021-12-15 (skipped)
No renewals were attempted.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

接下来可以将目前的文件备份到 github 上，以后只要安装完 Docker 环境，即使换服务器，也可以一键搞定。

<br />
<br />

<p style="text-align: center;">如果本文对您有帮助，欢迎打赏。</p>
<img src="/images/qr-wechat.png" alt="赞赏码" width="300"/>