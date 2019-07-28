+++
title = "SWIG 以Python为例"  # 文章标题
date = 2019-07-26T06:12:25+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["swig", "python", "混合编程"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
toc = true
categories = ["Python","SWIG"]
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## SWIG用途
SWIG是用于开发C/C++与Java，Python，Perl，C#等高级语言之间接口的框架，利用这个框架，我们可以很方便的将C/C++程序应用在Python等高级语言中。

## SWIG的步骤
### 以Python为例
- 安装SWIG([官网](http://www.swig.org/)下载)，选择对应的环境下载即可，安装完成后将swig命令所在目录添加至环境
- 准备好源文件和头文件
``` C++
// example.h

#include <iostream>
using namespace std;

class example {
    private:
        int num;
    public:
        void say_hello(void);
        void change(int din);
        int get_num();
};

```
``` C++
// example.cpp

#include "example.h"

void example::say_hello(void) {
    cout << "hello python,I am C++" << endl;
}

void example::change(int din) {
    num = din;
}

int example::get_num(void) {
    return num;
}
```
另外，需要编写对应源文件和头文件的i文件，i文件的具体语法请参考[官方文档](http://www.swig.org/doc.html)
``` SWIG
// example.i
%module Example_swig // 模块名

%{
#include "example.h" // 这里是用于编译时需要的环境
%}

// 这里是用于生成中间代码时需要的Interface，若想将整个头文件中的声明全导出，则选择使用%include "example"方式
// 若想导出部分，则将想导出的声明放在这里
%include "example.h" 
```

- swig生成中间代码
`swig -python -c++ example.i`

- 下面是以VS2017为编译环境，具体编译环境请以Python的底层编译环境为准，若不知道底层编译环境是什么，命令行`python`，出来的头部信息中带有相关的信息，例如Python3.6是MSVC140，即VS2015的编译器编译的。
    1. 首先建立空工程，选择输出dll
    2. 然后将源文件和头文件添加至项目中（可能还需要添加i文件），将生成中间代码步骤中生成的cxx文件添加至项目（这个文件才是生成最后动态链接库的主力文件）
    3. 然后将Python的include文件夹与libs文件夹分别添加至包含目录与附加库目录中，具体做法请[参考](https://www.jianshu.com/p/a257e630fe42)
    4. 若有其他需要添加的环境等请一并配置好
    5. 最后生成，然后将生成的dll文件改名为`_模块名.pyd`，将生成中间代码步骤中生成的py文件和pyd文件放在一起，就大功告成了。

## SWIG需要注意的地方
1. 整个过程分为两部分，第一部分是生成中间代码，第二部分是编译，生成中间代码会执行不含%{....%}的部分，也就是说如果生成中间代码出了错，请不要在这部分找问题
第二部分是编译，生成中间代码的步骤中包含%{....%}的地方将会被放在cxx文件中，也就是说这部分是在编译时起作用的。
2. 生成中间代码仅仅是生成待编译的cxx文件，swig只需要找到接口的声明就好，不关心其实现。
3. SWIG支持的语法不多，若生成中间代码步骤中报错，可以考虑是SWIG的问题，尽量采用ANSI C的语法。

## 编译整个海康威视SDK
如果你想编译整个SDK的话请参考我另外一篇[文章](https://hkvision.cn/2019/07/26/swig%E7%BC%96%E8%AF%91%E6%B5%B7%E5%BA%B7%E5%A8%81%E8%A7%86sdk-%E4%BD%BF%E7%94%A8golang/)




