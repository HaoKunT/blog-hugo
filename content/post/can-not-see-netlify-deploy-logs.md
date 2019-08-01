+++
title = "看不了netlify的部署日志"  # 文章标题
date = 2019-08-02T04:12:46+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["netlify"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["博客"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

这篇文章主要是解决我之前的一篇[文章](https://hkvision.cn/2019/07/27/%E4%BD%BF%E7%94%A8hugo-netlify%E9%83%A8%E7%BD%B2%E4%B8%AA%E4%BA%BA%E4%B8%BB%E9%A1%B5/)遗留的问题
## 问题
之前我在用`netlify`部署博客的时候，发现无法查看部署日志，主要表现为日志那里会显示
```
[ERROR] Deploy logs are currently unavailable. We are working on resolving the issue.
```
这个问题我找解决方案找了好久
## 解决方案
其实是因为在请求日志的时候有一个请求被墙了（是的，是其中一个请求被墙了）。因此你能看到整个网页，但是就是看不到日志。解决方案就是，**代理！！！**
