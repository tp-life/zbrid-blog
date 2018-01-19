---
title: "支付宝APP支付问题记录"
date: 2018-01-19T15:16:27+08:00
lastmod: 2018-01-19T15:16:27+08:00
draft: true
keywords: []
description: ""
tags: []
categories: []
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
---


> 这两天开发支付宝APP支付的时候遇到几个坑，主要体现：


#### 1、支付宝新版与旧版的区别

    1、支付宝SDK分为旧版与新版，新版与旧版的主要时间分段为2016年8月10日，在这之前的都是旧版的SDK，使用的时候可使用旧版SDK。
    2、新版APP支付与旧版支付签名不一致，这一点尤其的重要，新版主要使用appid，而旧版主要使用商户号。
    3、新版支付与旧版的API接口不一致，所需要的参数也是不一致的。
<!--more-->
#### 2、新版当中出现`AL40247`错误。
    对于这个错误出现的次数是最多的，很多的时候都是卡在这里，出现这个问题一般是由于appid或者密钥不正确的原因。
说到这个密钥自己也是被弄迷糊了好长的一段时间，到处都有冲斥着密钥，特别现在老版的支付还残留着的时候。
`注意`：这个主要有`rsa`与`rsa2`的区别，`rsa`的长度一般`1024`，`rsa2`是`2048`的长度，最开始我们使用rsa的密钥进行签名怎样都不正确，但是换成rsa2就成功了，不知道为什么阿里会出现这个问题。不过这个问题各位要注意一下了。

#### 3、其他相关问题。
其他问题要么是因为自己应用支付已经失效了或者因为app支付还未签约造成的。重新签约便可解决。不过阿里现在这个签约的资料真的是太麻烦了，我进行到一半的是时候就放弃了，阿里要不要这么烦躁啊，简便一点不好吗？😂😂😂😂😂😂😂😂😂



