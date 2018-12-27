---
layout: post
title:  " oracle 11gR2静默安装"
categories: Linux
tags:  Oracle
author: Utachi
---

* content
{:toc}



# 一.安装前准备
   1.内存及swap要求
     至于swap如何添加，后文将提到
   ```
        grep MemTotal /proc/meminfo
        grep SwapTotal /proc/meminfo
   ```
   2.硬盘空间
   ```
        df -h
   ```
   3.修改主机名，及ip对应关系
   --设置主机名，也可以直接修改配置文件/etc/sysconfig/network，不过这个是重启之后才起作用
   ```commandline
    hostnamectl set-hostname DB_2
   ```   
   4.关闭Selinux
   ```
    # sed -i "s/SELINUX=enforcing/SELINUX=disabled/"/etc/selinux/config  
    # setenforce 0
   ```
   5.下载oracle11gR2
   
   官网下载地址：[http://www.oracle.com/technetwork/database/enterprise-edition/downloads/112010-linx8664soft-100572.html]

# 二.修改内核参数
    
   1./etc/sysctl.conf 
     --修改或添加，具体参数意思，请百度或参考oracle官网解释
     #vim  /etc/sysctl.conf  
   ```bash
    net.ipv4.ip_local_port_range= 9000 65500 
    fs.file-max = 6815744 
    kernel.shmall = 10523004 
    kernel.shmmax = 6465333657 
    kernel.shmmni = 4096 
    kernel.sem = 250 32000 100128 
    net.core.rmem_default=262144 
    net.core.wmem_default=262144 
    net.core.rmem_max=4194304 
    net.core.wmem_max=1048576 
    fs.aio-max-nr = 1048576
    # sysctl -p  #使配置生效
   ```
   2.用户的限制文件/etc/security/limits.conf 
   ```buildoutcfg
    #vim /etc/security/limits.conf 在文件后增加
     oracle           soft    nproc           2047
     oracle           hard    nproc           16384
     oracle           soft    nofile          1024
     oracle           hard    nofile          65536
     oracle           soft    stack           10240
   ```
    
   --修改/etc/pam.d/login文件，增加如下：
   ```commandline
   session  required   /lib64/security/pam_limits.so  
                        #64位系统，千万别写成/lib/security/pam_limits.so，否则导致无法登录
   session  required   pam_limits.so

   ```
 
# 三.创建用户及组

* 创建用户及组
    ```commandline
    #groupadd oinstall 
    #groupadd dba
    #useradd -g oinstall -G dba -d /home/u11 oracle
    #passwd oracle
    ```


* 创建安装目录
    ```
    mkdir -p /opt/app/oracle/product/11.2.0/dbhome_1
    ```
* 数据文件存放目录
    ```commandline
    #mkdir -p /opt/app/oracle/oradata
    ```
* 数据恢复目录
    ```commandline
    #mkdir -p /opt/app/oracle/recovery_area
    ```

* 数据库创建及使用过程中的日志目录
    ```commandline
    #mkdir -p /opt/app/oracle/oraInventory
    ```

* 修改安装目录权限
    ```commandline
    #chown -R oracle:oinstall /opt/app/oracle
    #chmod 775 /opt/app/oracle
    ```

* 登录oracle用户，设置环境变量
    ```commandline
    #su - oracle
    $view .bash_profile
    　　export ORACLE_BASE=/opt/app/oracle
    　　export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1
    　　export PATH=$PATH:$ORACLE_HOME/bin
    　　export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
    　　export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib64:/usr/lib64:/usr/local/lib64
    　　export ORACLE_SID=wetalk
    　　      #如果设置NLS_LANG，容易产生导入sql或dmp出错，因为其他环境下的不是utf8
    　　export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
    $source .bash_profile   
              #使设置生效
    ```
    
#  四.安装oracle
* 1.安装依赖包
    ```commandline
    # yum -y install gcc gcc-c++ make binutilscompat-libstdc++-33 elfutils-libelf elfutils-libelf-develglibc glibc-commonglibc-devel libaio libaio-devel libgcclibstdc++libstdc++-devel unixODBC unixODBC-devel ksh
    或
    # yum -y installbinutils compat-libstdc++-33 compat-libstdc++-33.i686 
    elfutils-libelfelfutils-libelf-devel gcc gcc-c++ glibc glibc.i686 glibc-common 
    glibc-devel glibc-devel.i686glibc-headers ksh libaio libaio.i686 libaio-devel 
    libaio-devel.i686 libgcclibgcc.i686 libstdc++ libstdc++.i686 libstdc++-devel make 
    sysstat unixODBC unixODBC-devel
    ```
* 2.解压安装包
    ```commandline
    tar zxvf ***.bz2
    ```
* 3.数据库安装

    db_install.rsp 安装应答配置文件;
    
    dbca.rsp 创建数据库应答;
    
    netca.rsp 建立监听、本地服务名等网络设置应答
    
* 4.修改配置文件db_install.rsp，并安装
  [Oracle 11gR2 db_install.rsp详解](http://www.cnblogs.com/yingsong/p/6031452.html)
    ```commandline
    oracle.install.option=INSTALL_DB_SWONLY
    ORACLE_HOSTNAME=DB_m2
    UNIX_GROUP_NAME=oinstall
    INVENTORY_LOCATION=/opt/app/oracle/oraInventory
    SELECTED_LANGUAGES=en,zh_CN
    ORACLE_HOME=/opt/app/oracle/product/11.2.0/db_1
    ORACLE_BASE=/opt/app/oracle
    oracle.install.db.InstallEdition=EE
    oracle.install.db.DBA_GROUP=dba
    oracle.install.db.OPER_GROUP=oinstall
    oracle.install.db.config.starterdb.characterSet=AL32UTF8
    oracle.install.db.config.starterdb.storageType=FILE_SYSTEM_STORAGE
    oracle.install.db.config.starterdb.fileSystemStorage.dataLocation=/opt/app/oracle/oradata
    oracle.install.db.config.starterdb.fileSystemStorage.recoveryLocation=/opt/app/oracle/recovery_data
    DECLINE_SECURITY_UPDATES=true    //一定要设为true
    ```
* 5.登录oracle用户，执行安装

    ```commandline
      $./runInstaller-silent -responseFile /home/u11/database/response/db_install.rsp 
    ```    
    安装过程中，如果提示[WARNING]不必理会，此时安装程序仍在进行，如果出现[FATAL]，则安装程序已经停止了。
    打开另一个终端，执行命令
    ```commandline
      #tail -100 f /u01/app/oracle/oraInventory/logs/installActions......log
    ```
    可以实时跟踪查看安装日志，了解安装的进度。
    当出现以下配置脚本需要以 "root" 用户的身份执行。
 
    ```commandline
      /u01/app/oracle/oraInventory/orainstRoot.sh
      /u01/app/oracle/product/11.2.0/db_1/root.sh
    
    要执行配置脚本, 请执行以下操作:
        
       1. 打开一个终端窗口
         
       2. 以 "root" 身份登录
         
       3. 运行脚本
         
       4. 返回此窗口并按 "Enter" 键继续
         
          Successfully Setup Software.
   ```  
   出现这个的话，说明已安装成功，则需要按提示操作，操作完返回Enter成功
* 6.配置监听配置文件response/netca.rsp
    ```commandline
    $netca /silent /responsefile response/netca.rsp
    
    正在对命令行参数进行语法分析:
     参数"silent" = true
     参数"responsefile" = /home/oracle/response/netca.rsp
     完成对命令行参数进行语法分析。
     Oracle Net Services 配置:
     完成概要文件配置。
     Oracle Net 监听程序启动:
      正在运行监听程序控制:
        /opt/oracle/11.2.0/bin/lsnrctl start LISTENER
      监听程序控制完成。
      监听程序已成功启动。
     监听程序配置完成。
     成功完成 Oracle Net Services 配置
    ```    
    成功运行后，在/opt/oracle/11.2.0/network/admin目录下生成sqlnet.ora和listener.ora两个文件。
    
    完成后通过命令“netstat -tlnp”可以查看到1521端口已开
    ```commandline
       tcp  0   0 :::1521        :::*      LISTEN      5477/tnslsnr
    ```
    
* 7.修改配置文件response/dbca.rsp，静默建立新库
   
    ```
     RESPONSEFILE_VERSION = "11.2.0"  //不能更改
     OPERATION_TYPE = "createDatabase"
     GDBNAME = "utachi.cn"  //全局数据库的名字=SID+主机域名
     SID = "hello"    //对应的实例名字
     TEMPLATENAME = "General_Purpose.dbc" //建库用的模板文件
     DATAFILEDESTINATION = /opt/oracle/oradata //数据文件存放目录
     RECOVERYAREADESTINATION=/opt/oracle/recovery_data //恢复数据存放目录
     CHARACTERSET = "AL32UTF8"   //字符集，重要!!! 建库后一般不能更改，所以建库前要确定清楚。
     TOTALMEMORY = "5120"    //oracle内存5120MB   
    ```
    配置完之后，执行命令:
    ```commandline
     
     $dbca -silent -responseFile /etc/dbca.rsp
    
     37% 已完成
     正在创建并启动 Oracle 实例
     62% 已完成
     正在进行数据库创建
     66% 已完成
     100% 已完成
     有关详细信息, 请参阅日志文件 "/u01/app/oracle/cfgtoollogs/dbca/Utachi/Utachi.log"。
     
     查看日志文件
     $ cat /u01/app/oracle/cfgtoollogs/dbca/Utachi/Utachi.log
    ```
# 五. 开启归档模式，制定归档目录
　　[开启归档模式，归档日志已满处理](http://www.cnblogs.com/yingsong/p/6037531.html)