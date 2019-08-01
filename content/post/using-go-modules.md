+++
title = "Go Modules的使用"  # 文章标题
date = 2019-08-01T22:44:38+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["golang"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["golang"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

自从`go1.11`版本后，golang官方给出了一个管理第三方依赖的方式——`Go Modules`，这是官方提倡的新的包管理机制。

## `go modules`的初始化
如果你使用的是golang-1.11的版本，你需要先将环境变量`GO111MODULE`设置为`on`，网上也有说法当当前目录里面有`go.mod`文件的时候会自动开启（默认是`auto`），由于笔者已经将版本升至1.12，没有尝试。

和传统的`GOPATH`不一样，不要繁琐的`src`,`bin`等子目录，任何目录均可成为`modules`（前提是不在`GOPATH`里面），只要其中包含`go.mod`文件。

那么如何包含`go.mod`文件呢，你可以选择新建这个文件，然后在首行写上`module {name}`即可，更好的做法是执行下面的命令
``` shell
go mod init {name}
```

如果你的项目是从`github`上`clone`下来的，那么可以选择省略后面的`name`，默认会使用`git`管理的名字作为`module name`

初始化完成后会在目录下生成一个`go.mod`文件

## 包管理
那么我们如何进行包管理呢，在有`go.mod`的目录下，当我们执行`go build`,`go test`,`go list`等命令的时候，go会自动帮我们更新依赖，并将依赖关系写入其中

![show](https://raw.githubusercontent.com/HaoKunT/github-content/master/content/blog/post/using-go-modules/go-modules-show.PNG)

我们会发现目录下多了一个`go.sum`文件，看看里面是啥

![sum](https://raw.githubusercontent.com/HaoKunT/github-content/master/content/blog/post/using-go-modules/go-modules-sum.PNG)

是的，这个是我们引用的`package`和其自身引用的版本记录。并且会记录下其所需的版本以及时间戳和hash值。

如果我们不做任何修改，默认会使用最新的包版本，如果包打过tag，那么就会使用最新的那个tag对应的版本。

如果你需要特定版本的包，可以在`go.mod`里面修改`require`来指定版本。你也可以通过命令修改
``` shell
go mod edit -require="github.com/dgraph-io/badger@v1.5.4"
```
然后运行`go mod tidy`，就会自动切换版本

下面我们来编译代码
``` shell
go build -mod=readonly
```
这里的`readonly`是指任何会导致依赖关系变动的情况都会编译失败

## 缓存
在启用`Go Modules`之后，原先的`GOPATH`就不再适用了。包括你使用`go get`命令的时候，包不再被下载到`GOPATH/src`目录下，而是被下载到缓存下（包括使用`build`命令），这里的缓存是在`GOPATH/pkg/mod`目录下（不知道这个目录能不能进行更换，否则仍然对GOPATH有一定的依赖，当然这个依赖不是很明显），这个目录和原先的`src`目录结构类似，不同的地方在于，其每个`module`有不同的版本，通过`name@version`的方式对不同的包进行命名。

## 代理
原先在`GOPATH`时代，当需要下载墙外的包的时候（尤其是官方包`golang.org/x/...`），我们需要先从github上下载，然后调整包的位置，相当麻烦（当然也可以设置git的代理，但是总觉得为了golang下载包就得设置git的代理，有点奇怪）。当有了`Go Modules`，我们可以通过环境变量`GOPROXY`设置代理，在下载包的时候走代理即可。
这里有一个开源的代理[`https://goproxy.io`](https://goproxy.io)

## 依赖本地包
到目前为止，还有一个问题，即如果我们所依赖的包并没有发布在`github`这种网站上，我们也不想发布上去，或者其本身就是一个子模块，那么应该怎么办。我搜索了很长时间的解决办法，最终找到[这里]([https://](https://stackoverflow.com/questions/52123627/how-do-i-resolve-cannot-find-module-for-path-x-importing-a-local-go-module))

我们需要手动添加一下`go.mod`
``` go
require X v0.0.0
replace X v0.0.0 => "{local path to the X module}"
```
这里的`replace`相当于对包进行重命名，在你不使用代理的时候，对于墙外的包可以用这种方式进行处理。

> 注意：
> 这里的`local path to the X module`一定要用绝对路径（我尝试过相对路径，不行）
>
> 另外，在代码里面也不再需要`import ./X`，只需要`import X`，因为你是从modules里面找包，而不是从本地找包


