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

Warning：如果选择default配置，内核所在目录至少5G空间
````bash
wget https://mirrors.aliyun.com/linux-kernel/v4.x/linux-4.18.4.tar.gz

tar zxvf  linux-4.18.4.tar.gz
cd  linux-4.18.4
````
确保内核树干净，执行

````bash
make clean && make mrproper
````
# 内核配置

## 复制当前的内核配置文件
````bash
cp /boot/config-3.10.0-327.13.1.el7.x86_64 .config
````

## 高级配置
y 是启用, n 是禁用, m 是需要时启用. 

make nconfig: 新的命令行 ncurses 界面，以前是menuconfig。

# 编译和安装
````bash
make -j [N]       #如果是四核的机器，N可以是8;-j [N], --jobs[=N]    
                  #Allow N jobs at once; infinite jobs with no arg.
````

编译完内核后安装

Warning: 从这里开始，必须 root 权限执行命令，否则会失败. 

````bash
 make modules_install install
````

# 修改grub启动器
````bash
grub2-mkconfig -o /boot/grub2/grub.cfg
````
