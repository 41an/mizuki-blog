---
title: Kerberos协议详解
published: 2025-08-02
description: Kerberos协议里，各阶段的认证流程。
image: http://imgur.41an.com/picgo/2025-08/kerberos-1754621160013.png
tags:
  - Kerberos
  - Windwos
  - Protocol
category: 后渗透
draft: false
lang: zh_CN
---


## 1. AS 认证过程（获取 TGT）
> `AS` 判断  `Client` 是否有权限访问 `KDC`。
  
#### 1.1.1. Client 向 KDC 发送认证消息 KRB_AS_REQ
- Pre-authentication data：Client 用自己的密码哈希对时间戳签名。
- Client Info：用户名、计算机名、IP 等。
- Server Info：目标服务（这里是 TGS）。
#### 1.1.2. KDC 校验 Client 身份
- 由 `AS`判断 `Client` 是否在白名单（AD 中有无权限、用户名、地址合法等）。
- 如果合法，生成 `TGT`。
#### 1.1.3. KDC 向 Client 返回 KRB_AS_REP
- `Client hash` (Session Key)：使用 `Client密码hash` 加密的Session Key。
- `TGT`：用 `krbtgt密钥` 加密的（Session Key + Client Info + 有效期）。

![AD-AS-Exchange.png](http://imgur.41an.com/picgo/2025-08/AD-AS-Exchange-1754127921733.png)

## 2. TGS 票据申请过程（获取服务访问票据）
> `TGS` 判断  `Client` 是否有权限访问 `Server ` 
  
#### 2.1.1. Client 向 KDC 发送请求    
- TGT（证明自己已被认证）
- Session Key（解密 TGT 时用）
- Client Info、Server Info
#### 2.1.2. KDC 校验
- 解密 TGT，获得 Session Key。
- 用 Session Key 解密 Client 请求，验证时间戳和权限。
- 确认 Client 有权限访问指定服务。
#### 2.1.3. KDC 向 Client 返回
- Session Key (Server Session Key)：用于 Client 和 Server 通讯的会话密钥。
- Ticket（用 `服务器密钥` 加密的票据，含 Server Session Key、Client Info、域信息、到期时间等）。

![AD-TGS-Exchange.png](http://imgur.41an.com/picgo/2025-08/AD-TGS-Exchange-1754127997335.png)  

## 3. Client 访问服务器（服务认证）
`Client` 向 `Server` 发送 `KRB_AP_REQ`
- Ticket
- Server Session Key 加密的 Client Info 和时间戳

Server 校验票据和身份，允许访问

![AD-CS-Exchange.png](http://imgur.41an.com/picgo/2025-08/AD-CS-Exchange-1754127976483.png)  

最后，再完整来看一下整个认证流程。

![](https://imgur.41an.com/picgo/2025-08/kerberos-protocol-1754151453618.svg)

## 4. 为什么需要 KDC
> 分权、授权、缓存、互信
#### 4.1.1. 解耦应用和账号系统
- 每个服务无需单独维护用户账号与密码系统。
- 用户修改一次密码，全局统一生效，避免分布式账户管理的混乱。
#### 4.1.2. 降低风险面
- 减少口令传输的过程。
- 减轻认证压力，简化结构。

#### 4.1.3. 可扩展性
- 新服务上线仅需直接对接 TGS，无需再搭建认证服务。
#### 4.1.4. 体验
- 用户获取 TGT 后，可通过票据访问多个服务，无需重新登陆———SSO（Single Sign-On）单点登录。

因此预认证完会发放 `TGT`，后续可以使用 `TGT` 完成剩下的操作
## 5. 为什么会分出 AS 和 TGS 
>可控性、拓展性、SPR。

AS 主要负责校验账户真实性。

TGS 负责校验账户是否有访问某服务的权限。


## 6. Ref
[# Kerberos I - Overview](https://labs.lares.com/fear-kerberos-pt1/)