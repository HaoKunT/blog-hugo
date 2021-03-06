+++
title = "文件系统介绍"  # 文章标题
date = 2020-09-03T18:00:40+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["文件系统"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["文件系统"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true

+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## 缘起

这段时间研究了很多和文件系统有关的内容，于是打算写一篇博客记录一下，最近太忙都没时间打理博客了。

## 为什么要文件系统

文件系统这个东西是一个随着计算机发展而必然会产生的产物，我们以个人PC为例，现代的个人PC五大件：CPU，主板，显卡，硬盘，内存。文件系统就是专门为硬盘准备的。

我们想象一个场景，CPU哼哧哼哧的算了一个结果出来，放在内存里面，这个时候你想让这个结果持久化，也就是永久保存，即使我关机了，重新开机之后我也能读到我计算的结果，硬盘的作用就体现出来了，我们将结果放在硬盘里面，下次咱就可以读到了。但是我们思考一下，结果是如何存在硬盘中的呢？

### 硬盘

首先我们来看一下硬盘是啥，这里我们先局限于机械硬盘（HDD）。

硬盘是一个能持久化保存数据的设备，从逻辑上，我们可以将硬盘视为一个很大的线性数组，往这个线性数组里面存储数据，就可以让数据被持久化。当然，硬盘有自己的具体的读写操作，例如硬盘一般是按512字节作为一个扇区进行读。

### 最简单的存

最简单的存就是直接向这个线性数组写数据，这是最直接的，例如一个电子血压计，我们可以查到前50次的血压的历史数据，所以我们很简单的用一个101字节大小的磁盘，其中第一个字节存上一次的数据的位置，然后向后循环读取，2个字节存每一次的数据，这是最简单的方案。

但是这存在一个问题，随着我们要存的数据越来越复杂，硬盘越来越大，我们得知道我们的数据放在哪了，不可能存数据的时候还自己确定一下存第几个字节到第几个字节这种。因此，我们必须寻找其他的替代方式。

### 文件系统存

替代方案就是文件系统，几个概念完成了对文件系统的基本描述。

- 超级块

- inode

- 目录项

- 文件

这也是Linux中的VFS的四种主要对象。通过建立文件树这样的概念，将需要保存的数据抽象成了文件，并以树形的方式进行组织，以此来保存通用数据。

## 文件系统是怎么存的

这里我们以Linux为例（其实是因为Windows的没去了解），上面说了文件系统种最重要的四个对象，下面来介绍这四种对象分别起到了什么作用。

### 铺垫：块

在正式介绍之前，首先得介绍块的概念。

在Linux系统中，我们将类似于硬盘的存储设备称为块设备（block device），为何取这样的名字呢？是因为像硬盘这样的设备，在硬件上进行读取的时候，是按照一块一块这样读的（在Windows上称为簇）。

在文件系统中，通常将一个块设为4K的大小，目前并没有证据表明设为更大的值会带来啥性能上的提升，4K是一个约定俗成的大小，但是据一些比较玄学的说法，4K正好是目前大部分处理器MMU的大小，4K的块大小会给程序编写上带来很多方便。通常这个大小不用改变。

像目前的SSD都要求4K对齐，其实是为了保证在进行硬件读写的时候正好操作的是一个块，否则跨块操作，会带来极大的性能浪费（本来操作一个块的结果硬是要操作2个块）。

简而言之，块是文件系统操作的最小单位。

### 超级块

超级块（super block）是用于存储文件系统元数据的块，通常位于文件系统开头（也就是逻辑上硬盘的最开始的位置）。超级块中存储了很多很多文件系统的元数据，例如文件系统总大小，已使用的大小，空闲的大小，总块数，已使用块数，空闲块数，已使用的inode数，空闲inode数，上次挂载的时间，修改时间，开启了哪些特性，UUID等等等等。

总而言之，超级块就是一个用于描述文件系统的块，如果是查这个文件系统这一层级相关的东西，那肯定是找超级块。

### inode

inode（索引节点）是用于描述文件的结构，里面会存和某个具体的文件相关的东西，例如文件大小，权限，三个时间戳（atime, ctime, mtime），以及最重要的，文件的起始块位置。这样，这个文件在哪，多大等一些信息就知道了，需要注意的是，inode里面没有文件名，文件名在目录项里面。

### 目录项

在介绍目录项（Directory entry）之前，首先我们要介绍一下，Linux将一切都视为了文件，因此理论上来说，目录也是属于文件，但是目录项被单独拎出来是因为，目录项是一个特殊的文件，里面存储的是这个目录下的文件名以及其文件对应的inode号。

VFS的dentry设计上就是为了实现整个文件系统树状层次结构的，每个文件系统拥有一个没有父dentry的根目录（root dentry），这个dentry会被superblock引用，用来作为进行树形结构的查找入口。其余的所有dentry都是有唯一的父dentry，并可以由若干个孩子dentry。示例：对于一个文件`/home/user/a`，会存在`/`、`home`、`user`、`a`四个dentry，依次构成父子关系，每个dentry也都有一个inode与之关联，存储了具体的数据。

### 文件

文件当然就是具体的文件了，VFS里面文件其实是文件对象，还与其相关联的进程有关系，但是这是在内存里面的表现形式，在硬件设备上，这玩意的表现就是一段或几段连续的字节。

## 举个例子

举个例子，如果我要读`/home/test/hello_world.c`这个文件，文件系统到底是怎么保证其找到的？

首先，读取超级块（超级块总在开头），找到超级块里面存的根目录的inode，从这个inode找到根目录的目录项，从目录项里面找到home的inode，然后读这个inode，然后找到home的目录项，然后这样找到test的目录项，找到`hello_world.c`这个文件的inode，读取其属性数据，最后找到文件在硬盘上存的位置。

