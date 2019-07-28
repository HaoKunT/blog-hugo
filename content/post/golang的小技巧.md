+++
title = "Golang的小技巧"  # 文章标题
date = 2019-07-26T03:12:44+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["golang"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
toc = true
categories = ["Golang"]
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## go命令行参数

### 编译
`go build` 是最简单的编译命令，对于一个包，这个命令只会做检查，即检查包是否有编译错误，对于main则会生成一个可执行文件.

参数 | 参数含义
--- | ---
`-o` | 指定输出的可执行文件名称
`-x` | 输出详细信息，包含编译时的每一步骤

#### cgo
cgo是go语言对于和C语言混合编程所给出的官方解决方案，用C包解决，对于使用了cgo的包来说，其编译可以有额外的参数

参数 | 参数含义
--- | ---
`--ldflags` | 给`cgo`命令的参数
`--ldflags -extldflags` | 给`gcc`在链接时的额外参数


### 运行
`go run` 是运行命令，其等于 `go build xxx.go && ./xxx`，会在临时文件夹下编译相关文件，并运行。

> 注意:
> 
> 运行时是在临时文件夹下，因此其相对路径可能产生问题

### 安装
`go install` 是安装命令，安装命令会将编译后的结果安装，如果是可执行文件则将编译后的可执行文件放在`GOPATH/bin`下，如果是包则会放在`GOPATH/pkg`下

### 测试
`go test` 是测试命令，`golang`的测试分为三种

- 单元测试
- 基准（性能）测试
- Example

有关`golang`测试的详细信息请[参考][golangtest]




[golangtest]: https://blog.csdn.net/u011957758/article/details/81267972

