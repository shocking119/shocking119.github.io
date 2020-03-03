---
layout: post
title:  "Jekyll install"
date:   2020-03-02 18:16:03 +0800
categories: jekyll update
---







# Centos升级Ruby版本


在安装jekyll时，所需要的使用ruby工具进行操作，发现在线安装的Ruby版本过低，jekyll支持的版本最少为2.2。

```bash
[root@VM_16_3_centos ~]# gem install jekyll
Fetching: public_suffix-3.0.3.gem (100%)
ERROR:  Error installing jekyll:
        public_suffix requires Ruby version >= 2.1.
#在线安装ruby
yum install -y ruby && ruby -v

[root@VM_16_3_centos ~]# ruby -v
ruby 2.0.0p648 (2015-12-16) [x86_64-linux]
```


* 添加aliyun镜像并检测Ruby版本

```bash
gem sources --remove https://rubygems.org/
gem sources -a http://mirrors.aliyun.com/rubygems/
#gem sources -a https://mirrors.tuna.tsinghua.edu.cn/rubygems/

gem sources -l
```
* 安装RVM
RVM（Ruby Version Manager ）是一款RVM的命令行工具，可以使用RVM轻松安装，管理Ruby版本。RVM包含了Ruby的版本管理和Gem库管理(gemset)
```bash
curl -L https://get.rvm.io | bash -s stable
 ......[部分省略]

    gpg2 --keyserver hkp://pool.sks-keyservers.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
```


* 更新配置文件，使其立马生效：

```bash
source /etc/profile.d/rvm.sh
#查看RVM版本信息，如果可以代表安装成功。
rvm -v
rvm list know

#MRI Rubies
[ruby-]1.8.6[-p420]
[ruby-]1.8.7[-head] # security released on head
[ruby-]1.9.1[-p431]
[ruby-]1.9.2[-p330]
[ruby-]1.9.3[-p551]
[ruby-]2.0.0[-p648]
[ruby-]2.1[.10]
[ruby-]2.2[.10]
[ruby-]2.3[.7]
[ruby-]2.4[.4]
[ruby-]2.5[.1]
[ruby-]2.6[.3]
ruby-head
......[此处省略]

rvm install 2.5

ruby -v
```

