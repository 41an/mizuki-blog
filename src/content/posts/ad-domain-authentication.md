---
title: AD域认证
published: 2025-08-03
description: AD域的后渗透思路
image: http://imgur.41an.com/picgo/2025-08/ad-domain-1754621509710.png
tags:
  - Kerberos
  - Windwos
category: 后渗透
draft: true
lang: zh_CN
---

## 1. 票据工具  
- 黄金票据：使用 **`krbtgt`** 密钥伪造一个长期有效的 TGT。
- 白银票据：攻击者通过获取 **特定服务账户的密钥**（如 `HTTP/server`、`SQL/server`），直接伪造 **服务票据（Service Ticket）**。  
## 2. NTLM 认证相关
- NTLM Hash：密码经过 MD 4 生成 128 位哈希。
- 客户端对 Server 挑战（Challenge）使用 NTLM Hash 加密，生成响应。
- NTLM v 1 无时间戳，v 2 有±5 分钟时间容差。
- 被截获的 Net-NTLMv 2 Hash 必须快速中继。


![543fe766-d30b-4dac-adf9-ab3d29c7ac9d.png](http://imgur.41an.com/picgo/2025-08/kerberos-1754621616971.png)