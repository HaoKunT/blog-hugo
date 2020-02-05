+++
title = "函数计算搭建DNS服务器"  # 文章标题
date = 2020-02-05T13:27:27+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["DNS","DoH"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["技术"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## 缘起
见`DNS解析过程`这篇文章的缘起

## DoH 服务概念
DoH（DNS over HTTPS）,指的是将DNS解析的内容使用HTTPS协议进行传输，而不是UDP协议传输，其优势在于隐私和防篡改，可以有效的防止DNS劫持，关于DNS的劫持，可以看其他文章。

## DoT 服务器概念
DoT(DNS over TLS)，指的是将DNS解析的内容使用TLS协议进行传输，优势和DoH一样

## 安全性保证
DoH服务和DoT服务对于防篡改和防偷窥的安全保证是通过现有的CA证书体系保证的，其中DoH相较于DoT在TCP协议和DNS协议中间增加了一层HTTPS协议，性能差一点，但是由于HTPP的库非常广泛，协议大家也非常熟悉，因此开发相应的客户端和服务器端会更加方便，更具推广性

## 函数计算
函数计算是阿里云针对无服务所开通的产品，选择函数计算的原因是阿里云函数计算有较多的免费额度（每个月100w次，40wcu-s），因此对于个人使用而言，除了需要付流量费之外，基本上免费，其流量费大约`0.8元/GB`，一个月大约会使用`50M`的流量（每天12小时正常使用电脑）

## 配置过程
已有项目放在[`GitHub`](https://github.com/HaoKunT/aliyunfc-doh-server)上，这里详细介绍配置流程

项目架构：
1. Web服务器：caddy
2. DNS服务器：CoreDNS
3. forward：谷歌DNS（8.8.8.8）
   
下载[Caddy](https://caddyserver.com/)，我使用的`version 1.x`，创建Caddyfile，里面写入这样的内容
``` caddy
:9000 {
    gzip
    rewrite {
        r (.*?)resolver(.*)
        to {2}
    }
    proxy / http://localhost:8053 {
        transparent
    }
    log stdout
}
```
下载[CoreDNS](https://coredns.io/)，创建Corefile，里面写入这样的内容
``` caddy
https://.:8053 {
    cache
    log
    errors
    forward . tls://8.8.8.8 tls://8.8.4.4 {
        tls_servername dns.google
        health_check 30s
    }
    whoami
}
```
创建`bootstrap`文件
``` shell
#!/bin/bash

./coredns &
./caddy &
wait
```
去[阿里云](https://aliyun.com)上开通函数计算服务

创建`template.yml`，里面写入这样的内容
``` yml
ROSTemplateFormatVersion: '2015-09-01'
Transform: 'Aliyun::Serverless-2018-04-03'
Resources:
  Doh-Service:
    Type: 'Aliyun::Serverless::Service'
    Properties:
      Description: 'doh-server'
      InternetAccess: true
      # LogConfig: 
      #   Project: doh-server-log
      #   Logstore: doh-server-log-store
    resolver:
      Type: 'Aliyun::Serverless::Function'
      Properties:
        Handler: index.handler
        # Initializer: index.initializer
        CodeUri: ./code.zip
        Description: 'resolver function'
        Runtime: custom
      Events:
          httpTrigger:
            Type: HTTP
            Properties:
              AuthType: ANONYMOUS
              Methods: ['GET', 'POST', 'PUT', 'DELETE', 'HEAD']
```
安装fun工具
``` shell
npm install -g @alicloud/fun
```
然后执行
``` shell
fun config # 先配置一下fun工具 
fun deploy
```
OK，整个配置工程完成

## 为什么我们forward使用`8.8.8.8`
首先我一开始使用的是`1.1.1.1`，那我改到了`8.8.8.8`的原因是，我发现使用cloudflare的DNS解析和使用国内的DNS解析获得的IP不一样，其中使用cloudflare的IP使用不稳定，我估计是做了线路优化，然而我使用国外的DNS的话，如果网站做了线路优化，则明显会出问题，速度会非常慢，因此我使用了8.8.8.8

关于劫持的问题，首先我们使用了DoT，基本上劫持不会产生，并且我使用了tracert命令追踪了针对8.8.8.8的路线，应该是访问的美国谷歌的IP，虽然不知道为什么时延真的很低（40ms）

## 客户端
本人使用的客户端是[`DNSCrypt-proxy`](https://github.com/DNSCrypt/dnscrypt-proxy)，具体怎么使用看文档即可，主要是改一下源就行


