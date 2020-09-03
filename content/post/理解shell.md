+++
title = "理解shell"  # 文章标题
date = 2020-05-20T00:45:40+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["linux", "shell"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = [""]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true

+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## 缘起

最近在看《深入理解操作系统》，抛开了汇编那一块，书里面很多例子是拿Unix shell来举例的，这使得我对此有了更多的感悟，特此记录一下。



## shell与终端

shell不是终端，平时容易弄混的原因是因为这两个东西总是一起出现。准确来说，单独的终端是没有任何意义的，因为这个终端只能回显你输入的一堆字符，而不能执行你输入的命令，所以一般来说，你打开一个终端之后（打开终端是处于一个进程中），同时会执行一个shell（linux中是调用`execve`这个函数来在一个进程中执行另外一个程序，可以先`fork`一个子进程，然后在这个子进程里面执行shell）。

## shell做的事情

基本的shell功能就是解析用户输入的命令，解析为可执行文件，参数，环境变量等等，然后fork一份子程序，子程序调用`execve`函数来执行你输入的命令。此外，shell会内置一部分指令，例如`quit`这个指令并不是一个可执行程序，但是输入在shell中就会退出shell。

一个简单的shell包含三个不同的函数

- `parseline`: `parseline`函数用于解析你输入的指令，参数，环境变量
- `builtin_command`: `builtin_command`用于判断是否为内置指令，若是则直接执行内置指令，并且返回非零值，告诉`eval`我已经执行过了指令
- `eval`: `eval`用于执行整体流程

```C
int main() {
    while(1) {
        char cmdline[1024];
        ...
        eval(cmdline)
    }
}

void eval(char* cmdline) {
    ...
    parseline(buf, &command, &par, &eniron);
    if (builtin_command(command) == 0) {
        if (Fork() == 0) {
            ...
        }
    }
}
```

上面是一些伪代码，基本上可以勾画出整体的轮廓

## 前台与后台

这也是属于shell中的一个概念，即我的指令是放在后台执行的，前台执行的程序会阻塞shell直到程序运行完，这等于是在进入shell下一次while循环之前，就等待前台子进程执行完，直到子进程执行完后返回信号，后台程序则是不阻塞，而是注册一个信号处理函数，并将fork的子进程pid存起来，当有子进程结束的信号的时候，则对子进程pid的列表进行清理，并在前台显示子进程结束。当然你也可以选择忽略信号（默认行为），缺点是你不知道子进程到底是何时结束的。