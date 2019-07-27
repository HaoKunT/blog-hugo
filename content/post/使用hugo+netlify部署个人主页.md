+++
title = "使用hugo+netlify部署个人主页"  # 文章标题
date = 2019-07-27T09:20:55+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["hugo","netlify","maupassant"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["博客","hugo"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++

## 前述
起了无数次搭博客的心，前两天才下定决心用hugo弄个博客出来，毕竟码了几年的代码，也需要记点东西。
去年用django写了个博客的半成品，被老师吐槽了审美，也没兴致继续弄下去了。前段时间学了go语言，发现了hugo这个东西，听说这玩意生成网页很快，打算用这个弄。大家实践过程中如果出错了，可以先看一下后面我遇到的坑，也许对你有所帮助。

## Hugo
Hugo是由Go语言实现的静态网站生成器。简单，易用，高效，扩展性好，快速部署

### Hugo安装
安装方式很多，如果你是Mac，可以选择用`brew`安装，类似的linux可以用软件包管理工具安装(`apt`,`yum`)
``` shell
brew install hugo
```
可以选择去[github][hugo-github-release]上下载编译好的二进制包，如果你有`golang`的环境，可以选择源码编译
```shell
go get -v -u github.com/gohugoio/hugo
```

### 生成站点
使用Hugo快速生成站点，使用命令
```
hugo new site /blog
```
这样在`/blog`下生成了初始站点，`cd`进去
```
cd /blog
```

### 使用主题
到[主题列表][hugo-themes]中选择你喜欢的主题，这里我选择`maupassant`这个主题，有读者可能发现了，这个主题并没有提交到官方网站，选择这个主题主要是因为和网上推崇的`casper`主题比起来，这个主题是国人从hexo的主题迁过来的，比较符合国人的博客口味。国外的主题如果你没有fb，推特的账号的话基本上这个主题一半的功能就废了

下面我们来安装主题
``` shell
git clone https://github.com/rujews/maupassant-hugo themes/maupassant
```
安装好的主题放在`themes`下

接着是配置主题

网站根目录下新建`config.toml`文件，具体配置可看`GitHub`的[说明][maupassant-github]，下面是我的配置
``` toml
baseURL = "https://hkvision.cn"
languageCode = "zh-CN"
title = "HaoKunT的博客"
theme = "maupassant"
summaryLength = 10

[author]
  name = "HaoKunT"

[params]
  author = "HaoKunT"
  subtitle = "GIS学生党，喜欢学习新技术"
  keywords = "GIS,golang,go语言,python,HaoKunT,R,blog,博客"
  description = "数据库,Web后端,Django,Golang,GIS"
  busuanzi = true
  preserveTaxonomyNames = true
  disablePathToLower = true
  
  [params.utteranc]
    enable = true
    repo = "HaoKunT/blog-hugo-comments"    # 存储评论的Repo，格式为 owner/repo
    issueTerm = "pathname"  #表示你选择以那种方式让github issue的评论和你的文章关联。
    theme = "github-light" # 样式主题，有github-light和github-dark两种

  [[params.links]]
    title = "R语言空间数据处理与分析实践教程"
    name = "R语言空间数据处理与分析实践教程"
    url = "https://baike.baidu.com/item/R%E8%AF%AD%E8%A8%80%E7%A9%BA%E9%97%B4%E6%95%B0%E6%8D%AE%E5%A4%84%E7%90%86%E4%B8%8E%E5%88%86%E6%9E%90%E5%AE%9E%E8%B7%B5%E6%95%99%E7%A8%8B"

  [[params.links]]
    title = "CS-Tao的博客"
    name = "CS-Tao的博客"
    url = "https://home.cs-tao.cc"

  [[params.links]]
    title = "HPdell的博客"
    name = "HPdell的博客"
    url = "https://hpdell.github.io"

[menu]

  [[menu.main]]
    identifier = "tools"
    name = "工具"
    url = "/tools/"
    weight = 2

  [[menu.main]]
    identifier = "archives"
    name = "归档"
    url = "/archives/"
    weight = 3

  [[menu.main]]
    identifier = "about"
    name = "关于"
    url = "/about/"
    weight = 4

[params.cc]
    name = "知识共享署名-非商业性使用-禁止演绎 4.0 国际许可协议"
    link = "https://creativecommons.org/licenses/by-nc-nd/4.0/"
  
[permalinks]
  post = "/:year/:month/:day/:title/"
```
### 配置默认`Front Matter`
写文章之前我们先配置一下文章的默认`Front Matter`

网站根目录下新建文件`archtypes/default.md`，下面是我的配置
``` toml
+++
title = "{{ replace .Name "-" " " | title }}"  # 文章标题
date = {{ .Date }}  # 自动添加日期信息
draft = true  # 设为false可被编译为HTML，true供本地修改
tags = [""]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = [""] # 文章类型
toc = true # 用于开启文章目录
preserveTaxonomyNames = true
disablePathToLower = true
+++
```

### 写文章
OK前期所有的工作都完成了，我们可以开始愉快的写文章了，执行下面命令
``` shell
hugo new post/first.md
```
命令会在根目录下新建`content`目录，这个目录保存了你所有的文章内容，会新建一个目录`post`，下面则是你的各篇文章

打开`first.md`你会惊奇的发现自动给你添加了`Front Matter`，你根据你这篇文章的内容修改相应的头，例如这篇文章的头
``` toml
+++
title = "使用hugo+netlify部署个人主页"  # 文章标题
date = 2019-07-27T09:20:55+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["hugo","netlify","maupassant"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["博客","hugo"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++
```
然后开始写你的文章主体。

### 运行
文章写完后就可以开始运行了
``` shell
hugo server -t maupassant
```
这个命令用于开启本地调试，还可以自动更新，得益于Hugo的高效，你可以一边写文章一边实时的查看你的文章的网页表现

浏览器里打开: `http://localhost:1313`

> 这里插一句`归档`页面的事
> `归档`页面需要你在`content`下新建`archives`目录，新建`index.md`，然后填入以下内容
``` yaml
---
title: "归档"
description: GIS学生党一枚，喜爱Python和Golang，欢迎探讨GIS和Web开发
type: archives # 这里类型一定要是archives
---
```
> 这样才能正常归档


## 部署netlify
我博客部署在`netlify`上面，不选`GitHub Pages`主要是因为百度收录不了`GitHub Pages`，具体原因参考[参考][baidu-crow-github-pages]，作为`“内事不决问百度，外事不决问谷歌”`的`“内事”`，百度还是不能随便放弃的，于是我打算用`netfily`，当然你也可以用`Coding`了。

`Netlify`官网：[https://app.netlify.com](https://app.netlify.com)

进去后直接用`Github`登录，安装`Github app`，反正按照流程来就行了，记得直接将网站所在的repo作为持续部署的库，会自动识别这个库是Hugo的库，command那里会自动填充`Hugo`，`Publish directory`那里会自动填充`public`

一切顺利的话你的网站已经成功部署了，netlify也提供二级子域名供你使用，如果你有自己的域名的话，也可以自定义域名，然后在域名服务商那里调整一下解析路线。

`netlify`提供持续集成，自动HTTPS，检查是否混用等实用功能，当你更新文章后直接推送到github仓库，`netlify`会自动重新构建，构建成功会自动发布，非常方便。当然`GitHub Pages`也可以做持续集成，自己写一个小`shell`就行，网上教程很多，就不列举了。

## 几个坑
- `BaseUrl`一定要配置好，特别是后面用自己的域名的时候，或者你一开始就准备好了部署的域名。若出现更换域名的情况，则除了修改DNS以外，还要修改`BaseUrl`，重新发布。
- 大家可能发现`关于`页面，`工具`页面好像返回的`404`。其实这是因为你还没有写对应的文章，你可以在`content`下新建一个`about`目录，然后新建`index.md`，这里写自己的关于页面，`工具`页面同理
- 用`git clone`下来的主题其实都是`submodules`，但是没有相应的`.gitmodules`文件，而netfily只认`submodules`，只认`.gitmodules`文件，所以在下载主题的时候，要么用`git add submodules`命令，要么就手动添加一个`.gitmodules`文件，文件内容如下
``` .gitmodules
[submodule "themes/maupassant"]
	path = themes/maupassant
	url = https://github.com/rujews/maupassant-hugo
```
这是最大的坑，很折磨人，尤其是我对git的子模块一点不熟
- 有些人可能会问我明明添加了文章但是为什么不显示呢？通常这是因为文章的`draft`参数没有设置为`false`，设置为`true`的时候代表是草稿，是不会渲染的，所以你看不到。其实这个很简单但是刚开始用的时候总是会忘。

> 祝大家每一步都顺利

> 最后在这里感谢[JokerQyou][JokerQyou]，[飞雪无情][飞雪无情]提供的主题




[hugo-github-release]: https://github.com/gohugoio/hugo/releases
[hugo-themes]: https://themes.gohugo.io/
[maupassant-github]: https://github.com/rujews/maupassant-hugo
[baidu-crow-github-pages]: https://extendswind.top/posts/technical/hugo_blog_host_and_seo/
[飞雪无情]: https://www.flysnow.org/
[JokerQyou]: https://blog.mynook.info/

