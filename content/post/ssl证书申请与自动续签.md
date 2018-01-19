---
title: "Ssl证书申请与自动续签"
date: 2018-01-19T13:59:21+08:00
lastmod: 2018-01-19T13:59:21+08:00
draft: true
keywords: ["SSL","certbot"]
description: "SSL证书自动续签"
tags: ["SSL","certbot"]
categories: []
author: "zbrid"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
---


> 利用certbot 实现SSL证书自动续签功能

最近使用hugo在阿里云上搭建了一个个人博客，在使用博客的时候想到干脆把https也上上去吧，然后各种奔波，只为了一款比较好用的免费的SSL证书。当然啦，免费的当然要推崇`Let's encrypt`了。好了，话不所说了。上干货~
<!--more-->
## Certbot
- certbot 是一个申请证书并帮助我们实现自动续签的软件， 其主页地址[ 官网 ](https://certbot.eff.org/),官网当中有很多关于安装与使用的内容，为了简便与英文不好阅读的关系，简单记录下使用方法。

#### 1、下载并安装certbot
```perl
    wget https://dl.eff.org/certbot-auto
    chmod a+x certbot-auto
```
#### 2、停止nginx
    因为certbot 在创建证书期间是不能开启80端口的，因此我们这里需要关闭nginx

#### 3、生成证书
```perl
./certbot-auto certonly --standalone --email 邮箱 -d 域名1 -d 域名2
```
这个时候certbot证书会提示你是否同意相关协议，此时输入 `A` 表示argee 同意，接下来会询问你是否生成证书，输入`Y` 同意即可。在这里不过要注意些事情：
* 域名必须是备案成功了的。
* 域名解析须是指向某一台服务器的，解析类型为`A`
* 证书生成的地址为`/etc/letsencrypt/live/`

我几次生成证书都提示我域名404错误，最后在阿里云上才发现是因为没有解析在服务器上。

#### 4、nginx配置
```perl
    # TLS 基本设置
ssl_certificate /etc/letsencrypt/live/域名/fullchain.pem;#证书位置
ssl_certificate_key /etc/letsencrypt/live/域名/privkey.pem;# 证书位置

```
至此，ssl证书便算是可以了，最后一步便是加入计划任务里，以免其自动续期

#### 5、自动续期
使用`crontab -e`编辑计划任务，添加`0 0,12 * * * python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew 
`即可。

最后，如果只是想安装web服务器插件的类型的certbot的话，在certbot的主页选择你所对应的web服务器与服务器系统版本根据其步骤进行便可。

