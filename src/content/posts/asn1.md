---
title: ASN.1格式详解
published: 2025-08-06
description: 'X.509、ASN.1、DER/PEM、TLV之间的关联性。'
image: 'http://imgur.41an.com/picgo/2025-08/ASN.1-1754495593823.gif'
tags: [Crypt,GM]
category: '密评'
draft: false 
lang: 'zh_CN'
---

## 1. 什么是 ASN.1 
`ASN.1` 抽象语法标记（Abstract Syntax Notation One）：一种描述数据结构的标准（类似结构体的定义方式）。其定义了一种抽象语法，用于描述资料的结构，包括资料的类型、值范围、结构和命名。e.g.
```asn.1
StudentCards DEFINITIONS AUTOMATIC TAGS ::= BEGIN  
  
StudentCard ::= SEQUENCE {  
    dateOfBirthday DATE,  
    student StudentInfo  
}  
  
StudentInfo ::= SEQUENCE {  
    studentName VisibleString (SIZE (3..50)),  
    homeAddress Address,  
    contactPhone NumericString (SIZE (7..12))  
}  
  
Address::= SEQUENCE {  
    street VisibleString (SIZE (5 .. 50)) OPTIONAL,  
    city VisibleString (SIZE (2..30)),  
    state VisibleString (SIZE(2) ^ FROM ("A".."Z")),  
    zipCode NumericString (SIZE(5 | 9))  
}  
  
END
```
在上述的例子中，我们不难发现 `ASN.1` 与编程语言的 `结构体`（如 `C` 的 `struct` ） 有着许多相似之处。两者都用于描述复合数据类型的结构，即一个数据结构应由哪些字段组成，每个字段的数据类型、顺序、以及可能的长度和用途。
## 2. ASN.1 的编码格式
> `DER` 则是 `ASN.1` 的一种实现，是一种将 `ASN.1` 结构编码为二进制的方式（编码规则）。
> 即 `ASN.1` 定义“`长什么样`”，`DER/BER` 基于 `TLV` 格式定义把它“`怎么存`”。

DER、PEM 等是 ASN.1 编码后的“序列化格式”。区别是：
- `DER` 是以 `二进制` 编码表示 `ASN.1` 
- `PEM` 是以 `base64` 编码表示 `ASN.1` 

```text
ASN.1（抽象结构定义）
    ↓ 编码为 TLV
DER / BER / CER（编码规则，序列化为二进制）
    ↓ Base64 + 标头/尾
PEM（适合文本传输的表示方式）
```

| **格式** | **全称**                       | **编码方式**               | **特点**                       | **用途**                   |
| ------ | ---------------------------- | ---------------------- | ---------------------------- | ------------------------ |
| DER    | Distinguished Encoding Rules | 二进制格式                  | 严格的 ASN.1 DER 编码，体积小，适合程序处理 | 证书、密钥                    |
| PEM    | Privacy-Enhanced Mail        | Base 64 编码的 DER，加上头尾标记 | 易读，常用于配置文件                   | OpenSSL / Web 服务器配置中广泛使用 |

证书的文件扩展名可以是很多种，如 
-  `.crt`、`.cer` 它们均取自证书  `Certificate` 的缩写，内容通常是 `X.509` 公钥证书，其编码格式可以是二进制的 `.der` 也可以是 `base64` 的 `.pem`。 
- `.der` 和 `.pem` 则是直接表名了该证书文件是以什么编码格式编码的。
- `.key` 通常表示 `RSA` 或 `EC` 私钥。
## 3. TLV
> TLV 是解析 ASN.1 编码（如 DER）时的核心概念之一，ASN.1 的编码实现也是基于 TLV 的。

`TLV` 由三个部分组成 `Tag` 标识数据类型、`Length` 长度、`Value` 真实的数据内容。

#### Tag
在 ASN.1 编码（如 DER/BER）中，每一个数据项都是以 TLV（Tag-Length-Value） 格式存储的。Tag（标签） 是三者中的第一部分，它的作用是标明后面数据的类型与结构。如常见的 `0x02` 以 `INTEGER` 纯字符串的形式解析它的值，`0x30` 以 `SEQUENCE` 结构体的形式解析它。

| **类型**            | **十六进制 Tag** | **含义**                       |
| ----------------- | ------------ | ---------------------------- |
| BOOLEAN           | 01           | 布尔值（TRUE=0 xFF，FALSE=0 x 00） |
| **INTEGER**       | **02**       | **整数（补码表示）**                 |
| BIT STRING        | 03           | 位字符串（首字节表示未使用位数）             |
| OCTET STRING      | 04           | 字节串（如哈希、密钥）                  |
| NULL              | 05           | 空值（无值，仅表示存在）                 |
| OBJECT IDENTIFIER | 06           | 用于表示算法、协议等的 OID              |
| UTF 8 String      | 0 C          | UTF 8 字符串                    |
| PrintableString   | 13           | 可打印字符串（限字符集）                 |
| IA 5 String       | 16           | ASCII 字符串                    |
| UTCTime           | 17           | 时间格式（两位年份）                   |
| GeneralizedTime   | 18           | 时间格式（四位年份）                   |
| **SEQUENCE**      | **30**       | **顺序结构体**                    |
| SET               | 31           | 无序结构体（不常用于 DER）              |

#### Length
短格式：当长度 **小于** `128(0x80)` 时，直接使用一个字节表示长度。

长格式：当长度 **大于** `128(0x80)` 时，第一个字节的 `低7位` 表示后续有多少个字节用于表示长度本身。如有如下一串数据 ：
```
10000010 00000001 00111001 ...
```
每 4 个bit为一个间隔，转为 16 进制，即
```text
82 01 39 ...
```
首位置 1，第 7 位值为 2，表示后续两个字节均表示长度，`0x0139` 即 `313`，后续的 `313`  个字节数据都是值。

#### Value
`Value` 可以是简单的基本类型如 `整数`、`字符串` 也可以是复杂的、递归嵌套的 `TLV` 结构。

现在让我们来尝试分析一下这串 `sm2` 签名值。
```text
304502206483816d342350b68a425f9a4e9490dd049764b4af5056d39548d06abfe9075302210093bba6b3e0090483c53340394ff77f22a37f08f11b8c84f16c1f796fe2140611
```
其以 `0x30 45` 开头，我们不难发现他是一个长度为 `45` 个字节的 `SEQUENCE` 顺序结构体数据
```text
30 45 
02206483816d342350b68a425f9a4e9490dd049764b4af5056d39548d06abfe9075302210093bba6b3e0090483c53340394ff77f22a37f08f11b8c84f16c1f796fe2140611
```
再继续我们可以看到他以 `02 20` 开头，表示其为一个长度为 `0x20` 的 `INTEGER` 整数。
```text
30 45 
    02 20 
        6483816d342350b68a425f9a4e9490dd049764b4af5056d39548d06abfe90753
    02 21
        0093bba6b3e0090483c53340394ff77f22a37f08f11b8c84f16c1f796fe2140611
```
`ASN.1` 中的 `INTEGER` 是带符号的，如果最高 bit 为 1，必须在前面补一个 00。所以最终我们发现 `s` 虽然是 32 字节的值，但因为它的高位是 `0x93`（`高bit为1`），所以前面补了一个 `00`。
## 4. X.509
> `X.509` 是一套 ` 数字证书的国际标准`，它定义了证书应有哪些字段、每个字段的格式要求（比如 subject、issuer、public key、有效期等）。

或许你也会有一样的困惑，认为 `X.509`、`ASN.1` 、`DER`、`TLV` 设计的很繁琐冗余，为什么不能直接用 `X.509` 套上 `TLV`，下面就让我们来简单说说。

1. 当定义好证书该具备哪些字段后，会发现没有约束这些信息的 `顺序`，没有一套标准的解析顺序则会导致人们不知道这些字段应该什么格式、什么时候出现的。
2. 当使用 `ASN.1` 约束好信息的顺序后，我们会发现我们定义的证书机构的名字可能是以 `ASCII` 编码的一串字符串，签名值又可能是一串 `hex` 编码的字符串，我们又缺失一套将`不同编码`格式的字段转为`二进制`的规定。
3. 接着，我们使用了 `DER` 等规定，将不同编码的数据按照约定好的方式转换成二进制进行传输，最终这些证书终于可以正常传输被大家正常解析了，
4. 但是很快我们发现，代码按照预设的处理逻辑可以很快地识别出这是什么数据、要如何分析这些数据，但是肉眼却`很难直观的识别`。最终，我们想到了将 `DER` 再转换成 `BASE64` 并附上 `提示符` （如 `-----BEGIN CERTIFICATE-----`）以便识别。

## 5. openssl 库的使用
除了在线的分析网站，如[ASN.1 JavaScript decoder](https://lapo.it/asn1js/)，[pkitools-asn1](https://pkitools.net/pages/ca/asn1.html)之外，openssl 库自身就带 `ASN.1` 结构数据的解析功能。

#### 证书解析
```shell
openssl asn1parse -in cert.der -inform DER
```
#### 格式转换
```shell
# PEM → DER
openssl x509 -in cert.pem -outform der -out cert.der

# DER → PEM
openssl x509 -in cert.der -inform der -out cert.pem
```
---
## 6. 参考
1. [microsoft about-sequence](https://learn.microsoft.com/zh-tw/windows/win32/seccertenroll/about-sequence)
2. [密码学系列之:ASN.1接口描述语言详解](http://www.flydean.com/ASN.1/)