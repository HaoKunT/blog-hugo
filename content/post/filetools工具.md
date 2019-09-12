+++
title = "Filetools工具"  # 文章标题
date = 2019-09-12T19:37:13+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["工具","golang"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["工具"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## filetools工具
本人用`golang`写的一个小工具，用于对文件进行操作，项目地址在这：[filetools][filetools地址]

## 安装
``` shell
go get github.com/haokunt/filetools
```
暂时无法直接提供编译好的二进制文件，等我啥时候有时间了研究下CI工具再说。

## 使用
最初写这个工具的设想是将很多命令整合到一起，像`cp`,`mv`,`rm`等，所以所有功能均使用子命令的方式进行提供，可以用`-h`来查看使用帮助。对于子命令也可以在后面加上`-h`查看子命令使用帮助。

## 中文帮助
项目已经可以实现国际化，设置正确环境变量即可看到中文帮助。

设置LANG环境变量即可
``` shell
# Linux
export LANG=zh_CN
# Windows
set LANG=zh_CN
```

## 已实现的功能
### comapre
`compare`用于比较两个目录的不同，使用golang的`os.Stat`函数获取文件信息，比较大小和文件名两方面，加上`-c`参数则会打开每个文件进行详细比较。比较信息将打印在控制台，包括new file, modified file（文件名一样但是大小不一样或者）, deleted file。

使用场景是在进行目录复制的时候，如果中途断了，或者一些其他情况，导致我们无法判断目录是否复制成功，可以进行比较。

### copy
`copy`将一个文件或者目录拷贝到其他地方，这个命令最激动人心的地方是有进度条，加上`-pb`即可，有进度条的情况下会先统计源目录，因此速度会慢一些，我自己使用的时候是40w个文件统计需要花一段时间，不过不太长，可以接受。

使用场景是在对大文件，大目录进行拷贝的时候可以加上进度条，灰常舒爽

### info
`info`命令会输出文件或目录的一些信息，这些信息包括文件名，文件类型（真实类型），文件大小（真实目录大小），权限等

使用场景是探测真实文件类型，在Linux下统计真实目录大小

### rename
`rename`命令会重命名文件，加上`-k`参数可以保持原有文件（类似copy，但不支持目录）

使用场景略

### list
`list`命令会列出目录下的所有文件，这个命令可以排序，可以限制输出到控制台的数量

使用场景是对某目录下的大文件进行查找，快速定位哪些文件占用了较多的磁盘空间

## 未来的打算
未来将提供更多的文件相关的功能，要是有什么好的建议可以提issues。





[filetools地址]: https://github.com/haokunt/fieltools