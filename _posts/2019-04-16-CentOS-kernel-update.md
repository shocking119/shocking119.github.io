---
layout: post
title:  "CentOS 7 源码升级内核"
categories: Linux
tags:  linux kernel kernel-4.18.4 centos7  
author: Utachi
---

* content
{:toc}

系统环境：CentOS Linux release 7.2.1511 (Core)最小化安装

# 依赖准备
为提升下载速度可以配置本地源，配置文件在我的[**另一篇文档**](https://utachi.cn/2019/04/04/local-yum-createrepo/)
````bash
yum install -y gcc make git ctags ncurses-devel openssl-devel bison flex elfutils-libelf-devel bc
````

# 获取内核

Warning：如果选择default配置，内核所在目录至少5G空间用于存放make过程中的临时文件
````bash
wget https://mirrors.aliyun.com/linux-kernel/v4.x/linux-4.18.4.tar.gz

tar zxvf  linux-4.18.4.tar.gz
cd  linux-4.18.4
````
确保内核编译目录树干净，执行

````bash
make clean && make mrproper
````
# 内核配置

* 方法1：复制当前的内核配置文件
````bash
cp /boot/config-3.10.0-327.13.1.el7.x86_64 .config
sh -c ' echo "\r" | make oldconfig '
````
* 方法2：自定义选单生成
````bash
#待编译目录下执行
touch .config && make localmodconfig
make menuconfig
````
如果不使用"touch .config && make localmodconfig"则引用了boot目录下默认的配置文件作为默认选项，个别字段会有增删
**高级配置**

y 是启用, n 是禁用, m 是需要时启用. 

make nconfig ：传统命令行遍历所有选单

make menuconfig ：半图形化显示，支持字母快速选单

# 编译和安装
````bash
make -j [N]       #N=cpu线程数-1            可以写成    --jobs[=N]    
                  #Allow N jobs at once; infinite jobs with no arg.
````

Warning: 从这里开始，必须 root 权限执行命令，否则会失败. 

````bash
 make modules_install install && make clean
````

# 修改grub启动器
````bash
grub2-mkconfig -o /boot/grub2/grub.cfg
awk -F \' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
    0 : CentOS Linux (4.18.4) 7 (Core)
    1 : CentOS Linux (3.10.0-693.el7.x86_64) 7 (Core)
    2 : CentOS Linux (0-rescue-a6c51d4e386e4d5e9bf6422748d53480) 7 (Core)


grub2-set-default 0                 
                  #设置默认启动项。
grub2-editenv list

````

整理一下：
````bash
cd /home
yum install -y wget vim gcc make git ctags ncurses-devel openssl-devel bison flex elfutils-libelf-devel bc
wget https://mirrors.aliyun.com/linux-kernel/v4.x/linux-4.18.4.tar.gz
tar zxvf  linux-4.18.4.tar.gz
cd  linux-4.18.4
make clean && make mrproper
make menuconfig && cp .config /boot/config-4.18.4
make -j 4
make modules_install install 
grub2-mkconfig -o /boot/grub2/grub.cfg && grub2-set-default 0
cd && rm -rf /home/linux-4.18.4
````
