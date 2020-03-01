+++
title = "阿里云API网关与函数计算的基础理解"  # 文章标题
date = 2020-03-01T19:43:31+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["apigateway","函数计算"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["阿里云","技术"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true

+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## 背景
想法是在函数计算上搭建一个web命令行（加速GitHub，国内太慢了），并且通过websocket实时将执行过程发送到前端。在本地其实已经成功了，但是实际上放到函数计算的时候出现了一个大问题，就是函数计算的Http触发器不支持异步调用，更是不支持websocket协议，查了些资料之后发现和阿里云的API网关配合起来使用可以用上websocket。在查这些资料的过程中终于弄明白了函数计算的`custom runtime`是怎么回事，这里记录一下。

## 函数计算
首先，我们需要明白，阿里云的函数计算（custom runtime）更像是给你一个极小极小的小型虚拟机，理论上来说，你可以拿这个小虚拟机做正常主机任何的操作（在阿里云给你的权限下）。这里给一下阿里云的文档的说法

![fc-custom-runtime](https://haokunt-pic.oss-cn-beijing.aliyuncs.com/20200301200347.png)

大家可以看到这里，custom runtime本质上是一个`HTTP Server`，也就是说，如果我们能在`HTTP Server`中写一些执行linux的命令的代码，也就意味着我们有了一个位于阿里云的小型虚拟机，最关键的是这个虚拟机我们还不需要花钱。

当然这世界上没有那么好的事情，阿里云也不是傻瓜，这样得到的虚拟机是有限制的，这个限制就是我们必须要在他的权限下来使用，例如你不可能在上面安装一个`Mysql`数据库，因为装数据库这种事情超过了限制，但是这也不意味着我们啥都不能做，只是我们会遇上这样的一些问题

1. 如果我们想要执行的命令在函数计算的虚拟机上没有怎么办？
2. 如果我们想要执行的命令超过了函数计算虚拟机所提供的权限怎么办？

## 解决函数计算限制的方法

​		其实我们所遇到的问题就像你有一台主机，但是你只有一个有限的权限，而且你不可能拿到sudo的权限，这两个限制很像。

​		个人的解决方法就是**使用golang语言开发命令行程序**

​		为什么这么说，首先，如果一个命令没有的话，完全可以将命令打包进函数计算所需要的zip包，或者将命令放到到阿里云的NAS上，函数计算挂载NAS，每一次启动的时候就可以有命令可用。第二，如果一个命令超过了权限限制，很多都是文件不能访问之类的，那么最好的一个方案就是，一个命令不依赖其他的库，文件。这个时候用golang就很方便了，静态编译，而且个人使用，完全可以把认证用的用户名和密码都硬编码到程序里面，或者更加复杂的，就把配置文件写到NAS里面去，或者把配置文件都放到OSS中，需要的时候内网请求即可。



## 函数计算的调用和触发

函数计算是需要事件触发的，触发的方式在阿里云的文档当中有比较详细的介绍，我建议大家先去将阿里云的[文档](https://help.aliyun.com/document_detail/52895.html)看一下，我这里记录一下我自己研究函数计算之后对于custom runtime的理解。（以下的函数计算特指custom runtime）

### Event的概念

在函数计算中，event是一个很重要的概念，每次函数计算的调用，对于函数都会传入一个叫做event的结构，这个结构是一个map（Python dict）,里面带有调用函数时所需要的一些参数。对于非custom runtime来说，event是请求时直接向后端传输的。但是对于custom runtime来说，event就是POST请求中的body（为什么是POST请求后面会有解释），这点一定要弄明白。

### 函数计算的调用

前面我们说到custom runtime本身是一个很小的虚拟机。那么这个虚拟机是怎么调用到我们的函数的呢？

首先这个调用分为两种，一种是普通调用，一种是http触发器调用。我们前面说到，custom runtime是一个http server，那么调用和http请求肯定脱不了干系了。实际上，一个调用就是一个http请求，但是这http请求是不一样的。

#### 普通调用

普通调用的http请求一定是POST请求，并且Path是`/invoke`，这两点是不变的，这是普通调用（实际上第一次的调用的时候会先调用初始化，这个的Path是`/initialize`），当然还有一些headers，这个就去查阿里云的文档就好了。

相信可能有人会问了，那这一个http服务器，我不可能只服务这一个Path吧，我一开始的时候也有这个疑惑，觉得不可思议，但是后来我研究之后，大概明白了，阿里云其实并不是想你把这个函数计算当作一个万金油般的存在，要是你的逻辑比较复杂，那你就多加几个函数呗。

那么如果我真的想让一个函数就挑起一个服务器的所有的事情，比如说像我这样的，每天根本没几个人会访问我的网站，我没必要弄很多函数，并且弄了很多的函数之后，函数之间很难进行通讯，那如果我一个逻辑是需要两个函数协同工作的时候，就比较难弄了（可能得上MQ之类的，得你又得去阿里云开通他的MQ服务）。那么如果你的网站比较传统，你可以使用HTTP触发器。

#### HTTP触发器

HTTP触发器其实是一个特殊的触发器，可以看到阿里云的文档，我一开始看文档的时候直奔HTTP触发器，结果后来搞的以为其他的触发器和HTTP触发器一样的。后来才知道HTTP触发器比较特殊。

首先，HTTP触发器只能在函数建立的时候创建，当一个函数已经创建之后就不能创建HTTP触发器了。其次，函数一旦设置了HTTP触发器之后就不能设置其他类型的触发器，并且每个函数只能创建一个HTTP触发器。当然还有一些其他的限制，这里不多说。

从上面的描述，我们可以推断出来一个问题，就是HTTP触发器的Path，HTTP请求类型，已经和普通调用完全不一样了。这也是HTTP触发器和其他的调用是互斥的原因。HTTP触发器将这个函数变的和普通的http server没什么两样了，你可以在函数里面服务一堆的Path，都没问题。

**但是，记住这个但是！！！**，HTTP触发器的函数，不支持websocket，不支持异步调用，如果你的网站使用了websocket，或者用了异步调用，那么你就只能借用API网关了，而我最开始的想法就是将命令的执行过程用websocket实时的传输过来。

## API网关

API网关也是阿里云的一个服务，API网关这个名字很适合他，因为这个平台是在后端和客户端之间增加一层，这层是做了什么事情呢？

1. 可以过滤一些无关的参数，并且对参数做验证。这点上对于安全性来说是比较重要的。
2. 屏蔽了后端服务和前端的直接联系。如果你的web站的安全性不怎么样，或者你不想客户端能通过抓包等方式直接找到你的后端服务器，那么借用API网关就是一个非常棒的事情。和客户端打交道的永远是API网关，有本事你先去干翻阿里云喽。

我一开始并没有想到API网关的意义，后来想想上述两点其实对于个人用户和中小企业来说还是比较重要的。

好，不说废话了，我们来讨论下API网关的用法。还是建议大家先去看看API网关的[文档](https://help.aliyun.com/document_detail/29464.html?spm=a2c4g.11186623.6.542.582e5602bIbMor)，我这里记录下我的一些收获。

### API网关的普通请求和双向通讯

API网关是有普通请求和双向通讯两种的，普通的请求其实就是传统的http请求，这里重点我们说一下双向通讯，也就是websocket协议（API网关是用的这个协议）。

首先上文档的对于双向通讯的流程![流程文字](https://haokunt-pic.oss-cn-beijing.aliyuncs.com/20200302010419.png)

![流程图](https://haokunt-pic.oss-cn-beijing.aliyuncs.com/20200302010510.png)

说实话这个流程有一部分是有点不清楚的。这里我们慢慢的解释

1. 建立websocket连接，这个建立的过程没什么魔法，标准建立连接就好。然后直接在websocket通道上发送一个注册信令，格式类似于这样子

   > `命令字：RG  `
   >
   > `含义：在API网关注册长连接，携带DeviceId`
   >
   > `命令类型：请求  `
   >
   > `发送端：客户端  `
   >
   > `格式：RG#DeviceId  `
   >
   > `示例：RG#ffd3234343dae324342@12344133`
   
2. 这个时候要等待API网关返回注册成功的信令，成功后这个时候API网关应该就做好了接收的准备了，然后再websocket通道上发送一个注册的API请求，这个时候需要增加一个标识注册API的header: `x-ca-websocket_api_type:REGISTER`，之后API网关会将deviceId以一个http协议发送到后端去，然后后端去记住这个deviceId，后端返回200后，API网关即可将注册成功的信令在websocket通道上发送给前端。这个时候请注意，websocket连接就已经建立好了，你还得用心跳信令保活，这个看看阿里云的文档。

   >  `命令字：RO  `
   >
   >  `含义：DeviceId注册成功时，API网关返回成功，并将连接唯一标识和心跳间隔配置返回`
   >
   >  `命令类型：应答  `
   >
   >  `发送端：API网关  `
   >
   >  `格式：RO#ConnectionCredential#keepAliveInterval  `
   >
   >  `示例：RO#1534692949977#25000`

   > `失败命令字：RF`
   >
   > `含义：API网关返回注册失败应答`
   >
   > `命令类型：应答`
   >
   > `发送端：API网关`
   >
   > `格式：RF#ErrorMessage`
   >
   > `示例：RF#ServerError`

3. 前面两个步骤其实已经成功的建立了一个websocket通道。之后便是两块内容了，一个是发送，一个是接收，由于客户端的请求其实没必要用websocket，因此发送这里我们就暂时不管。主谈后端向前端推送的过程。首先我们需要定义一个API，这个API是后端需要请求的API，协议可以选择http协议等，后端将需要传递给前端的消息发送到这个API上，并且带上deviceId的信息，那么对于API网关来说，他就知道需要向哪个客户端发送消息了，于是在对应的websocket通道上将消息发送过去。当然对应的也有下行通知的信令

   > `命令字：NF`
   >
   > `含义：API网关发送下行通知请求`
   >
   > `命令类型：请求`
   >
   > `发送端：API网关`
   >
   > `格式为NF#MESSAGE`
   >
   > `示例：NF#HELLO WORLD!`
   
   > `命令字：NO`
   >
   > `含义：客户端返回接收下行通知应答`
   >
   > `命令类型：应答`
   >
   > `发送端：客户端`
   >
   > `没有其他参数，直接发送命令字`

4. 最后是注销的过程，这个过程和注册类似，这里就不展开了，具体信令看看文档就知道了

> PS: 关于信令的文档: [https://help.aliyun.com/document_detail/88178.html](https://help.aliyun.com/document_detail/88178.html)

总体来说，想要使用websocket协议，我们需要另外增加3个API，分别是注册，下行通知，注销，这三个API是必不可少的，信令是用来和API网关进行通讯的，API是用来通知后端的。

说到这里，不知道有没有有人有这样的疑问，websocket通道上怎么传输HTTP请求。这个问题我一开始以为是直接将HTTP协议在websocket通道上发过去，后来发现其实不是，阿里云在这里做了一个转换，将HTTP协议转换成了json的数据格式，我们只需要将json的数据格式发送过去就好了。下面是一个示例（摘自文档）

```
POST /http2test/test?param1=test HTTP/1.1  
host: apihangzhou444.foundai.com  
x-ca-seq:0 
x-ca-key:12344133
ca_version:1
content-type:application/x-www-form-urlencoded; charset=utf-8
x-ca-timestamp:1525872629832
date:Wed, 09 May 2018 13:30:29 GMT+00:00
x-ca-signature:kYjdIuCnCrxx+EyLMTTx5NiXxqvfTTHBQAe68tv33Tw=
user-agent:ALIYUN-ANDROID-DEMO
x-ca-nonce:c9f15cbf-f4ac-4a6c-b54d-f51abf4b5b44
x-ca-signature-headers:x-ca-seq,x-ca-key,x-ca-timestamp,x-ca-nonce
content-length:33

username=xiaoming&password=123456789
```

这是一个标准的HTTP请求，那么在websocket通道上我们应该发送什么呢

```json
{
    "headers": {
        "accept": ["application/json; charset=utf-8"],
        "host": ["apihangzhou444.foundai.com"],
        "x-ca-seq": ["0"],
        "x-ca-key": ["12344133"],
        "ca_version": ["1"],
        "content-type": ["application/x-www-form-urlencoded; charset=utf-8"],
        "x-ca-timestamp": ["1525872629832"],
        "date": ["Wed, 09 May 2018 13:30:29 GMT+00:00"],
        "x-ca-signature": ["kYjdIuCnCrxx+EyLMTTx5NiXxqvfTTHBQAe68tv33Tw="],
        "user-agent": ["ALIYUN-ANDROID-DEMO"],
        "x-ca-nonce": ["c9f15cbf-f4ac-4a6c-b54d-f51abf4b5b44"],
        "x-ca-signature-headers": ["x-ca-seq,x-ca-key,x-ca-timestamp,x-ca-nonce"]
    },
    "host": "apihangzhou444.foundai.com",
    "isBase64": 0,
    "method": "POST",
    "path": "/http2test/test",
    "querys": {"param1":"test"},
    "body":"username=xiaoming&password=123456789"
}
```

这里我们可以看到，json里面有HTTP协议所需要的一些元素，首先是host，然后是path，method，还包括query参数，以及POST方法所需要的body，另外就是headers了。我们将这些json在websocket通道上发送到API网关，API网关就会自动将json还原成http请求，然后就是和你已经建立的API进行匹配，调用。

### API网关与函数计算的结合

前面主要讲了客户端与API网关的交互，现在我们讲讲API网关与后端的交互。

依然将请求分为两个部分，普通请求和双向通讯。当后端是一个传统的http server的时候，这部分很简单，几乎和没有API网关时是一样的，唯一的区别就是，在双向通讯的时候，后端需要将需要发送到客户端的内容发送至下行通知的API上。

当后端是函数计算的时候，事情就变的稍显复杂了。因为我们前面说到了，函数计算中只有HTTP触发器是遵循传统的HTTP协议的，那如果是普通调用（对于函数计算来说就是API网关触发器），那就必须进行转换（因为普通调用必是POST，且path为`/invoke`），转换过程其实在文档中也有说明，这里不详细赘述，拿个例子就行了。

```json
{
    "path":"api request path",
    "httpMethod":"request method name",
    "headers":{all headers,including system headers},
    "queryParameters":{query parameters},
    "pathParameters":{path parameters},
    "body":"string of request payload",
    "isBase64Encoded":"true|false, indicate if the body is Base64-encode"
}
```

这是API网关向后端发送的event，具体的定义大家可以看看[文档](https://help.aliyun.com/document_detail/54788.html)。

```json
{
    "isBase64Encoded":true|false,
    "statusCode":httpStatusCode,
    "headers":{response headers},
    "body":"..."
}
```

这是函数计算返回到API网关的内容。

由此即可完成一次API网关与函数计算之间的交互。

## 后记

我还没有实现这个Web命令行，主要是因为我前端的能力较差，没办法实现这个复杂的客户端部分，这又让我多了个学前端的理由（哦不对，golang的webassembly貌似可以让我不用去学前端，API这块就让assembly去搞好了哈哈哈，之后研究一下）。后端部分其实还比较简单。

后来我发现了阿里云有[云shell](https://shell.aliyun.com)，说实话这个玩意大概就是我想做的内容，但是功能更加丰富，因为阿里云内置了很多环境，各种语言环境，容器编排等等，还支持挂载NAS（就是这个NAS非要是性能型的按需付费NAS，有点贵了），还能下载文件啥的，挺爽的。但是仍然无法满足我的需求，shell分配的地区，我上次试了是上海地区，然后我试了下git命令，速度还是不太理想，时快时慢的，有点不爽，还是想把我的web命令行做好了之后放到海外的节点上试试。

我的想法是自行开发一个命令行工具，然后内置到函数计算上去。这个工具的作用就是将文件传输到OSS上，这样就能避免函数计算的高流量费，转而使用OSS的流量，这个流量就便宜许多。当然如果能借用API网关，就更好了，貌似API网关是不收费的，这样在内网让函数计算将文件传输到API网关，流量就不是走的函数计算的流量而是API网关的流量了。



