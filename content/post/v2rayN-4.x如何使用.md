+++
title = "V2rayN 4.X版本更新后如何使用"  # 文章标题
date = 2021-03-07T00:39:48+08:00  # 自动添加日期信息
draft = false # 设为false可被编译为HTML，true供本地修改
tags = ["v2ray"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["科学上网"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true

+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## 缘起

今天脑抽了更新了V2rayN，结果发现整个软件与之前大不相同，之前一直没仔细研究什么PAC什么的，这次稍微看了下V2ray的路由规则部分，然后把这个梯子又给修好了。后来发现很多小伙伴也有这个问题，在网上找到一个小白教程，想简单点的直接看那个照着做就行了，现在给出一个稍微深入的一点的解析。

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/4D-6A0qRuv4" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## V2ray路由的三大域名解析策略和三大出站类型

先声明这个是我自己起的名字，不喜勿喷

**三大域名解析策略**

- `"AsIs"`：只使用域名进行路由选择。默认值。
- `"IPIfNonMatch"`：当域名没有匹配任何规则时，将域名解析成 IP（A 记录或 AAAA 记录）再次进行匹配；
- 当一个域名有多个 A 记录时，会尝试匹配所有的 A 记录，直到其中一个与某个规则匹配为止；
  - 解析后的 IP 仅在路由选择时起作用，转发的数据包中依然使用原始域名；
- `"IPOnDemand"`：当匹配时碰到任何基于 IP 的规则，将域名立即解析为 IP 进行匹配；

这个我没研究，我就无脑选了第二个，就能用。



**三大出站类型**

1. direct: 直连类型，意思就是直接访问，不走代理
2. proxy: 代理类型，就是走代理
3. block: 阻止，即请求被拦截，不会真的发到服务器，实际上也就是没有请求

这三个出站类型分别对应了：正常访问国内网站，代理访问国外网站，阻止某些广告推送，跟踪脚本等



## 如何配置路由规则

V2ray中有路由规则，路由规则集两个概念，其中路由规则集是方便大家切换路由规则的（不然总是要编辑规则会很烦），重点在于这个路由规则，而相信大家其实只是对`geoip:cn`和`geosite:google`这种东西比较疑惑吧。



### geoip

`geoip`是一个ip地址库，大家要知道ipv4地址早就被各个国家的运营商给分完了，因此理论上任何一个公网的ipv4的地址都是可以直接找到这个ip对应的国家的（运营商，哪个省哪个市都可以，不信看[这个](https://ip.cn/)），所以当V2ray拿到一个请求的时候，通过解析dns地址，拿到A记录，去这个库里面找这个IP对应的是那个国家，然后基于这个做分流，这是完全可行的。因此这就是`geoip:cn`的作用，其中`geoip`标签代表这个规则匹配的是geoip数据库里面的数据，`cn`代表中国的国家代号，当`geoip:cn`出现在`direct`出站类型的规则下面的时候，就代表，当V2ray解析的IP是在国内的时候，就直接访问，不走代理，这非常符合我们的实际需求。

### geosite

类似于`geoip`，但是我们知道域名是在不断新增，变化的，因此这个库只是一个预定义的域名库，这个库的维护是在[这个项目](https://github.com/v2fly/domain-list-community)下面，由志愿者进行维护。这个库给很多域名分了标签，例如`apple`就是`data`目录下的`apple`文件，代表的是苹果公司的域名，分的类型非常多，大家可以去看一下。同样的`geosite:cn`代表搜集的很多的中国的域名，得益于正则表达式，可能`cn`顶级域名都在里面（没仔细看看），当`geosite:cn`出现在`direct`出站类型的规则下面的时候，就代表，当V2ray解析的IP是在国内的时候，就直接访问，不走代理，这非常符合我们的实际需求。



### DIY

![image-20210307012449979](https://haokunt-pic.oss-cn-beijing.aliyuncs.com/image-20210307012449979.png)

介绍了`geoip`和`geosite`，相信大家对V2rayN预设的绕过大陆这个规则集里面代表的什么意思有所了解了，是不是很简单，大家都可以动手DIY一下自己的规则，把自己喜欢的一些规则加进来吧。当然动手前去看看预设域名库里面有没有你想要的东西呢，例如我就看到了一个很有意思的类别`cnki`（懂的都懂），还有一个本专业的`esri`类别

## 写在最后

V2ray真的很好用，希望能多坚持一会



