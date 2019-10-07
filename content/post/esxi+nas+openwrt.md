+++
title = "Esxi+NAS+Openwrt"  # 文章标题
date = 2019-10-07T23:42:21+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["esxi","nas","openwrt"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["服务器","运维"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
[graphviz]
  enable = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

接上一篇文章：[Esxi的安装和使用](https://hkvision.cn/2019/10/01/esxi%E7%9A%84%E5%AE%89%E8%A3%85%E5%92%8C%E4%BD%BF%E7%94%A8/)，这一篇文章我们通过esxi来搭建一个虚拟机网络

## 框架
首先我们明确一个概念，整个esxi是将主机进行了虚拟化，因此我们可以任意使用宿主的计算和存储资源。下面是我设想中的一个网络架构

```viz-dot
graph G {
    size = "8, 10"
    esxi[label="ESXI主机"]
    nas[label="NAS"]
    openwrt[label="Openwrt"]
    xu1[label="虚拟机1"]
    xu2[label="虚拟机2"]
    outnet[label="外部网络"]
    vswitch1[label="虚拟交换机1" shape=box]
    vswitch2[label="虚拟交换机2" shape=box]

    subgraph cluster_switch1{
        label="有外网ip的"
        bgcolor="#73A04F"
        outnet -- vswitch1
        vswitch1 -- {esxi openwrt nas}
        
    }

    subgraph cluster_switch2{
        label="无外网ip的"
        labeljust = "r"
        labelloc = "c"
        bgcolor="#73A04F"
        {nas openwrt} -- vswitch2
        vswitch2 -- {xu1 xu2}
    }
}
```
首先明确一点，与`VMware Workstations`不同，`ESXI`中没有NAT网络，所有设备均通过虚拟交换机进行连接，虚拟交换机可以设定上行链路，也就是物理网卡连接的链路，`EXSI`本身也连接在虚拟交换机上，也就是说，所有连接在交换机上的虚拟机天生就是桥接模式。当你IP供应充足的时候，你可以将所有虚拟机都连在一个交换机上，毫无问题。由于当初网管只给了我们5个ip，因此我们需要将部分虚拟机用nat技术组成一个小型局域网，这就是使用`Openwrt`软路由的原因。

`虚拟交换机1`是有上行链路的，即与外部网络相连，这里的`NAS`和`Openwrt`均有外部网络的IP，然后将`Openwrt`,`NAS`与`虚拟交换机2`相连，其余新增的虚拟机也与2号交换机相连，这个小型局域网由`Openwrt`进行路由管理。这样`NAS`就有两个网卡，一个与外部网络相连方便我们管理，另外一个与局域网相连，以便其余虚拟机能访问`NAS`

## 安装
### NAS安装
这里的NAS我们选择黑群晖，在网上下载好黑群晖的镜像（论坛上），按照教程安装，黑群晖的安装首先是给虚拟机分配两个盘，一个是现有的引导磁盘（就是网上下好的），另外一个是放系统的磁盘，记得用SATA控制器，不然可能会引导不了。引导完成后（看到`booting kernel`），用群晖助手（论坛或群晖官网）搜索

> 这一步注意：由于外部网络没有DHCP服务器，因此此时黑群晖并没有被分配到IP，浏览器搜索是无效的，必须用助手，推测助手的搜索是在链路层进行的。另外不能在ESXI里面设定IP，因为ESXI根本不管这个事情（应该可以ESXI开DHCP，但是ESXI连在外部网上，开了DHCP可能就乱套了，还是不进行危险的操作）

搜索到了之后还是用助手，助手会默认在浏览器里面打开未安装的群晖的那个ip，但是由于这个ip是一个保留地址，无效，所以我们需要用助手来进行安装，这里右键你看到的那个黑群晖，然后安装黑群晖，这一步把那个大的安装文件拖过去安装就好了，这里按照提示来就好。

### Openwrt安装
Openwrt的安装也很简单，去官网上下载x86/x64的固件，然后用`qemu-img`转一下格式，转成`vmdk`格式，直接用这个虚拟机磁盘启动就好了

> 注意这里有一个小坑：当你感觉你的启动卡住了的时候，多按几下回车，直到出现命令行，网上说要PS/2的键盘，我这里反正是虚拟机，大家直接按就好。

启动完成后配置一下网络

### 网络配置
首先我们新建两个端口组，然后新建一个交换机，将其中一个端口组（bridge）连上自带的那个交换机（可以访问外部网络的），另外一个端口组（nat）连上新建的交换机（openwrt），新建的交换机不要设置上行链路

我们将NAS连到端口组bridge,nat上（需要两个网络适配器），Openwrt也是一样的，这样网络就搭建好了，以后再创建虚拟的时候就将网络适配器连到nat这个端口组上就好。

### 配置Openwrt
然后我们配置一下Openwrt，记住，连到bridge上的网卡是软路由的wan口，另外一个是lan口，这里wan口设置一个静态地址，lan口设置DHCP就好，这里lan口是路由模式不是桥接模式。这里贴一下我的配置

``` config
# vim /etc/config/network
config interface 'loopback'
        option ifname 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'

config globals 'globals'
        option ula_prefix 'fdaa:49d4:569a::/48'

config interface 'wan'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '192.168.190.208'
        option netmask '255.255.255.0'
        option gateway '192.168.190.254'
        option dns '223.5.5.5'
        option ip6assign '60'

config interface 'wan6'
        option ifname 'eth0'
        option proto 'dhcpv6'

config interface 'lan'
        option ifname 'eth1'
        option proto 'static'
        option ipaddr '192.168.1.1'
        option netmask '255.255.255.0'
        option ip6assign '60'
```
这一步完成后你的Openwrt就已经可以正常工作了，但是我们肯定需要web管理界面，也就是`luci`，但是我们注意，在管理的时候，我们是从wan口进去的，但是默认情况下Openwrt是会拒绝wan口的流量的，因此我们需要设置一下防火墙，这里修改一下

``` config
# vim /etc/config/firewall
...
config zone
        option name             wan
        list   network          'wan'
        list   network          'wan6'
        option input            ACCEPT
        option output           ACCEPT
        option forward          ACCEPT
        option masq             1
        option mtu_fix          1
...
```
重启一下网络和防火墙
``` shell
/etc/init.d/network reload
/etc/init.d/firewall reload
```

安装一下luci
``` shell
opkg update
opkg install luci
opkg install luci-i18n-base-zh-cn // 这里安装中文翻译
```
然后就大功告成了，直接在浏览器里面输入Openwrt的地址就能访问到luci的管理界面了，记得提前设好root密码。

暂时只写到这里，我继续摸索第三部分，ipxe+iscsi无盘启动，未完待续...
