+++
title = "Github图片加载不出来"  # 文章标题
date = 2020-01-07T14:35:49+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["chrome","dns"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["dns"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## 背景
最近访问github的时候图片居然都加载不出来了，包括头像等等，后来去网上查了下，觉得比较靠谱的说法就是DNS污染了，不过也不一定，可能是DNS缓存，包括阿里家的，谷歌家的，114的，对于github图片的dns都有问题，所以无奈之下只能改Host文件了

## 步骤
首先查到加载不出来的图片的url，然后提取出二级域名，域名基本上就是这样的没跑了

`avatars0.githubusercontent.com`

`avatars1.githubusercontent.com`

然后查[IPAddress](https://www.ipaddress.com/)，看一下IP，然后回到Host文件里面添加一下

`C:\Windows\System32\drivers\etc\hosts`

```
199.232.28.133 user-images.githubusercontent.com
199.232.28.133 raw.githubusercontent.com
199.232.28.133 camo.githubusercontent.com
199.232.28.133 avatars0.githubusercontent.com
199.232.28.133 avatars1.githubusercontent.com
199.232.28.133 avatars2.githubusercontent.com
199.232.28.133 avatars3.githubusercontent.com
199.232.28.133 avatars4.githubusercontent.com
199.232.28.133 avatars5.githubusercontent.com
199.232.28.133 avatars6.githubusercontent.com
199.232.28.133 avatars7.githubusercontent.com
199.232.28.133 avatars8.githubusercontent.com
```

清一下DNS缓存`ipconfig /flushdns`，刷新，OK

## 另一种可能的方案
通过DNS污染联想到了另外一种可能的方案，利用现在的DNS over https(DoH)，杜绝在浏览的时候发生DNS污染的问题，不过这个方案我在firefox上貌似成功了，在chrome上貌似还是不成功，仍然显示不出来。

我用nslookup命令，通过不同的DNS解析服务查了github的ip地址，另外还有IPAddress上找到的地址
```
1.1.1.1 : 192.30.255.112
8.8.8.8 : 13.229.188.59
223.5.5.5 : 13.229.188.59
114.114.114.114 : 52.74.223.119
IPAddress : 140.82.114.4
```
得，就四个数据源给了我三个不同的IP，难道是cdn？算了头疼，不去想了。