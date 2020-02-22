---
layout: post
title:  "jenkins+docker 持续集成"
categories: CI/CD
tags:  Jenkins maven docker
author: Utachi
---

* content
{:toc}

## 一、基本思路
springboot项目做演示

1、项目的部署在Jenkins上执行构建

2、Jenkins自动从github上面拉取代码到服务器

3、maven将项目打成jar包

4、创建个docker镜像，通过docker容器部署项目

## 二、环境搭建
### 1、系统环境
```bash
[root@node0 ~]# cat /etc/redhat-release
CentOS Linux release 7.2.1511 (Core)
```
### 2、Install JDK

* 下载并安装jdk

```bash
wget https://download.oracle.com/otn/java/jdk/8u221-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u221-linux-x64.rpm?AuthParam=1546653417_8f345955cac417a6625ab6e4f27c79f6
rpm -ivh jdk-8u221-linux-x64.rpm
```





* 配置环境变量

```bash
vi ~/.bashrc

export JAVA_HOME='/usr/local/jdk1.8.0_221'
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

source ~/.bashrc
```

* 测试

```bash
[root@sky ~]# java -version
java version "1.8.0_221"
Java(TM) SE Runtime Environment (build 1.8.0_221-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.221-b11, mixed mode)
```

### 3、Install Jenkins

* 官方文档：[https://jenkins.io/zh/doc/book/installing/](https://jenkins.io/zh/doc/book/installing/)介绍了很多种方式，本文以war包方式说明
* 下载war包：[https://mirrors.tuna.tsinghua.edu.cn/jenkins/war](https://mirrors.tuna.tsinghua.edu.cn/jenkins/war)
* 启动：java -jar  jenkins.war(默认端口8080，可通过 ***--httpPort*** 指定端口；
        比如：java -jar jenkins.war --httpPort=8090。
        默认安装目录=$user.home/.jenkins  可通过修改环境变量 ***${JENKINS_HOME}*** 指定webroot)
        

```bash
Running from: /root/jenkins-test/jenkins.war
#当**${JENKINS_HOME}**为空值
#webroot: $user.home/.jenkins
#当**${JENKINS_HOME}**非空
#webroot: EnvVars.masterEnvVars.get("JENKINS_HOME")
Jan 05, 2019 6:18:33 AM org.eclipse.jetty.util.log.Log initialized
INFO: Logging initialized @421ms to org.eclipse.jetty.util.log.JavaUtilLog
......[此处省略]
INFO: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@4e3ecb46: defining beans [filter,legacy]; root of factory hierarchy
Jan 05, 2019 6:18:43 AM jenkins.install.SetupWizard init
INFO: 
*************************************************************
*************************************************************
*************************************************************
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

1e359e8444534a91b4017f53bbbc5a7e

This may also be found at: /root/.jenkins/secrets/initialAdminPassword
*************************************************************
*************************************************************
*************************************************************
Jan 05, 2019 6:19:47 AM hudson.model.UpdateSite updateData

#若长时间卡在这里，需要配置插件反向代理
INFO: Completed initialization
Jan 05, 2019 6:20:03 AM hudson.WebAppMain$3 run
INFO: Jenkins is fully up and running
```
### 4、plugins反向代理
* 修改 **${JENKINS_HOME}**/hudson.model.UpdateCenter.xml
将默认的源替换为https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
*  修改本地hosts解析
```bash
echo '127.0.0.1 mirrors.jenkins-ci.org' >> /etc/hosts
```
*  配置Nginx反向代理
```bash
#编辑/etc/nginx/conf.d/vhost-jenkins.conf
server{
    listen 80;
    server_name mirrors.jenkins-ci.org;

    location / {
        proxy_redirect off;
        proxy_pass https://mirrors.tuna.tsinghua.edu.cn/jenkins/;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Accept-Encoding "";
        proxy_set_header Accept-Language "zh-CN";
    }
    index index.html index.htm;

    #error_page   404   /404.html;
    location ~ /\.
    {
        deny all;
    }
    #access_log   /usr/share/nginx/html/access.log;
    #error_log    /usr/share/nginx/html/error.log;
}
```

