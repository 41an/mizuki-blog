---
title: HTB-soulmate WriteUp
published: 2025-10-15
description: HackTheBox soulmate 通关记录。
image: http://imgur.41an.com/picgo/2025-10/htb-soulmate-1760543558844.png
tags:
  - Penetration
  - PrivEsc
  - Linux
category: 打靶日记
draft: false
lang: zh_CN
---
## 1. Recon
```shell
nmap -sS -Pn -n --open --min-hostgroup 4 --min-parallelism 1024 --host-timeout 30 -T4 -iL ip.txt  -oA nmapscan/syn
...
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.88 seconds
```
访问 `http://10.10.11.86`，提示需要解析域名
```shell
echo '10.10.11.86 soulmate.htb' >> /etc/hosts
```
搜索一圈没有可利用的信息，爆破子域
```
ffuf -u http://10.10.11.86 -H "Host:FUZZ.soulmate.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 154 -t 500

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

...

ftp                     [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 468ms]
```
加域名加入`hosts`文件中解析
```shell
echo '10.10.11.86 ftp.soulmate.htb' >> /etc/hosts
```
访问 `ftp.soulmate.htb`

![htb-soulmate-crushftp.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-crushftp-1760543138168.png)
## 2. Foothold
搜索`CurshFtp vuln`发现`CVE-2025-54309`，`POC`脚本也提示成功，但尝试多次利用都无法成功...
[The One Where We Just Steal The Vulnerabilities (CrushFTPCVE−2025−54309CrushFTP CVE-2025-54309)](https://labs.watchtowr.com/the-one-where-we-just-steal-the-vulnerabilities-crushftp-cve-2025-54309/)

![htb-soulmate-CVE-2025-54309.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-CVE-2025-54309-1760543201952.png)
最终发现 [CVE-2025-31161](https://github.com/Immersive-Labs-Sec/CVE-2025-31161) 可用

![htb-soulmate-CVE-2025-31161.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-CVE-2025-31161-1760543222188.png)
使用给出的凭证登录，可修改 `ben` 用户的口令

![htb-soulmate-chpwd.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-chpwd-1760543238698.png)
登录 `ben`，可在网站目录下上传 `反弹shell`

![htb-soulmate-rev.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-rev-1760543256611.png)
接收 `反弹shell`

![htb-soulmate-nc.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-nc-1760543272597.png)
在 `config` 目录下发现数据库文件

![htb-soulmate-config.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-config-1760543285923.png)
将`soulmate.db`转成`base64`格式复制到本地解码
```shell
base64 /var/www/soulmate.htb/data/soulmate.db

echo "your_base64_code" | base64 -d > soulmate.db
```
读取配置信息
```shell
$ sqlite3 soulmate.db
SQLite version 3.46.1 2024-08-13 09:16:08
Enter ".help" for usage hints.
sqlite> .tables
users
sqlite> select * from users;
1|admin|$2y$12$u0AC6fpQu0MJt7uJ80tM.Oh4lEmCMgvBs3PwNNZIR7lor05ING3v2|1|Administrator|||||2025-08-10 13:00:08|2025-08-10 12:59:39
```
可见是`bcrypt`加密，但爆破无果..
继续搜集信息，`ps -aux` 查看进程时发现可疑文件

![htb-soulmate-ps.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-ps-1760543339426.png)
`cat /usr/local/lib/erlang_login/start.escript` 可见 `ben` 的口令

![htb-soulmate-start.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-start-1760543352684.png)
登录 `ben/HouseH0ldings998`

![htb-soulmate-22.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-22-1760543367832.png)
## 3. PrivEsc
查看`ben`用户家目录，

![htb-soulmate-home.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-home-1760543393798.png)
`netstat -antup`查看端口开放情况时发现有个`2222`端口

![](http://imgur.41an.com/picgo/2025-10/htb-soulmate-netstat-1760543424202.png)
尝试连接

![htb-soulmate-2222.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-2222-1760543454053.png)
`Erlang SSH` 加载 `os` 模块直接提权

![htb-soulmate-priv.png](http://imgur.41an.com/picgo/2025-10/htb-soulmate-priv-1760543487825.png)

---