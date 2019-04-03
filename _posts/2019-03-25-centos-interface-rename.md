---
layout: post
title:  "CentOS 7网卡ethx格式命名"
categories: Linux
tags:  linux 
author: Utachi
---

* content
{:toc}

## 一、centos7网卡命名规则
规则1：
对于板载设备命名合并固件或 BIOS 提供的索引号，如果来自固件或 BIOS 的信息可读就命名，比如eno1，这种命名是比较常见的，否则使用规则2。

规则2：命名合并固件或 BIOS 提供的 PCI-E 热插拔口索引号，比如 ens1，如果信息可读就使用，否则使用规则3。

规则3：命名合并硬件接口的物理位置，比如 enp2s0，可用就命名，失败直接到方案5。

规则4：命名合并接口的 MAC 地址，比如 enx78e7d1ea46da，默认不使用，除非用户选择使用此方案。

规则5：使用传统的方案，如果所有的方案都失败，使用类似 eth0 这样的样式。

## 二、修改网卡名称样式为ethx

### 1、编辑 grub 配置文件

vim /etc/sysconfig/grub   # 其实是/etc/default/grub的软连接
为GRUB_CMDLINE_LINUX变量增加2个参数，net.ifnames=0 biosdevname=0：
```bash
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=cl/root rd.lvm.lv=cl/swap net.ifnames=0 biosdevname=0 rhgb quiet"
```
### 2、重新生成 grub 配置文件
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```
### 3、修改网卡配置文件

原来网卡配置文件名称为 ifcfg-ens33，这里需要修改为 ethx 的格式，并适当调整网卡配置文件。
```bash
mv /etc/sysconfig/network-scripts/ifcfg-{ens33,eth0}
# 修改ifcfg-eth0文件如下内容(其它内容不变)
NAME=eth0
DEVICE=eth0
```
然后重新启动 Linux 操作系统，通过 ip addr 可以看到网卡名称已经变为 eth0 。
注意：ifcfg-ens33 文件最好删除掉，否则重启 network 服务时候会报错。