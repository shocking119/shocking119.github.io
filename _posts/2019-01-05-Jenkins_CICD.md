---
layout: post
title:  "jenkins+docker 持续集成"
categories: CI/CD
tags:  Jenkins maven docker
author: Utachi
---

* content
{:toc}



# 一、基本思路
springboot项目做演示

1、项目的部署在Jenkins上执行构建

2、Jenkins自动从github上面拉取代码到服务器

3、maven将项目打成jar包

4、创建个docker镜像，通过docker容器部署项目

# 二、环境搭建

## 1、系统环境
```bash
[root@node0 ~]# cat /etc/redhat-release
CentOS Linux release 7.2.1511 (Core)
```

## 2、Install JDK

* 下载并安装jdk
```bash
wget https://download.oracle.com/otn/java/jdk/8u131-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u131-linux-x64.rpm?AuthParam=1546653417_8f345955cac417a6625ab6e4f27c79f6
rpm -ivh jdk-8u131-linux-x64.rpm
```

* 配置环境变量
```bash
vi ~/.bashrc

export JAVA_HOME=/usr/java/latest
export PATH=$PATH:$JAVA_HOME/bin

source ~/.bashrc
```
* 测试

```bash
[root@sky ~]# java -version
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)
```

## Install Jenkins
* 下载：[http://updates.jenkins-ci.org/download/war/](http://updates.jenkins-ci.org/download/war/)
* 启动：java -jar  jenkins.war(默认端口8080，可通过httpPort指定端口)

```bash
Running from: /root/cicd/jenkins.war
webroot: $user.home/.jenkins
Jan 05, 2019 6:18:33 AM org.eclipse.jetty.util.log.Log initialized
INFO: Logging initialized @421ms to org.eclipse.jetty.util.log.JavaUtilLog
Jan 05, 2019 6:18:33 AM winstone.Logger logInternal
INFO: Beginning extraction from war file
Jan 05, 2019 6:18:35 AM org.eclipse.jetty.server.handler.ContextHandler setContextPath
WARNING: Empty contextPath
Jan 05, 2019 6:18:35 AM org.eclipse.jetty.server.Server doStart
INFO: jetty-9.4.z-SNAPSHOT; built: 2018-06-05T18:24:03.829Z; git: d5fc0523cfa96bfebfbda19606cad384d772f04c; jvm 1.8.0_131-b11
Jan 05, 2019 6:18:36 AM org.eclipse.jetty.webapp.StandardDescriptorProcessor visitServlet
INFO: NO JSP Support for /, did not find org.eclipse.jetty.jsp.JettyJspServlet
Jan 05, 2019 6:18:36 AM org.eclipse.jetty.server.session.DefaultSessionIdManager doStart
INFO: DefaultSessionIdManager workerName=node0
Jan 05, 2019 6:18:36 AM org.eclipse.jetty.server.session.DefaultSessionIdManager doStart
INFO: No SessionScavenger set, using defaults
Jan 05, 2019 6:18:36 AM org.eclipse.jetty.server.session.HouseKeeper startScavenging
INFO: node0 Scavenging every 600000ms
Jenkins home directory: /root/.jenkins found at: $user.home/.jenkins
Jan 05, 2019 6:18:36 AM org.eclipse.jetty.server.handler.ContextHandler doStart
INFO: Started w.@59aa20b3{Jenkins v2.131,/,file:///root/.jenkins/war/,AVAILABLE}{/root/.jenkins/war}
Jan 05, 2019 6:18:36 AM org.eclipse.jetty.server.AbstractConnector doStart
INFO: Started ServerConnector@5c1bd44c{HTTP/1.1,[http/1.1]}{0.0.0.0:8080}
Jan 05, 2019 6:18:36 AM org.eclipse.jetty.server.Server doStart
INFO: Started @3499ms
Jan 05, 2019 6:18:36 AM winstone.Logger logInternal
INFO: Winstone Servlet Engine v4.0 running: controlPort=disabled
Jan 05, 2019 6:18:38 AM jenkins.InitReactorRunner$1 onAttained
INFO: Started initialization
Jan 05, 2019 6:18:38 AM jenkins.InitReactorRunner$1 onAttained
INFO: Listed all plugins
Jan 05, 2019 6:18:40 AM jenkins.InitReactorRunner$1 onAttained
INFO: Prepared all plugins
Jan 05, 2019 6:18:40 AM jenkins.InitReactorRunner$1 onAttained
INFO: Started all plugins
Jan 05, 2019 6:18:40 AM jenkins.InitReactorRunner$1 onAttained
INFO: Augmented all extensions
Jan 05, 2019 6:18:41 AM jenkins.InitReactorRunner$1 onAttained
INFO: Loaded all jobs
Jan 05, 2019 6:18:41 AM hudson.model.AsyncPeriodicWork$1 run
INFO: Started Download metadata
Jan 05, 2019 6:18:42 AM org.springframework.context.support.AbstractApplicationContext prepareRefresh
INFO: Refreshing org.springframework.web.context.support.StaticWebApplicationContext@1fce53d0: display name [Root WebApplicationContext]; startup date [Sat Jan 05 06:18:42 UTC 2019]; root of context hierarchy
Jan 05, 2019 6:18:42 AM org.springframework.context.support.AbstractApplicationContext obtainFreshBeanFactory
INFO: Bean factory for application context [org.springframework.web.context.support.StaticWebApplicationContext@1fce53d0]: org.springframework.beans.factory.support.DefaultListableBeanFactory@c7fb9e1
Jan 05, 2019 6:18:42 AM org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
INFO: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@c7fb9e1: defining beans [authenticationManager]; root of factory hierarchy
Jan 05, 2019 6:18:43 AM org.springframework.context.support.AbstractApplicationContext prepareRefresh
INFO: Refreshing org.springframework.web.context.support.StaticWebApplicationContext@4e5f5340: display name [Root WebApplicationContext]; startup date [Sat Jan 05 06:18:43 UTC 2019]; root of context hierarchy
Jan 05, 2019 6:18:43 AM org.springframework.context.support.AbstractApplicationContext obtainFreshBeanFactory
INFO: Bean factory for application context [org.springframework.web.context.support.StaticWebApplicationContext@4e5f5340]: org.springframework.beans.factory.support.DefaultListableBeanFactory@4e3ecb46
Jan 05, 2019 6:18:43 AM org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
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
INFO: Obtained the latest update center data file for UpdateSource default
Jan 05, 2019 6:19:48 AM hudson.model.DownloadService$Downloadable load
INFO: Obtained the updated data file for hudson.tasks.Maven.MavenInstaller
Jan 05, 2019 6:19:48 AM hudson.model.AsyncPeriodicWork$1 run
INFO: Finished Download metadata. 66,986 ms
Jan 05, 2019 6:20:02 AM hudson.model.UpdateSite updateData
INFO: Obtained the latest update center data file for UpdateSource default
Jan 05, 2019 6:20:03 AM jenkins.InitReactorRunner$1 onAttained
INFO: Completed initialization
Jan 05, 2019 6:20:03 AM hudson.WebAppMain$3 run
INFO: Jenkins is fully up and running
```



