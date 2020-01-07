+++
title = "利用acme自动更新证书"  # 文章标题
date = 2019-09-24T17:55:50+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["SSL"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["技术"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true
+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## acme
`Let's Encrypt`制定的自动化证书管理协议，通过这个协议我们可以自动更新证书，无需人工干预


## acme.sh
`github`上已经有一个[项目](https://github.com/Neilpang/acme.sh)，用shell写的脚本，功能很强大，直接用就行了

### 安装
``` shell
curl https://get.acme.sh | sh
```
重新载入一下`.bashrc`
``` shell
source ~/.bashrc
```
之后你就可以直接用`acme.sh`命令了，有alise

### 申请证书
申请的方式有几个，大致的流程是

- 提出申请
- `Let's Encrypt`给你提出挑战（challenge），挑战分别有两种，1.网站根下弄一个文件。2.DNS解析加一条TXT记录。随便验证一条就行了
- 完成挑战中的一种
- 签发

整个过程很清晰，主要是验证你对这个网站有操作权。这里我们选择DNS解析，第一种需要你本身有一个http站点，而且最关键的问题是80端口要开，这对于80被封掉的人来说简直就是一个悲剧。DNS验证就没这个问题了。
``` shell
acme.sh --issue --dns dns_ali -d xxx.xxx.com
```
具体你哪个DNS提供商的具体指令看[这里](https://github.com/Neilpang/acme.sh/wiki/dnsapi)
阿里云的需要你在环境变量里面加key和srcret
```
export Ali_Key="xxxxxxxxxxxx"
export Ali_secret="xxxxxxxxxxx"
```
> 阿里云现在子账号里面支持DNS解析的权限了

成功的话你会看到你的证书，并且提示你成功了，证书文件放在`~/.acme.sh`目录里面，你之前的配置什么的脚本都放在了配置文件里面，你不用担心下次执行的时候找不到配置

### 安装证书到nginx
记住不要直接把证书放到nginx的配置下，要用脚本提供的命令，因为这样脚本在证书快过期的时候自动更新才会知道应该放到哪，并做什么操作
``` shell
acme.sh --installcert -d xxx.xxx.com \ 
               --key-file      /home/ubuntu/www/ssl/xxx.xxx.com.key  \ 
               --fullchain-file /home/ubuntu/www/ssl/xxx.xxx.com.key.pem \ 
               --reloadcmd     "sudo service nginx force-reload"
```
`--reloadcmd`会让脚本在更新完后执行一个命令，这里就是重新载入配置

### 修改sudoer
修改一下sudoer
``` shell
sudo visudo
```
新增一条
```
ubuntu  ALL=(ALL) NOPASSWD: /usr/sbin/service nginx force-reload
```
这样在执行`sudo service nginx force-reload`的时候就不用输密码了

### 生成DH交换密钥
本人之前配ssl的时候没有这一步，这次在网上发现很多博主说要配这个，网站更加安全。
``` shell
openssl dhparam -out /home/ubuntu/www/ssl/dhparam.pem 2048
```
这一步大概要花比较长的时间，耐心等待一下，具体原因是这样的

> dhparam 算法是在 2^4096 个数字中找出两个质数，所以需要的时间挺长。..... 意思是有可能的质数，+ 是正在测试的质数，* 是已经找到的质数。（引自:[Nginx 配置上更安全的 SSL & ECC 证书 & Chacha20 & Certificate Transparency](https://www.pupboss.com/nginx-add-ssl/)）

### 修改nginx配置
``` config
server {
    ssl_certificate /home/ubuntu/www/ssl/xxx.xxx.com.key.pem
    ssl_certificate_key /home/ubuntu/www/ssl/xxx.xxx.com.key
    ssl_dhparam /home/ubuntu/www/ssl/dhparam.pem
}
```

检查配置后重启
``` shell
sudo service nginx configtest
sudo service nginx restart
```

### 跑个分
[https://ssllabs.com/ssltest/](https://ssllabs.com/ssltest/)(国外的，不能指定端口)

[https://myssl.com/](https://myssl.com/)(国内的，可以指定除443以外的端口)

拿到A+就可以了，你还可以看到具体的评测结果

### 自动更新
这部分在你第一次申请的时候`acme.sh`就帮你做了，用的是`cron`定时计划
``` shell
$ crontab -l
0 0 * * * "/home/ubuntu/.acme.sh"/acme.sh --cron --home "/home/ubuntu/.acme.sh" > /dev/null
```
你可以尝试手动执行一下，在没过期的时候会跳过，你可以指定`--force`参数来强制更新
``` shell
acme.sh --cron --force
```
没问题的话就代表万事大吉了，恭喜你不用管证书的更新问题了

## 后记
你可以配置OCSP装订，也不难，网上教程很多，[Nginx 配置上更安全的 SSL & ECC 证书 & Chacha20 & Certificate Transparency](https://www.pupboss.com/nginx-add-ssl/)里面就有

