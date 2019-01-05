---
layout: post
title:  "主流ssl证书配置"
categories: Linux
tags:  ssl https
author: Utachi
---

* content
{:toc}

## Nginx

1.打开 Nginx 安装目录下 conf 目录，将下载好的证书解压；

2.修改nginx.conf 文件，找到server定义，或者单独include一个conf，填写vhost。 

``` bash
    server {
        listen 443;
        server_name localhost;
        #localhost 设置成你的主机名
        ssl on;
        root html;
        index index.html index.htm;
        ssl_certificate   cert/1536195675718.pem;
        #这里是证书路径，根据你上传的位置修改
        ssl_certificate_key  cert/1536195675718.key;
        #同上
        ssl_session_timeout 5m;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        location / {
            root html;
            index index.html index.htm;
            }
        }
```
3.重载conf文件：nginx -s reload**

## Apache

1.打开 Apache 安装目录下 conf 目录，将下载好的证书解压；

修改httpd.conf 文件，找到以下内容并去掉“#”

```bash
#LoadModule ssl_module modules/mod_ssl.so (如果找不到请确认是否编译过 openssl 插件)
#Include conf/extra/httpd-ssl.conf
```

2.打开 apache 安装目录下 conf/extra/httpd-ssl.conf 文件 (也可能是conf.d/ssl.conf，与操作系统及安装方式有关)， 在配置文件中查找以下配置语句:

```bash
# 添加 SSL 协议支持协议，去掉不安全的协议
SSLProtocol all -SSLv2 -SSLv3
# 修改加密套件如下
SSLCipherSuite HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM
SSLHonorCipherOrder on
# 证书公钥配置
SSLCertificateFile cert/public.pem
# 证书私钥配置
SSLCertificateKeyFile cert/1536195675718.key
# 证书链配置，如果该属性开头有 '#'字符，请删除掉
SSLCertificateChainFile cert/chain.pem
```

3.重启 Apache


## Tomcat

Tomcat支持JKS格式证书，从Tomcat7开始也支持PFX格式证书，两种证书格式任选其一

1.证书格式转换
 
在Tomcat的安装目录下创建cert目录，并且将下载的全部文件拷贝到cert目录中。如果申请证书时是自己创建的CSR文件，附件中只包含1536195675718.pem文件，还需要将私钥文件拷贝到cert目录，命名为1536195675718.key；如果是系统创建的CSR，请直接到第2步。

到cert目录下执行如下命令完成PFX格式转换命令

```markdown
openssl pkcs12 -export -out 1536195675718.pfx -inkey 1536195675718.key -in 1536195675718.pem
```    
此处要设置PFX证书密码，请牢记

2.PFX证书安装

找到安装Tomcat目录下该文件server.xml,一般默认路径都是在 conf 文件夹中。找到 <Connection port="8443"标签，增加如下属性：

```markdown
keystoreFile="cert/1536195675718.pfx"
keystoreType="PKCS12"
keystorePass="证书密码"             #此处的证书密码，请参考附件中的密码文件或在第1步中设置的密码
```

完整的配置如下，其中port属性根据实际情况修改：

```markdown
 <Connector port="8443"
            protocol="HTTP/1.1"
            SSLEnabled="true"
            scheme="https"
            secure="true"
            keystoreFile="cert/1536195675718.pfx"
            keystoreType="PKCS12"
            keystorePass="之前的PFX证书密码"
            clientAuth="false"
            SSLProtocol="TLSv1+TLSv1.1+TLSv1.2"
            ciphers="TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA256"
        />
```

注意:不要直接拷贝所有配置，只需添加 keystoreFile,keystorePass等参数即可，其它参数请根据自己的实际情况修改

3.重启Tomcat

## IIS
* IIS 6 
    * 证书导入

        • 开始 ->运行 ->MMC；
        
        • 启动控制台程序，选择菜单“文件"中的"添加/删除管理单元”-> “添加”，从“可用的独立管理单元”列表中选择“证书”-> 选择“计算机帐户”；
        
        • 在控制台的左侧显示证书树形列表，选择“个人”->“证书”，右键单击，选择“所有任务"-〉"导入”,根据"证书导入向导”的提示，导入PFX文件（此过程当中有一步非常重要： “根据证书内容自动选择存储区”）。
        
        • 分配服务器证书
    
    *  目录安全性->服务器证书->分配现有证书->443端口
* IIS 7/8
	* 证书导入
		• 开始 ->运行 ->MMC；
		
		• 启动控制台程序，选择菜单“文件"中的"添加/删除管理单元”-> “添加”，从“可用的独立管理单元”列表中选择“证书”-> 选择“计算机帐户”；
		
		• 在控制台的左侧显示证书树形列表，选择“个人”->“证书”，右键单击，选择“所有任务"-〉"导入”,根据"证书导入向导”的提示，导入PFX文件（此过程当中有一步非常重要： “根据证书内容自动选择存储区”）。安装过程当中需要输入密码为您当时设置的密码。导入成功后,可以看到如图所示的证书信息。
		
	* 分配服务器证书,打开 IIS8.0 管理器面板,找到待部署证书的站点,点击“绑定”。
	
		• 设置参数
		
		• 选择“绑定”->“添加”->“类型选择 https” ->“端口 443” ->“ssl 证书【导入的证书名称】” ->“确定”,SSL 缺省端口为 443 端口(请不要随便修改。 如果您使用其他端口如:8443,则访问时必须输入:https://www.domain.com:8443)。
		
	* 其他windows 配置SSL证书的问题，请参考视频：
		[https://help.aliyun.com/video_detail/54215.html?spm=5176.2020520163.cas.158.21ea2b7aWb5BAu](https://help.aliyun.com/video_detail/54215.html?spm=5176.2020520163.cas.158.21ea2b7aWb5BAu)


