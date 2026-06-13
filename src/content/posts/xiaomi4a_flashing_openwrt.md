---
title: 小米4a千兆路由器刷入OpenWrt
published: 2026-06-14
description: ""
image: http://imgur.41an.com/piclist/2026-06/openwrt-sys-9950bb6f7e8843121a73db0706899a98.png
tags:
  - dom/ops
category: 运维日记
draft: false
lang: zh_CN
---
## 1. 开启 Telnet
> 可以开启 Telnet 的原理是因为 `2.28.62` 版本存在 `RCE`
### 1.1. 升/降级
下载 [miwifi_r4a_firmware_72d65_2.28.62.bin](https://raw.githubusercontent.com/acecilia/OpenWRTInvasion/master/firmwares/stock/miwifi_r4a_firmware_72d65_2.28.62.bin) 后，将其传至管理界面更新即可。
![image.png](http://imgur.41an.com/piclist/2026-06/openwrt-3ce1e161086e2418b16ce0029353c4cb.png)
### 1.2. 开刷
顺利运行 `remote_command_execution_vulnerability.py` **需满足**：
1. 版本为 `2.28.62`
2. 需要通互联网
3. 执行脚本的机器与路由器需要处于同一网段
```shell
git clone https://github.com/acecilia/OpenWRTInvasion
cd OpenWRTInvasion
pip install -r ./requirements.txt
python3 ./remote_command_execution_vulnerability.py 
```
> 有些原生的 python 环境不允许额外装包，可以先开一个虚拟环境再下载
> 1. `python -m venv venv`
> 2. `source venv/bin/activate`

运行脚本后，按照提示输入 IP 、口令、stok 值（管理界面的 url 处可获得），提示 done 即表示成功，默认账户口令为 `root/root`。
## 2. 刷入 Breed
> 为防止误操作变砖需先刷入 Breed
### 2.1. 备份
同网段的设备 telnet 上路由器后开始备份
```shell
┌──(root㉿kali)  
└─#telnet {route_ip}

root@XiaoQiang:/# mdkir /tmp/backup
root@XiaoQiang:/# cd /tmp/backup
dd if=/dev/mtd0 of=/tmp/backup/mtd0-ALL.bin
dd if=/dev/mtd1 of=/tmp/backup/mtd1-Bootloader.bin
dd if=/dev/mtd2 of=/tmp/backup/mtd2-Config.bin
dd if=/dev/mtd3 of=/tmp/backup/mtd3-Bdata.bin
dd if=/dev/mtd4 of=/tmp/backup/mtd4-Factory.bin
dd if=/dev/mtd5 of=/tmp/backup/mtd5-crash.bin
dd if=/dev/mtd6 of=/tmp/backup/mtd6-cfg_bak.bin
dd if=/dev/mtd7 of=/tmp/backup/mtd7-overlay.bin
dd if=/dev/mtd8 of=/tmp/backup/mtd8-OS1.bin
dd if=/dev/mtd9 of=/tmp/backup/mtd9-rootfs.bin
dd if=/dev/mtd10 of=/tmp/backup/mtd10-disk.bin
```
> **Factory. bin**是硬件厂商专为该硬件额外优化信号的重要配置文件

在 pc 上开启 9999 端口
```shell
┌──(root㉿kali)
└─# nc -lvnp 9999 > backup.tar
```
用 nc 命令将备份文件传回 pc
```shell
root@XiaoQiang:/tmp/backup#
tar cf - backup | nc {pc_ip} 9999
```
### 2.2. 开刷
```shell
mkdir /tmp/flashing  
cd /tmp/flashing
```
小米 4 a 千兆版的芯片是 `mt7621`，下载与其适配的[固件](https://breed.hackpascal.net/breed-mt7621-pbr-m1.bin)：
```shell
wget https://breed.hackpascal.net/breed-mt7621-pbr-m1.bin
```
验证一下完整性
```shell
md5sum breed-mt7621-pbr-m1.bin 
24e62762809c15ba3872e610a37451a3  breed-mt7621-pbr-m1.bin
```
刷入
```shell
mtd -r write /tmp/flashing/breed-mt7621-pbr-m1.bin Bootloader
```
等待一会后，可以将路由器的 `WAN` 口与主机相连接，访问 `192.168.1.1` 或者自己另外配 IP 再去访问
> 如果提示 `请求的页面不存在`，换个 `IE` 内核的浏览器就行了

![image.png](http://imgur.41an.com/piclist/2026-06/openwrt-breed-error-d5e3e0dd388227a2b1fd62a197a057ee.png)
### 2.3. 固件备份
将 `eeprom` 与编程器固件进行备份
![image.png](http://imgur.41an.com/piclist/2026-06/openwrt-breed-backup-2a0790b59dcef1c77b725505d9dfd383.png)
## 3. 刷入 openwrt
> Breed 的默认引导地址并不是从 `4a` 的 `0x180000` 开始写的，因此使用 GUI 更新固件会导致设备一直重启

这里使用 [25.12.4](https://downloads.openwrt.org/releases/25.12.4/targets/ramips/mt7621/openwrt-25.12.4-ramips-mt7621-xiaomi_mi-router-4a-gigabit-squashfs-sysupgrade.bin) 版本的 openwrt
```
breed> wget https://{pc_ip}/25.12.4.bin
...
Length: 7733838/0x76024e (7MB) [] Saving to address 0x80001000 
...
```
可见固件大小为 `0x76024e`，需要擦除出一个比 `0x76024e` 大的空间，这里凑个整选取 `0x800000`
>  `4a` 的固件引导地址是从 `0x180000` 开始的，所以需要从 `0x180000` 开始
```shell
breed> flash erase 0x180000 0x800000
```
写入
```shell
breed> flash write 0x180000 0x80001000 0x76024e
```
从 `0x180000` 处启动
```
breed> boot flash 0x180000 
```
> 如果之前是选择接入 `WAN` 口访问的 `192.168.1.1` ，重启后需接回 `LAN` 口才能访问
## 4. 优化
### 4.1. 启用 Breed 内部环境变量
正常情况下刷新网页后应该就可以见到 `openwrt` 的登录界面了，但我们需要通过**拔插电源，长按 reset 键 5~10 s**重新进入 Breed 页面（有任何意外都可通过此操作回到 Breed）。
在 `环境变量设置` 处选择**启用控制**，其余配置如下
![image.png](http://imgur.41an.com/piclist/2026-06/openwrt-3191944f7e606bbee22f7b63f7503382.png)
### 4.2. 设置启动地址
在 `环境变量编辑` 处添加键值对，字段为 `autoboot.command`，值为 `boot flash 0x180000`
![image.png](http://imgur.41an.com/piclist/2026-06/openwrt-breed-setEnv-5dfe4060bfbdbe9aff589160cb05787c.png)
保存后直接重启即可
### 4.3. 恢复 eeprom
> factory 是整个设备的生产信息分区，eeprom 是 factory 分区里的核心子数据（主要是无线校准数据）
```shell
wget http://{pc_ip}/Factory.bin
```
擦除并写入
```shell
flash erase 0x50000 0x10000
flash write 0x50000 0x80001000 0x10000
```
## 5. 总结
理想很丰满，显示很骨感。刷完才发现 `overlay` 空间小的可怜，根本无法正常跑 `openclash`，就当个普通的路由系统用吧~
![image.png](http://imgur.41an.com/piclist/2026-06/openwrt-memory-054c42d32a46c58fd1b2561df5390399.png)
## 6. Reference
[更换小米路由器 4A 千兆版的系统](https://www.entropy-tree.top/2024/10/04/flash-the-xiaomi-router/)</br>
[恢复 eeporm](https://blog.csdn.net/cqqjlxy/article/details/140302217)