+++
title = "Web终端仿真器"  # 文章标题
date = 2020-03-06T23:10:14+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["web", "tty"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["技术"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true

+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)



## 背景

​		在我的上一篇文章中已经说过了，我想做一个web命令行。但是呢，我这两天碰到了一些问题，主要是基础不牢引起的一些概念性的问题，导致我在搜索资料的时候花费了很多时间。

​		最开始我碰到的问题是我在执行`git clone`命令的时候，我发现如果我把执行结果重定向到一个文件上，或者是我用Golang的`exec.Command`命令执行的时候将结果用管道扔到我的程序里，执行的结果就只有一行`Cloning xxx into xxx...`，而正常的结果应该是像这样的

![image-20200306231815194](C:\Users\11515\AppData\Roaming\Typora\typora-user-images\image-20200306231815194.png)

​		这个问题我一开始以为是clone的进度是一个类似于进度条，导致这种进度条无法被重定向，一度找问题找错了方向。后来发现不是这个原因，我在[这个地方](https://stackoverflow.com/questions/32685568/git-clone-writes-to-sderr-fine-but-why-cant-i-redirect-to-stdout)找到了原因，原来是因为git这个命令，将stdout和stderr赋予了不同的涵义，stderr是一个**人更感兴趣**的输出，而stderr是**机器更感兴趣**的输出，因此有一个静默模式，就是说在git发现输出的终点不是一个终端（terminal）的时候，将默认不将人感兴趣的内容输出，也就是更加的安静（quiet），在git的[文档](https://mirrors.edge.kernel.org/pub/software/scm/git/docs/git-clone.html)中我们也可以发现这点，这里引文档的这段话

> --progress
>
> Progress status is reported on the standard error stream by default when it is attached to a terminal, unless `--quiet` is specified. This flag forces progress status even if the standard error stream is not directed to a terminal.

​		好，找到这个问题了，我们的目标就是让程序的输出变成一个终端。首先要明确的是什么是终端，这里因为我的概念不熟悉的原因又将我引入了死路，我在github上搜索的关键词一直是`webshell`，这倒是让我搜出来了一些东西，但是重点是并不是我想要的，例如我找到了很多个在web上执行自定义命令的，啊反正很多。直到我找到了[这篇文章](https://www.cnblogs.com/sddai/p/9769086.html)，这篇文章一扫我心中的阴霾，简直就是豁然开朗。

​		那么到底是什么问题呢？实际上我将`shell`和`terminal`这两个东西弄混了，相信很多人也是像我这样的。具体的大家去看这篇文章，我这里简单的说一下。`shell`是和`bash`,`sh`,`zsh`对应的，`terminal`是和`tty`对应的，相信大家对这几个内容肯定有所耳闻，我平时接触到的肯定是bash居多，tty这玩意见的不多，当时也没在意，就把这两者弄混了。其实shell更多的是指执行命令的程序，一个用户用来操作系统的入口，而terminal则是指的我们见到的黑框框这个界面本身。

​		不知道大家有没有一个疑惑，就是为什么我们的命令行是怎么将我们的键盘输入显示在界面上的，其实这一步就是终端做的，终端是一个用于捕捉用户的输入，还有光标的移动等等的处理程序，而shell是用于执行命令的程序，也就是说，当你敲下回车的时候，终端将你输入的这一行内容交给shell来执行，并将shell的输出显示在终端中。

> 现在我们知道，终端干的活儿是从用户这里接收输入（键盘、鼠标等输入设备），扔给 Shell，然后把 Shell 返回的结果展示给用户（比如通过显示器）。而 Shell 干的活儿是从终端那里拿到用户输入的命令，解析后交给操作系统内核去执行，并把执行结果返回给终端。

​		当然实际情况更加复杂，大家可以看推荐的那篇文章。

## web终端

​		知道了终端和shell的区别了之后，我就明白了为什么git指令会进入静默模式。那么我们首先就是要去找一个终端仿真器（terminal emulator），github上有很多大神写的终端仿真器，我得要求是能在web里面运行的，远程的。最终我找到了两个库——`xterm`和`hterm`，有一篇[文章](https://www.v2ex.com/t/642177)比较了两个库的优劣。我还并没有决定使用哪个库。

​		用库不是重点，重点是我依然没有弄明白怎么打开一个终端，这带来的最大的一个问题就是，我无法自己写代码来实现自己的程序（用websocket或者其他的手段将远程的终端映射到浏览器里面），我只能用别人写好的，这当然不行，大家都用的websocket协议，但是就像我上篇文章写到的，阿里云的API网关对websocket协议的通讯做了一大堆的包装，github上的程序都不满足我的需求。直到我看到了一段代码

```go
userFile := "/dev/tty"
```

或者还有这段

```go
func open() (pty, tty *os.File, err error) {
	p, err := os.OpenFile("/dev/ptmx", os.O_RDWR, 0)
	if err != nil {
		return nil, nil, err
	}

	sname, err := ptsname(p)
	if err != nil {
		return nil, nil, err
	}

	err = unlockpt(p)
	if err != nil {
		return nil, nil, err
	}

	t, err := os.OpenFile(sname, os.O_RDWR|syscall.O_NOCTTY, 0)
	if err != nil {
		return nil, nil, err
	}
	return p, t, nil
}
```

这让我明白了一些东西，就是我原先就知道的，linux里面任何设备都是一个文件，所以其实终端（terminal）作为一个古老的设备，其在linux的表现一定会是一个文件。第一步就走出来了，所以我如果想打开一个终端，我只需要打开这个设备文件就好了。

​		今天先写到这里，主要是我还没有写这个程序，但是经过我查了这么多资料，程序大概怎么写我已经明白了，剩下的就是在写代码的过程中继续深挖了。我有预感这个程序会大大的增加我对计算机的认识，过程很艰难，希望这个程序能走下去。