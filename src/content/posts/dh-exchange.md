---
title: Diffie-Hellman秘钥交换
published: 2025-08-10
description: ""
image: "http://imgur.41an.com/picgo/2025-08/DH-1754545504533.png"
tags:
  - Crypt
  - Protocol
category: 密评
draft: true
lang: zh_CN
---

> 有限域的离散对数问题的复杂度正是支撑 Diffie-Hellman 密钥交换算法的基础。

Alice 和 Bob 共享一个素数 q, 以及整数 `a(a<q)` , 且 a 是 q 的原根。

$((a^x mod p)^y) mod p = a^xy mod p$<br>
$(a*b)mod p = (a mod p * b mod p) mod p$




<table border="1">
  <tr>
    <th colspan="1" style="text-align: center;">Alice​</th>
    <th colspan="1" style="text-align: center;">Bob​</th>
  </tr>
  <tr>
     <td style="text-align: center;">Public Keys available = P, G</td>
     <td style="text-align: center;">Public Keys available = P, G</td>
  </tr>
  <tr>
	 <td style="text-align: center;">Private Key Selected = a</td>
	 <td style="text-align: center;">Private Key Selected = b</td>
  </tr>
    <tr>
    <td style="text-align: center;">Key generated =<br>x=G<sup>a</sup>modP</td>
    <td style="text-align: center;">Key generated =<br>y=G<sup>b</sup>modP</td>
  </tr>
  <tr>
    <th colspan="2" style="text-align: center;">Exchange of generated keys takes place</th>
  </tr>
  <tr>
    <td style="text-align: center;">Key received = y</td>
    <td style="text-align: center;">key received = x</td>
  </tr>
    <tr>
    <td style="text-align: center;">Generated Secret Key = <br>k<sub>a</sub>=y<sup>a</sup>modP</td>
    <td style="text-align: center;">Generated Secret Key = <br>k<sub>b</sub>=x<sup>b</sup>modP</td>
  <tr>
    <th colspan="2" style="text-align: center;">Algebraically, it can be shown that <br><br>k<sub>a</sub>=k<sub>b</sub></th>
  </tr>
  <tr>
    <th colspan="2" style="text-align: center;">Users now have a symmetric secret key to encrypt</th>
  </tr>
</table>


 

---
## 1. 拓展
椭圆曲线 Diffie-Hellman 密钥交换


---
## 2. ref
[Implementation of Diffie-Hellman Algorithm](https://www.geeksforgeeks.org/computer-networks/implementation-diffie-hellman-algorithm/)

[可厉害的土豆-Diffie-Hellman密钥交换算法](https://www.bilibili.com/video/BV12w411f7c5/?spm_id_from=333.337.search-card.all.click&vd_source=2898185808023cdd085691a99b81da2f)

