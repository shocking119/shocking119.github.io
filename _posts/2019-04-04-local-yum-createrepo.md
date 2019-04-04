---
layout: post
title:  "基于rpm包制作本地yum仓库"
categories: Linux
tags:  linux yum 
author: Utachi
---

* content
{:toc}

我们可以将较常使用的rpm安装包归到一个文件里面制作成一个可以被系统识别的yum仓库，
通过配置yum仓库指向文件可以将它设置成本地的yum源也可以是通过http发布的共享yum源。

# 一. 用downloadonly下载

##1.1  处理依赖关系

自动下载到/localrepo目录，pages这个目录会自动创建
yum install --downloadonly --downloaddir=/localrepo  ceph-deploy

注意，如果下载的包包含了任何没有满足的依赖关系，yum将会把所有的依赖关系包下载，但是都不会被安装。


# 二、不使用downloadonly ，自动安装或升级的同时保留RPM包
yum 默认情况下，升级或者安装后，会删除下载的rpm包。
不过，我们也可以如下设置升级后不删除下载的rpm包

\# vim /etc/yum.conf

```bash
[main]
cachedir=/localrepo
keepcache=0
```
将 keepcache=0 修改为 keepcache=1， 安装或者升级后，在目录 /var/cache/yum 下就会有下载的 rpm 包了。






# 三. 利用rpm安装包文件进行自己的yum仓库的制作

## 3.1. 首先将rpm包放在一个文件夹里面。

### 3.1.1 安装createrepo命令
```bash
 yum install createrepo

 #或者
 rpm -ivh libxml2-python-2.9.1-6.el7_2.3.x86_64.rpm 
 rpm -ivh deltarpm-3.6-3.el7.x86_64.rpm 
 rpm -ivh python-deltarpm-3.6-3.el7.x86_64.rpm 
 rpm -Uvh createrepo-0.9.9-28.el7.noarch.rpm 
```
### 3.1.2 生成符合要求的yum仓库

执行createrepo /localrepo 
将放置rpm安装包的文件夹创造成一个仓库文件,文件夹里面会多出一个repodata仓库数据文件夹。


### 3.1.3可以看到多了一个repodata的仓库数据文件，此时创建库成功。


如果添加或者删除了个人的rpm包，不需要再次重新create，只需要--update就可以了

createrepo --update  ./

# 4.创建新的repo

\# vim /etc/yum.repos.d/CentOS-Media.repo

````bash
[local-repo]
name=loalrepo - Local
baseurl=file:///localrepo
gpgcheck=0
enabled=1
````
更新cache
````bash
yum clean all && yum makecache
````
#附录
## 阿里云yum源

#aliyun base

[base]

name=CentOS-$releasever - Base - mirrors.aliyun.com

failovermethod=priority

baseurl=http://mirrors.aliyun.com/centos/$releasever/os/$basearch/

gpgcheck=0

enabled=1

#released updates

[updates]

name=CentOS-$releasever - Updates -mirrors.aliyun.com

failovermethod=priority

baseurl=http://mirrors.aliyun.com/centos/$releasever/updates/$basearch/

enabled=0

gpgcheck=0

[extras]

name=CentOS-$releasever - Extras - mirrors.aliyun.com

failovermethod=priority

baseurl=http://mirrors.aliyun.com/centos/$releasever/extras/$basearch/

enabled=0

gpgcheck=0

[epel]

name=Extra Packages for Enterprise Linux $releasever - $basearch - mirrors.aliyun.com

baseurl=http://mirrors.aliyun.com/epel/$releasever/$basearch

failovermethod=priority

gpgcheck=0

enabled=1

[qemu-ev]

name=CentOS-$releasever - QEMU EV

baseurl=http://mirrors.aliyun.com/centos/$releasever/virt/$basearch/kvm-common/

gpgcheck=0

enabled=1