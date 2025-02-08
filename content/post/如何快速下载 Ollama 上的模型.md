+++
title = "如何用 ollama 快速下载 deepseek 模型"  # 文章标题
date = 2025-02-09T01:46:34+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["LLM", "本地部署"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["LLM"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
[graphviz]
  enable = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## 写在开头
最近由于工作的原因，已经有段时间没有更新博客了。一方面担心可能会引发不必要的纠纷，另一方面确实因为工作太忙、太累，实在抽不出时间和精力来写文章。不过，恰逢DeepSeek R1的热度持续上涨，我也想借这个机会，分享一个之前踩到的小坑，希望能对大家有所帮助。

## 背景
目前网上教的本地通过 ollama 来部署大模型的方式，都需要使用 `ollama run <model>` 这样的命令，或者通过一些可视化的前端来安装模型，但是这样的命令很可能因为直连，出现下载缓慢的状况，针对这样的状况，目前，网上的教程大多建议用户从ModeScope下载模型文件后手动包装modelfile文件再给Ollama使用。这种方法不仅步骤繁琐且容易出错。

既然是下载慢的问题（并且原因大家应该明白），那么我们可以使用比较经典的解决方案（大家应该懂，这里说多了容易被封，下文用T代替），但是我最初尝试了半天都失败了，原因是没用上T，后来发现是踩了个小坑。

## Ollama 的服务端/命令行/UI界面

```viz-dot
digraph ollama_components {
    graph [rankdir=TB];
    
    node0[label="远端模型文件"];
    node1[label="Ollama Server"];
    node2[label="Ollama 命令行"];
    node3[label="Ollama 界面"];
    
    node0 -> node1[dir=back];
    node1 -> node2;
    node1 -> node3;
}
```

上图是一个简易的 Ollama 部署在本地时的架构，当你完成 Ollama 安装的时候，实际上是在你电脑上打开了一个 Ollama Server，然后当你使用`ollama run <model>`命令的时候，实际上是向本地的 Ollama Server 发送了一个指令，然后 Ollama Server 会检查本地有没有这个模型，如果没有的话就会去远端下载对应的模型文件。

这里大家要注意的是，实际进行下载的程序是 Ollama Server，而这个 Ollama Server 安装之后就是一个跑起来的服务，我的电脑是 Windows 的系统，这里就是一个启动项，我对 Windows 系统做服务器不甚熟悉，找了下没找到能往启动项里面注入环境变量的方法，但是因为 Ollama Server 其实是可以用过命令行`ollama serve` 启动的，这就给了我注入环境变量的机会。而且只是在下模型的阶段用一下就行了，其他时候还是可以直接用那个启动项的。

> 这里解释一下为什么注入环境变量，由于 ollama 是一个已经写好的程序，一般如非必要，一般不会改源码然后重新编译二进制，而这种情况下如果想用上T，在命令行环境下一个通常的做法就是注入环境变量，一般程序都会默认从这个环境变量里面读T的配置然后使用T，这就做到了不改源码的情况下使用上T。

## 大致流程
好了，想必大家应该明白我的意思了，下面说一下大概流程
1. 关掉 ollama 服务。
2. 开一个终端，然后注入环境变量。
3. 在前面的那个终端下执行 `ollama serve`
4. 重新开一个终端，执行 `ollama run <model>`
如果一切顺利的话，那么你就应该能看到下载速度飞起来了，当然前提是你的T得够快。

## 前后对比
- 原速

C:\Users\admin>ollama run deepseek-r1:1.5b
pulling manifest
pulling aabd4debf0c8...  81% ▕█████████████████████████████████████████████           ▏ 899 MB/1.1 GB  1.3 MB/s   2m44s 
C:\Users\admin>ollama run deepseek-r1:1.5b
pulling manifest
pulling aabd4debf0c8... 100% ▕███████████████████████████████████████████████████████ ▏ 1.1 GB/1.1 GB  205 KB/s     17s

- 加速

C:\Users\admin>ollama run deepseek-r1:1.5b
pulling manifest
pulling aabd4debf0c8...  48% ▕██████████████████████████                              ▏ 536 MB/1.1 GB   19 MB/s     29s

根据我的测试，加速后不仅速度更快，而且下载过程也更稳定，不会出现最后一点很慢，下不动的情况。

## 我踩了什么坑
一开始我一直以为实际下载的就是那个命令行程序，因此一直在那上面较劲，但其实是 server 实际承担了下载任务。

## 最后
希望这篇文章对你有用，当然，用不上最好，那代表你原速就很快哈哈哈！
