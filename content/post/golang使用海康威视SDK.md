+++
title = "Golang使用海康威视SDK"  # 文章标题
date = 2019-09-13T15:22:39+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["golang","SWIG","混合编程"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["Golang","SWIG","技术"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
[graphviz]
  enable = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

接上一篇文章[SWIG编译海康威视SDK-使用golang](https://hkvision.cn/2019/07/26/swig%E7%BC%96%E8%AF%91%E6%B5%B7%E5%BA%B7%E5%A8%81%E8%A7%86sdk-%E4%BD%BF%E7%94%A8golang/)，这篇文章讲述的是如何使用编译好的文件，这涉及到SWIG和golang结合的问题

## 引入
编译好了之后`.a`文件会出现在`GOPATH`下，直接在import里面引入`hikvision`就可以了，build的时候会自动找到对应的`.a`文件

## 使用
下面从两个方面来介绍
- golang中已有的类型对应关系
- C里面`typedef`的值的对应关系，例如海康威视有`BOOL`,`DWORD`等

## golang中已有的类型对应关系
这里说的已有的类型是指在C语言中已经有了，比如说int，char*等，具体的对应关系表网上有很多，SWIG在这里的策略和`cgo`是一样的，这里给一个[链接](https://books.studygolang.com/advanced-go-programming-book/ch2-cgo/ch2-03-cgo-types.html)。这里就不多说了，在函数调用的时候可以直接把对应的go值传进去而不需要转换

## C里面`typedef`的值的对应关系
这里主要来说这个问题

由于海康威视的SDK大量使用了`typedef`（应该是为了解决跨平台编译的问题），导致很多函数的返回值，参数值都不是正常的C类型，再加上大量的结构体，像我这样的菜鸟一开始根本无从下手，后来仔细看了SWIG生成的`wrapper`文件和搜索引擎了一番，才弄明白SWIG的逻辑。

SWIG的实现其实就是调用了cgo，只是他给你封装了一层，你如果弄明白了cgo的逻辑，那SWIG的逻辑也就不难理解了。

### Swigcptr uintptr unsafe.Pointer
这是核心，这三个东西构成了所有的C和go直接的类型交互。首先介绍一下C和go之间进行参数传递的流程

```viz-dot
digraph G {

  node [width=3]

  a0 [label="go值"]
  a1 [label="go野指针"]
  a2 [label="内存地址的uint值"]
  a3 [label="C指针"]

	subgraph cluster_go2c {
		style=filled;
		a0 -> a1 [label=" unsafe.Pointer"];
    a1 -> a2 [label=" uintptr"]
    a2 -> a3 [label=" Swigcptrxxx"]
		label = "Go to C";
	}

  b0 [label="C指针"]
  b1 [label="内存地址的uint值"]
  b2 [label="go野指针"]
  b3 [label="go值"]

	subgraph cluster_c2go {
    style=filled;
		b0 -> b1 [label=" Swigcptr"];
    b1 -> b2 [label=" unsafe.Pointer"]
    b2 -> b3 [label=" 指针转换"]
		label = "C to Go";
	}
}
```
这里的`Swigcpterxxx`后面的`xxx`对应的类型名称，Swig会自动帮我们生成这个函数。

整个的核心在于`unsafe.Pointer`函数，这个函数接收一个指针类型的变量（也可以接收`uintptr`类型的值），输出的是一个指针，但是对于go来说，这个指针已经变成了类似于野指针的存在，也就是说go不再承认这个指针是go体系里面的内容，gc也不会回收这块内存，需要我们自行管理。这个指针和C里面的void指针类似，代表这个指针可以指向任何一块内存。因此可以将这个指针对应的uint值（内存地址都可以用uint类型的变量来存储，这是一个特殊的类型，在GO和C中是`uintptr`，本质和uint是一样的）传递到C中。C的值传递到GO也是一样的过程，都是通过`void*`类型和`unsafe.Pointer`类型的值可以互传来实现的。

### 多重指针
在海康威视的SDK中有一种类型本身就是指针，例如`LPVOID`，那么使用SWIG编译后，对于GO程序而言，同样要用多重指针的思路来解决。

多重指针和单指针是一样的道理，在将GO值传递到C之前，我们首先要将这个值转化为多重指针，然后再进行上述步骤，这样海康威视SDK拿到的值才是正确的，不然就会报错。我也是实验了很多次才发现这个问题的，其实应该一开始就明白这个问题，还是基础不够扎实，没能一眼看明白。

下面是一个示例
``` go
lpOutBuffer_t := hk.NewNET_DVR_DIGITAL_CHANNEL_STATE()
defer hk.DeleteNET_DVR_DIGITAL_CHANNEL_STATE(lpOutBuffer_t)
dwCommand_t := hk.NET_DVR_GET_DIGITAL_CHANNEL_STATE

// 注意这里的转换与下面的转化的不同
lpOutBuffer_p := unsafe.Pointer(lpOutBuffer_t.Swigcptr())
lpOutBuffer := hk.SwigcptrLPVOID(uintptr(unsafe.Pointer(&lpOutBuffer_p)))

// 注意这里只用了一次`unsafe.Pointer`函数
dwCommand := hk.SwigcptrDWORD(uintptr(unsafe.Pointer(&dwCommand_t)))

```
这是截取的代码的一小部分，这里我是需要调用海康威视的`NET_DVR_GetDVRConfig`函数，这个函数有一个参数即为`LPVOID`类型，就是`lpOutBuffer`这个变量，实际上这个变量的值类型是`NET_DVR_DIGITAL_CHANNEL_STATE`，大家可以看到，这里首先在在创建一个`NET_DVR_DIGITAL_CHANNEL_STATE`类型后，先用`Swigcptr`函数拿到变量对应的地址，然后用两次`unsafe.Pointer`函数，然后才转成`LPVOID`，其余的变量我们只转换了一次。

## 函数调用
函数调用非常简单，当我们用`unsafe.Pointer`组织好各个参数后，直接调用即可，注意返回值也需要进行转换，这里就是进行逆转换了，对于海康威视SDK，大家可以写一个公共函数来进行转换，这样方便调用。
