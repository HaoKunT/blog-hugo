+++
title = "Seafile"  # 文章标题
date = 2019-07-26T06:07:06+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["seafile", "存储", "云盘"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
toc = true
categories = ["存储"]
preserveTaxonomyNames = true
disablePathToLower = true
+++
## seafile
[seafile][seafile]是一个同步云盘，可以在自己的服务器上自行搭建同步云盘，基于django开发，python版本为python2.7，目前有社区版可用。

## 部署
从[seafile官网][seafile download]上下载linux安装包即可，然后按照[官网的服务器手册][manual]完成一键部署

这里需要注意权限问题，建议新开一个seafile用户来运行seafile

## 配置开机自启和服务
按照[官网的服务器手册][manual]来即可

## 配置ssl
按照官网即可，这里由于没有443端口，因此无法混用

## 集成onlyoffice
官网有相关内容，总体来说按照官网内容即可，由于笔者没有多余的域名，因此采用子文件夹的方式运行，注意将map中的代码进行修改
```
# Required for only office document server
map $http_x_forwarded_proto $the_scheme {
        default $http_x_forwarded_proto;
        "" $scheme;
    }

map $http_x_forwarded_host $the_host {
        default $http_x_forwarded_host;
        "" $host:{port};# 因为笔者无法使用443端口（大家都懂的），因此这里在映射的时候需要增加端口，否则office在重定向的时候有误（丢失端口）
    }

map $http_upgrade $proxy_connection {
        default upgrade;
        "" close;
    }
```
自此就完成了部署，若以后持续为seafile增加功能，本文将持续更新


[seafile]: https://www.seafile.com/home/ "seafile官网"
[seafile download]: https://www.seafile.com/download/ "seafile官网下载页面"
[manual]: https://manual-cn.seafile.com/ "seafile官方文档"
