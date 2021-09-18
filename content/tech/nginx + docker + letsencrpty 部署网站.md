---
title: "nginx + docker + letsencrpty 部署网站"
date: 2021-09-17T20:00:39+09:00
slug: "deploy-website-with-nginx-docker-letsencrypt"
dropCap: false
draft: true
---

前端后端都稍微写过，也有一个基于hugo的静态博客，不过在部署网站方面都是靠别人或者工具自动化，真要自己从头部署一个网站还真是两眼一抹黑，抓瞎。所以记录一下部署一个https网站的折腾过程，给有需要的人。

##### 目标

- 在ubuntu系统上启动nginx容器
- 通过Let's encrypt得到HTTPS功能

##### 需要准备的东西

- 域名
- 服务器

##### 1. 安装Docker

根据[文档](https://docs.docker.com/engine/install/ubuntu/#install-using-the-convenience-script)，得到安装docker的脚本，运行安装。

```bash
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
Executing docker install script
```