+++
title = "IPXE+netboot+ISCSI 网络启动"  # 文章标题
date = 2019-10-15T19:00:45+08:00  # 自动添加日期信息
draft = true  # 设为false可被编译为HTML，true供本地修改
tags = [""]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = [""]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

接上一篇文章[Esxi+NAS+Openwrt](https://hkvision.cn/2019/10/07/esxi-nas-openwrt/)，这篇文章我们来讲无盘启动

## 缘由
读了上篇文章的童鞋知道我是有NAS的，为了存储管理方便，服务器的硬盘我做了RAID6然后整个给了NAS做数据盘（ESXI直通），ESXI和NAS还有Openwrt的系统都是安装在U盘上的，因此如果我想继续安装其他的虚拟机的话，就必须在NAS上划ISCSI出来，并且需要PXE进行引导。

## IPXE
话说我一开始只知道PXE，后来在b站上看到了一个[视频](https://www.bilibili.com/video/av54581193?from=search&seid=15415418677811388691)，知道了IPXE这个东西，看了他操作一遍，再结合网上的一堆零碎的教程，终于我自己实现了，这玩意真的是难者不会，会者不难。

简单的理解，IPXE是PXE的加强版

## ISCSI
网上有很多介绍，那些专业化的词语我就不说了（也不会说），就简单来说，ISCSI是指在服务器上划出来一个空间（通常用一个文件表示），将此空间以磁盘的形式（注意不是目录）提供给客户端使用。对于客户端而言就仿佛是添加了一块硬盘一样，同样可以进行格式化，分区等操作，毫无问题，并且可以通过在服务器端的修改，动态的扩展和缩小磁盘大小。无盘启动就是将操作系统安装在了ISCSI中。

## 原理
贴一张网上的图片，个人觉得还是很形象的

![原理图](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1571148967074&di=361a6315668ce948410d357b8e764919&imgtype=jpg&src=http%3A%2F%2Fimg1.imgtn.bdimg.com%2Fit%2Fu%3D1783510831%2C2864552118%26fm%3D214%26gp%3D0.jpg)

1. 向DHCP服务器发送请求，请求自己的IP地址（这是肯定的，不然之后网络启动啥的就无从谈起了），注意这一步同样请求到了tftp服务器的所在位置
2. DHCP服务器应答，除了IP地址外，同样包含tftp服务器的位置
3. 向tftp服务器请求网络启动的启动镜像
4. tftp服务器回复镜像，使用镜像进行网启
5. 根据镜像配置，向有操作系统安装文件的服务器请求相关文件
6. 安装操作系统

下面我们来解答一些疑问
- 首先是tftp的配置，这里tftp服务器的地址是由DHCP服务器提供的，需要在DHCP服务器开启相关配置，等于DHCP服务器多传了一段信息，在正常的DHCP中这段信息是会被忽略掉的，只有在网络启动的时候才会被使用。OpenWRT和群晖的DHCP均可开启tftp。
- tftp服务器需要放一些什么？tftp服务器需要放PXE的相关文件，意思就是说，网启的时候会首先去读这个启动镜像来加载IPXE的启动程序，进入IPXE的启动流程中
- IPXE的配置需要我做什么？如果是看了b站的那个视频的话，其实很简单，你需要在menu.ipxe，boot.ipxe.cfg这些文件里面配置好你的ISCSI位置，操作系统安装文件的位置这两大位置，然后用IPXE的语法设计一套菜单出来（你可以用网上那些牛人写的菜单，但是他们的菜单只考虑了单ISCSI的情况，或者根本没有ISCSI，只是安装的时候进行网启）。菜单的设计也不难，首先是需要选择ISCSI的位置，然后选择是要安装系统还是直接启动，安装系统的话选择好安装的系统版本，就可以了。
- 安装操作系统的时候，安装文件放哪？安装文件需要你建站，要么建一个web站，用http进行传输，要么也可以试试ftp站，ftp协议进行传输。怎么建web站和ftp站网上有很多教程，群晖的话更简单，web station界面化建站，ftp直接开启，都是可以的。
- ISCSI呢？ISCSI仅仅是提供一块硬盘位置，群晖里面提前弄好，然后再IPXE里面配置好就行了。

## 安装后的再次启动
再次启动的话直接选择好ISCSI位置，直接boot就行

## 附上我的IPXE配置
```
#!ipxe

# Variables are specified in boot.ipxe.cfg

# Some menu defaults
# set menu-timeout 0 if no client-specific settings found
isset ${menu-timeout} || set menu-timeout 0
set submenu-timeout ${menu-timeout}
isset ${menu-default} || set menu-default exit

# Figure out if client is 64-bit capable
cpuid --ext 29 && set arch x64 || set arch x86
cpuid --ext 29 && set archl amd64 || set archl i386

###################### SAN,ISCSI SELECTION ##########################
:san-choose
menu Choose the ISCSI target
item ubuntu1              ubuntu1(ipxe.ubuntu1)
item --gap --             ------------------------- Advanced options -------------------------------
item shell                Drop to iPXE shell
item reboot               Reboot computer
item --key x exit         Exit iPXE and continue BIOS boot               
choose --timeout ${menu-timeout} --default ${menu-default} selected || goto cancel
set menu-timeout 0
goto ${selected}

:ubuntu1
set iscsi-suffix ipxe.ubuntu1
set iscsi-name ubuntu-18.04-desktop
goto install-boot

####################### Install or Boot #############################
:install-boot
menu Install or direct boot ?
item install              install
item boot                 boot
item custom-back          Back to top menu
iseq ${menu-default} install-boot && isset ${submenu-default} && goto menu-install-boot-timed ||
choose selected && goto ${selected} || goto san-choose
:menu-install-boot-timed
choose --timeout ${submenu-timeout} --default ${submenu-default} selected && goto ${selected} || goto san-choose

:boot
echo Booting ${iscsi-name} from iSCSI for ${initiator-iqn}
set root-path ${base-iscsi}:${iscsi-suffix}
set keep-san 1
sanboot ${root-path} || goto failed
goto san-choose

:custom-back
set submenu-timeout 0
clear submenu-default
goto san-choose

###################### Install ######################################
:install
menu The system image(iso)
item ubuntu18.04          Install Ubuntu 18.04 desktop (ubuntu-18.04.1-desktop-amd64)
item custom-back          Back to top menu
iseq ${menu-default} install && isset ${submenu-default} && goto menu-cinstall-timed ||
choose selected && goto ${selected} || goto san-choose
:menu-cinstall-timed
choose --timeout ${submenu-timeout} --default ${submenu-default} selected && goto ${selected} || goto san-choose

:ubuntu18.04
echo Starting Ubuntu 18.04 ${archl} installer for ${initiator-iqn}
set root-path ${base-iscsi}:${iscsi-suffix}
set base-url ${sanboot-url}ubuntu
set keep-san 1
sanhook ${root-path} || goto failed
kernel ${base-url}/ubuntu-18.04.1-desktop-amd64/install/netboot/ubuntu-installer/amd64/linux
initrd ${base-url}/ubuntu-18.04.1-desktop-amd64/install/netboot/ubuntu-installer/amd64/initrd.gz
imgargs linux auto=true DEBCONF_DEBUG=5 partman-iscsi/login/address=${iscsi-server} partman-iscsi/login/targets=${base-iqn}:${iscsi-suffix}
boot || goto failed
goto san-choose
```
目前暂时只实现了Ubuntu的，根据IPXE的官网，Linux系统与Ubuntu类似，FreeBSD应该也没问题，Windows系统需要用WinPE（微软官方的PE制作工具）