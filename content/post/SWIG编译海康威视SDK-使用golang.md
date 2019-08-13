+++
title = "SWIG编译海康威视SDK 使用golang"  # 文章标题
date = 2019-07-26T06:13:00+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["swig", "golang", "混合编程"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
toc = true
categories = ["Golang","SWIG"]
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## 吐槽的话
先允许我吐槽一下海康威视的SDK有多难用SWIG编译

## 编译过程
### 准备工作
先声明我编译的是linux版本的，环境是ubuntu 18.04，还没有开始做开发，但是编译的过程没有报错

> Tips:

> windows的也尝试了(win10)，没有问题

首先你需要去海康威视的官网下载[SDK][SDK]，然后安装好Go和SWIG，设定好`GOPATH`，然后将SDK里面的`HCNetSDK.h`文件放至`$GOPATH/src/hkvision`下

>不知道GOPATH是什么的童鞋请自行百度

### 编写interface文件
interface文件是SWIG的接口文件，用来定义`C`语言环境中的变量，函数等在目标语言环境中的表达，具体可参考我另一篇[文章][interface编写]。这里我们简单粗暴的直接包含头文件好了

``` C
#include <HCNetSDK.h>
```

### 改写头文件
海康威视提供的头文件是不能被SWIG所识别的，这是由于SWIG对于一些C的语法不支持，导致其会疯狂的报语法错误，再加上语法错误的原因语焉不详，SWIG能支持啥不能支持啥完全懵逼，只能靠猜，目前已知的一些不支持的语法

- 注释里面套注释，例如这样的`//这里是注释1 /*这里是注释2*/`
- `#define xxx`时，若后面函数被`xxx`修饰，当`xxx`无对应的值而仅仅是被定义的时候，SWIG认为需要在`xxx`后面加分号
- 联合嵌套在C++中是不支持的

因此我们需要先对头文件进行改写，另外请注意SWIG之后需要用cgo来进行编译，而cgo是不支持C++的，所以我们需要将C++的语法去掉，尤其是`extern "C"`，用宏定义即可（C++自带`__cplusplus`宏）

#### 使用正则表达式去掉所有注释
- `//.*$` 
- `/\*[\w\s\d\[\];\.\\]*\*/` (这里中括号中间要加一个代表中文的东西)

#### 使用正则表达式去掉函数前面的`NET_DVR_API`和`__std`

>这里的正则想不起来了反正到时候自己看这写就好了

#### 去掉CALLBACK
这里正则也想不起来了

>这里是因为CALLBACK在SWIG的时候过不了

#### 使用正则为无tag的结构体加上tag
这里正则也想不起来了

>多说一句，这里是因为后来gcc报了一个`warning`，没有这个tag的结构提貌似在外部就不可见了，不知道什么情况，反正我给这玩意加上了，要是有大神知道什么情况的话可以选择性忽略

#### 删除无实现的函数
有一些函数在so里面并没有实现，我在官方的文档上也没看到这个函数，但是头文件里面已经有相关的函数定义，而没有实现的话在链接的时候gcc会因为没有找到函数实现报错，因此这一点比较坑，你需要先编译一遍，然后对着报错手动把这些函数删掉，通常包括旧有方法的新版本，例如`xxxx_V50`等等，和具体的SDK版本有关

> 这里有已经修改好的一份[修改好的头文件]可以用，版本具体记不清了，是`19年3月`的版本，64位

### 开始编译
#### 生成xxx_wrap.c文件和xxx.go文件
首先生成wrap文件和go文件，执行的指令是`swig -go -cgo -intgosize 64 hkvision.i`，这一步大概需要半小时，因具体的机器而异，我用的是E5 2690 v3

#### 编译
编译之前，先打开go文件，在cgo的前面的注释上加上`#cgo LDFLAGS: -L./ -lxxxx`，就是给gcc参数，gcc链接的时候去找动态库

然后开始编译，`go install -x -v`，这一步需要的时间很长，我机器上大概4个半小时，你可以晚上挂着跑第二天起来看结果，或者`nohup`挂在后台，你去做其他的事情。

一切顺利的话在`$GOPATH/pkg/linux_amd64/`下面你会看到`hikvision.a`文件（超大，有200M）

### 报错了怎么办
沉着冷静别惊慌，先去看输出，输出里面有很多有用信息，这就是为什么编译的时候带上`-x`参数，你可以看到详细的编译过程，报错通常有两种

- 语法错误，配置错误等，这种错误一般马上就会报错，然后程序退出，这都是低级错误，你很快就能改好
- 编译错误，这种错误就很麻烦了，一般都是你等了几个小时，结果给你报错，不过别慌，无非就是你头文件没改好，或者你库的位置不对，链接不到动态库

### 其他语言
其他语言应该都差不多，具体的差别就是看你使用的编译器了，`msvc` `mingw` 原生`gcc`等

>PS：`msvc`和`gcc`差别蛮大的，另外cgo没有使用g++的原因是会报一个错，在`pthread`这个库里面，这个库是cgo原生需要引入的一个库，而这个库貌似不支持g++

[SDK]: https://www.hikvision.com/cn/download_61.html
[interface编写]: https://hkvision.cn/2019/07/26/swig-%E4%BB%A5python%E4%B8%BA%E4%BE%8B/
[修改好的头文件]: https://raw.githubusercontent.com/HaoKunT/github-content/master/content/blog/post/SWIG编译海康威视SDK-使用golang/HCNetSDK.h