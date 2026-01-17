---
title: HTB-OutBound WriteUp
published: 2025-08-09
description: HackTheBox OutBound 通关记录。
image: http://imgur.41an.com/picgo/2025-08/htb-outbound-1754754068049.png
tags:
  - Penetration
  - PrivEsc
  - Linux
category: 打靶日记
draft: false
lang: zh_CN
---
## 1. Recon
`nmap` 收集一下端口信息。
```shell
nmap -sT --min-rate=10000 -iL 10.10.11.77 -p- -Pn 
...
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

访问 80 端口，发现使用了 `webmail` 框架。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-80-1754752065491.png)
## 2. Foothold
搜索发现`webmail`存在 [CVE-2025-49113](https://github.com/hakaioffsec/CVE-2025-49113-exploit) 漏洞。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-exp-1754752047524.png)
反弹 shell。
![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-nc-1754752032366.png)
翻找roundcube的配置文件，发现数据库凭证。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-config-1754752005317.png)
拖库，并将文件传至本地，导入mysql分析。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-dump-1754752188317.png)
发现 sesion 表下有段可疑数据。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-session-1754752234129.png)
将其以 `base64` 格式解码，发现一个加密后了的口令`L7Rv00A8TuwJAr67kITxxcSgnIk25Am/`。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-base64-1754752296262.png)
结合之前在 `./config.inc.php` 发现的 `$config[‘des_key’]` 字段，判断其可能为DES 加密。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-key-1754752453784.png)
但直接使用DES无法解密，搜索一下他的默认加密方式，可见是3DES-CBC。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-3des-1754752421597.png)
DES格式的密文前面8个字节即为IV，将其转为hex格式提取出来。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-iv-1754752520653.png)
解密，得到口令 `595mO8DmwGeD`。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-pwd-1754752542646.png)
使用此凭证再次登录邮箱，发现口令泄露。可以直接用于ssh 登录。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-login-1754752566718.png)
## 3. PrivEsc
`sudo -l` 发现可以 `root` 身份运行 `/usr/bin/below`。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-sudo-l-1754752642492.png)
google 发现 `below` 存在一个提权漏洞 [CVE-2025-27591](https://github.com/BridgerAlderson/CVE-2025-27591-PoC)，小脚本一跑，提权成功。

![image.png](http://imgur.41an.com/picgo/2025-08/htb-outbound-webmail-priv-1754752663434.png)