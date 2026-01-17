---
title: IPsec框架详解
published: 2025-08-07
description: "--"
image: http://imgur.41an.com/picgo/2025-08/IPSec-1754617679961.png
tags:
  - Protocol
category: 密评
draft: false
lang: zh_CN
---
## 1. IPSec 框架简介
IPSec 通过加密与验证等方式，从以下几个方面保障了用户业务数据在 Internet 中的安全传输：

数据来源验证：接收方验证发送方身份是否合法。
数据加密：发送方对数据进行加密，以密文的形式在 Internet 上传送，接收方对接收的加密数据进行解密后处理或直接转发。
数据完整性：接收方对接收的数据进行验证，以判定报文是否被篡改。
抗重放：接收方拒绝旧的或重复的数据包，防止恶意用户通过重复发送捕获到的数据包所进行的攻击。

IPsec 框架通过 `手工` 配置或是 `IKE` 自动协商秘钥，在通过 `AH` | `ESP` 协议传输数据。
## 2. IKE (Internet Key Exchange)
> IKE = ISAKMP + 密钥交换算法（如 DH）+ 身份认证机制

`IKE v1` 于1998年左右引入，并于2005年被 `IKE v2` 取代。
其流程主要分为两步
1. 阶段一建立 `ISAKMP Tunnel` (Internet Security Association and Key Management Protocol) 隧道。也称为 IKE 第一阶段隧道 `IKE Phase 1 Tunnel`、`IKE SA`。
2. 阶段二建立 `IPsec Tunnel`。也称为 IKE 第二阶段隧道 `IKE Phase 2 Tunnel`。

### 2.1. 阶段一 
> 阶段一 （`IKE phaes one`）用于交换秘钥（exchanging key）,为后续密钥协商建立一个安全信道。

#### Step 1 ：`Negotiation`
双方协商加密信息。
- 哈希 `Hashing`：`SHA`/ `MD5` 等
- 身份认证方式 `Authentication`：预共享密钥或数字证书
- DH 组 `Diffie Hellman Group`：确定 DH 交换的过程中使用的秘钥的强度
- 生命周期 `Lifetime`：默认 86400 秒
- 加密方式 `Encryption`：`DES` / `3DES` / `AES` 等
#### Step 2 : `DH Key Exchange`
通过 DH 密钥交换协商出共享密钥。
#### Step 3  : `Authentication`
进行身份认证
- 预共享密钥（PSK），双方提前配置一个共享密码，认证时通过对关键数据进行 HMAC 验证。配置简单，但扩展性差
- 数字证书（RSA 签名），使用 X.509 数字证书和私钥对特定消息签名，对方用公钥验证，安全性更高
- 公钥加密（RSA 加密），对挑战进行加密，对方能解密即验证身份。IKEv 1 支持，IKEv 2 不推荐
- EAP（扩展认证协议），常用于远程拨号用户认证，例如 EAP-MSCHAPv 2，IKEv 2 支持
- DSS 签名，早期支持，但不常用了

最终，建立一个加密认证后的通道 `ISAKMP Tunnel`。

![image.png](http://imgur.41an.com/picgo/2025-08/ike-phase-1-tunnel-1754622600147.png)

IKE 1 阶段隧道仅用于管理流量。我们使用此隧道作为一种安全的方法来建立名为 IPSEC 隧道的第二个隧道，以及诸如 Keepalives 的管理流量。
#### 主模式 main mode

![image.png](http://imgur.41an.com/picgo/2025-08/ike-main-mode-1754637090752.png)

**MM 1**

- 发起者（Initiator）发送第一个包，并使用随机的 `SPI`，responser 的 `SPI` 设置为0。
- 如果在此期间丢包则继续使用之前的 `SPI`，直至达到最大失败重传数。
- SPI 是一一对应的，如果在此期间发现有其他的 SPI 则说明有其他 Initiator 在协商。

![image.png](http://imgur.41an.com/picgo/2025-08/mm1-1754640148632.png)

![image.png](http://imgur.41an.com/picgo/2025-08/mm1-wireshark-1754640188620.png)

**MM 2**

- responser  发送与 `MM1` 中相匹配的已选策略。
- responser SPI 设置为除 0 和 initiator SPI 外的随机值。

 ![image.png](http://imgur.41an.com/picgo/2025-08/mm2-1754640539517.png)

![image.png](http://imgur.41an.com/picgo/2025-08/mm2-wireshark-1754640561881.png)

**MM 3 和 4** 

使用 DH 密码交换协议交换秘钥。

![image.png](http://imgur.41an.com/picgo/2025-08/mm34-1754640642810.png)

**MM 5 和 6**

消息开始加密，双方传输认证信息（通常包含证书、预共享密钥（PSK）或签名）。

![image.png](http://imgur.41an.com/picgo/2025-08/mm56-1754655420041.png)

#### 野蛮模式 Aggressive Mode

![image.png](http://imgur.41an.com/picgo/2025-08/main-vs-aggressive-1754657809049.png)

- AM 1 由 MM 1 和 MM 3 合成，并携带 initiator 的 `身份信息`。
- AM 2 由 MM 2、MM 4 和 MM 6 合成，此时并 `没有加密` 传输 responser 的 `认证信息`，这也是野蛮模式脆弱性的来源。
- AM 3 发送 initiator 的 `身份认证信息`。
### 2.2. 阶段二 
phases two, 为 ESP 或 AH 准备实际的数据加密通道
1.	双方用第一阶段的安全通道发送新的 SA 提议
2.	选择 ESP/AH 的加密算法、认证算法等
3.	使用第一阶段协商出的密钥，派生出新的密钥
4.	完成 IPSec SA 的创建


![image.png](http://imgur.41an.com/picgo/2025-08/ike-phase-2-tunnel-1754623789573.png)



![image.png](http://imgur.41an.com/picgo/2025-08/ike-tunnel-1754623815645.png)

#### 快速模式
快速模式协商共享的IPsec策略（IPSec policy），IPSEC安全算法(IPSec security algorithms)并管理IPSEC SA建立的密钥交换。

![image.png](http://imgur.41an.com/picgo/2025-08/quick-mode-1754658563218.png)
## 3. AH 协议（Authentication Header Protocol）
> AH 仅用于身份认证，而不作数据加密。

![image.png](http://imgur.41an.com/picgo/2025-08/AH-Header-1754563228647.png)

| 字段名                             | 长度   | 描述                                                                |
| ------------------------------- | ---- | ----------------------------------------------------------------- |
| Next Header                     | 1 字节 | 指示下一层协议类型（如 TCP=6，UDP=17）                                         |
| Payload Length                  | 1 字节 | AH 头的长度，以 4 个字节为单位，为了兼容 IPv 6 扩展头的格式规定还需减去 2 个单位（也就是 AH 头的 8个字节）。 |
| Reserved                        | 2 字节 | 保留，必须为 0                                                          |
| SPI (Security Parameters Index) | 4 字节 | 表示用于这条连接的安全参数                                                     |
| Sequence Number                 | 4 字节 | 防止重放攻击，每个包都要递增 +1                                                 |
| Authentication Data             | 可变   | 认证值（HMAC、SHA 1 等计算出来的摘要）                                          |

---
AH 特性：
- 不兼容 NAT。因为 AH会对 IP 头做完整性保护，但 NAT 设备会改 IP 头（比如改源地址），导致校验失败。
- 计算 HMAC 时，涵盖了整个 IP 报文（包括 IP 头）但是要注意：
    - IP 头中一些传输过程中会改变的字段（如 TTL、checksum）在计算认证时会被归零处理，以避免认证失败。
    - Authentication Data 字段在计算时也被视为全 0，否则认证码无法一致。

![image.png](http://imgur.41an.com/picgo/2025-08/AH-1754662629175.png)
#### AH in Transport Mode
- 当数据包抵达目的地后，系统会验证其身份与完整性，验证通过后，`IPSec 模块` 会移除 `AH Header`，并根据其中标记的 `Next Header` 字段将数据交由正确的上层处理程序。

![image.png](http://imgur.41an.com/picgo/2025-08/AH-Transport-1754563267967.png)

#### AH in Tunnel Mode
- 当数据包抵达目的地后，系统会验证其身份与完整性，验证通过后，`IPSec 模块`会移除 `IP Header` 和 `AH Header` ，并根据其中标记的 `Next Header` 字段将数据交由正确的上层处理程序。


![image.png](http://imgur.41an.com/picgo/2025-08/AH-Tunnel-1754563286918.png)

## 4. ESP 协议
 > ESP（Encapsulating Security Payload，封装安全载荷），用作数据加密，也可选择是否做完整性校验。
 
![image.png](http://imgur.41an.com/picgo/2025-08/esc-wo-auth-1754669388161.png)


| 字段名                     | 长度   | 描述                                                               |
| ----------------------- | ---- | ---------------------------------------------------------------- |
| SPI                     | 4字节  | 标识这个 IPsec 会话的安全参数，由 IKE 协商生成。                                   |
| Sequence Number         | 4 字节 | 防止重放攻击，每个包都要递增 +1                                                |
| Encrypted Payload       | 可变   | 原始的传输层数据（TCP/UDP/…） 或者原始 IP 数据包（隧道模式）。<br>可能包含填充（Padding）来对齐加密块。 |
| Pad Length              | 1 字节 | 表示填充的字节数                                                         |
| Next Header             | 1 字节 | 指示上层协议类型                                                         |
| Authentication Data（可选） | 可变   | MAC/HMAC 值，长度取决于算法                                               |

ESP 还可以选择提供身份验证，其 HMAC 与 AH 中的相同。但与 AH 不同的是，此身份验证仅适用于 ESP 标题和加密有效载荷，它虽不涵盖完整的 IP 数据包，但再在实质上却没有削弱身份验证的安全性。

![image.png](http://imgur.41an.com/picgo/2025-08/esp-with-auth-1754669403362.png)


![image.png](http://imgur.41an.com/picgo/2025-08/esp-1754662664888.png)

#### ESP in Transport Mode
> 与 AH 的传输模式类似。


![image.png](http://imgur.41an.com/picgo/2025-08/esp-transport-1754669754921.png)

#### ESP in Tunnel Mode
> 与 AH 的隧道模式类似。

![image.png](http://imgur.41an.com/picgo/2025-08/esp-tunnel-1754669771051.png)

## 5. AH-ESP
单独的 AH 在现实中已经很少应用了，更别说 AH+ESP 了。原因如下：
- 功能重叠，ESP 也带有完整性以及身份认证功能。
- 成本的增加，`带宽` 的开销以及设备处理其的 `性能` 开销。
- AH 不兼容 NAT。

但在理论其仍可使用，传输模式以及隧道模式，包结构仍类似单独的 AH 或 ESP 的传输模式以及隧道模式。
## 6. 连接建立后的通信流程
#### 名词解释
- SA（Security Association，安全关联），描述一条安全通信通道的“合同”或“规则集合”，规定双方通信时要用的安全参数
- SPI（Security Parameter Index，安全参数索引），是 SA 的唯一 ID（加上目的 IP、协议号，才能唯一定位），接收端根据 SPI 知道应该去哪个 SA（在 SAD 中查找）来解密和验证。
- SAD（Security Association Database，安全关联数据库），存储当前已经建立的 SA 的“运行时数据库”。
- SPD（Security Policy Database，安全策略数据库），存放安全策略的规则表，决定“对某个数据流做什么处理”。

IPsec 工作原理：

![image.png](http://imgur.41an.com/picgo/2025-08/ipsec-working-principles-1754726052330.png)


SPD 结构如下：

| 序号  | 源地址            | 目的地址         | 协议   | 源端口 | 目的端口 | 操作类型          | 关联 SA 标识       | 备注          |
| --- | -------------- | ------------ | ---- | --- | ---- | ------------- | -------------- | ----------- |
| 1   | 192.168.1.0/24 | 10.0.0.0/24  | TCP  | any | 80   | Protect (ESP) | SPI=0x12345678 | HTTP 流量加密传输 |
| 2   | 0.0.0.0/0      | 0.0.0.0/0    | ICMP | any | any  | Bypass        | -              | ICMP 流量不加密  |
| 3   | 10.0.0.10      | 192.168.1.10 | UDP  | 500 | 500  | Protect (AH)  | SPI=0x87654321 | IKE 协议数据包处理 |
| 4   | any            | any          | any  | any | any  | Discard       | -              | 其它流量拒绝      |

SAD 结构如下：

| SPI        | 目的地址        | 协议  | 加密算法        | 认证算法         | 加密密钥      | 认证密钥      | 生效时间                   | 过期时间                   |
| ---------- | ----------- | --- | ----------- | ------------ | --------- | --------- | ---------------------- | ---------------------- |
| 0x12345678 | 10.0.0.1    | ESP | AES-CBC-128 | HMAC-SHA 256 | 0xA1B2C3… | 0x112233… | 2025-08-01<br>12:00:00 | 2025-08-15<br>12:00:00 |
| 0x87654321 | 192.168.1.1 | AH  | None        | HMAC-SHA 1   | (无)       | 0x556677… | 2025-08-01<br>12:00:00 | 2025-08-15<br>12:00:00 |

## 7. IPSec Q&A
#### 为什么需要 IKE，而不能直接调用 ISAKMP、DH 等
如下为 IKE 中使用的各组件的作用及缺陷：

| 组件      | 作用                                                       |
| ------- | -------------------------------------------------------- |
| ISAKMP  | 是一种通用框架协议，用于协商、建立、修改和删除安全关联（SA），但不定义加密或认证算法。仅是个类似空盒子的框架。 |
| DH 密钥交换 | 一种算法，用于安全地协商出共享密钥，但它不具备身份认证能力，易受中间人攻击。                   |
| 身份认证机制  | 验证对方是谁（PSK、证书、EAP 等），但它需要配合具体流程和消息格式才好用。                 |

由此可见
- ISAKMP 是“架子”，但没有加密和身份认证
- DH 能产生密钥，但无法验证是谁在和你通信
- 身份认证解决“你是谁”，但得和密钥配合才能安全通信
#### IKE 协商相较手动配置的优势
1. 通过IKE 协商出来的 IPsec 通道，会使用 DH 交换出来的主密钥去生成派生秘钥，用这个派生秘钥来进行 SA 的加密。因此 IKE 可以 `周期性` 生成主密钥，主密钥再生成临时秘钥（`SA/session key`）, 即使主密钥泄露，攻击面也有限（前向安全）。
2. IKE 自动协商时会生成临时密钥并包含防重放计数器。手动协商本身不生成任何序列号或计数器。
3. 手动易配置错误，且难以应对大型网络。
#### 为什么 IKE协商时可以直接发送 DH 公钥，而不用交换 p，g?
在 main mode 和 aggressive mode 里 p 和 g 都是写死在协议里的固定值，不需要通信双方再去协商，只需选择不同的 DH Group 即可直接使用不同长度的 p 和 g 以及不同的算法（整数域和椭圆曲线），组数越大越安全，相应的计算成本也会提升。
#### 为什么不在原有协商好的的 IKE SA 基础进行通信呢，为什么要另外协商一个 IPSec sa?
IKE 特性：
- 本质为了协商而用，包含复杂的握手、认证、协商信息，每条消息都需要很多协商字段、加密签名和状态维护。
- 消息头部比较大，报文格式复杂，导致每次发送的控制消息包头占比高。
- 运行在用户空间（UDP 500端口），IPsec 则是内核层（或硬件层）直接处理的协议。

因此如果用 IKE SA 直接传输数据，会导致数据包中大量冗余协商字段，浪费带宽，效率低下。
## 8. Ref
[An Illustrated Guide to IPsec](http://www.unixwiz.net/techtips/iguide-ipsec.html)

[ZWIR45xx Application Note - Security Using IPsec and IKEv 2 in 6 LoWPANs](https://www.mouser.com/pdfdocs/IDT_ZWIR45xxAppNote-6LoWPAN-SecurityIPsec-IKEv2-160412_APN_20160420.pdf?srsltid=AfmBOor7V1JCwWab0gs22NTXUQt-TbunNg3Lkl98SNC90CPYilOCU1aW)

[P4-IPsec: Site-to-Site and Host-to-Site VPN with IPsec in P4-Based SDN](https://ar5iv.labs.arxiv.org/html/1907.03593)

[IPSec VPN之IKEv2协议详解](https://cshihong.github.io/2019/04/09/IPSec-VPN%E4%B9%8BIKEv2%E5%8D%8F%E8%AE%AE%E8%AF%A6%E8%A7%A3/)