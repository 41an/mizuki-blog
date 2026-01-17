---
title: SM4算法详解
published: 2025-08-21
description: ''
image: ''
tags:
  - Crypt
  - GM
  - Protocol
category: 密评
draft: true
lang: zh_CN
---

### 0.1. 轮函数 F
$$
F(X_0,X_1,X_2,X_3, rk) = X_0 \oplus T(X_1 \oplus X_2 \oplus X_3 \oplus rk)
$$
### 0.2. 合成置换 T
$$T(B) = L(\tau(B))$$
### 0.3. 线性变换 L
$$
L(B) = B \oplus (B \lll 2) \oplus (B \lll 10) \oplus (B \lll 18) \oplus (B \lll 24)
$$


$L$ 线性变换中，循环左移运算的移位数包括 2、10、18、24
$L^\prime$ `密钥扩展`线性变换中，循环左移运算的移位数包括 13、23

