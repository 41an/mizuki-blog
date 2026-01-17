---
title: SM2算法详解
published: 2025-08-19
description: 椭圆曲线离散对数问题。
image: https://imgur.41an.com/picgo/2025-08/point_multi_demo_ecc-point-add-1755602849225.gif
tags:
  - Crypt
  - GM
  - Protocol
category: 密评
draft: false
lang: zh_CN
---
> 写在前面：加密算法的安全性依赖于一个尚未被有效求解的数学问题。
## 1. ECC
> ECC 是一种公钥密码算法，其安全性基于椭圆曲线离散对数问题 (ECDLP, Elliptic Curve Discrete Logarithm Problem) 的困难性。
### 1.1. 椭圆曲线方程 / Elliptic Curve Equation
最常用形式：短 Weierstrass 方程（适用于大多数密码学应用）：
$$
E : y^2 \equiv x^3 + ax + b \pmod{p}
$$
条件：曲线必须是 `非奇异` (nonsingular)(即曲线不包含奇点)，即判别式不为零：
$$
\Delta = -16(4a^3+27b^2) \neq0
$$
> 如果椭圆曲线是连续的值则会导致如下问题，以致并不适合加密。

1. 不可计算：实数有无限小数展开，计算机只能处理有限精度 → 无法精确实现群运算。
2. 安全性崩溃：实数上定义的“椭圆曲线离散对数问题 (ECDLP)”就不再是离散的，而是“连续对数问题”，可以用微积分、牛顿迭代之类的数值方法近似求解，完全没安全性。
3. 没有有限群结构：加密需要在有限群里定义难题（类似 RSA 的 $\mathbb{Z}_n^*$），而实数上的椭圆曲线形成的是连续群，不具备离散难题的条件。

因此需要把椭圆定义在有限域[^1]上，并通过模运算将结果始终限定在在 $[0, p-1]$，只有这样，才能形成一个 有限、离散、可计算的群，才能保证运算可实现，结构完备，并依赖 ECDLP 提供安全性。
> 在密码学中，曲线是定义在有限域 $\mathbb{F}p$ 或 $\mathbb{F}{2^m}$ 上，而不是实数域。
### 1.2. 完整过程
1. 选一条 椭圆曲线 $Ep (a, b)$ , 并取椭圆曲线上一点作为 ` 基点 P `。
2. 选定一个大数 `k 作为私钥`, 并生成 `公钥` $Q=KP$。
3. 加密：选择 `随机数 r`, 将 `消息 M` 生成 `密文 C`。密文是一个点对, 即 $C= (rP, M+rQ)$`。
4. 解密：$M + rQ - K(rP) = M + r(kP) -k(rp) = M$
### 1.3. 加密详解
$Q = kP$
-  `P` 为基点（`base point`），即 P (x, y)
- `k` 为私钥（`provate key`）
- `Q` 为公钥（`public key`）

![](https://imgur.41an.com/picgo/2025-08/ecc-point-add-1755602849224.gif)
#### 点的加法 $P + Q$
给定两点 $P=(x_1,y_1)$, $Q=(x_2,y_2)$：
- 如果 $P = \mathcal{O}$ → $P + Q = Q$
- 如果 $Q = \mathcal{O}$ → $P + Q = P$
- 如果 $x_1 = x_2$ 且 $y_1 \equiv -y_2 \pmod p$ → $P+Q = \mathcal{O}$ （互为对称点）
- 否则，斜率为：
    $$
    \lambda = \frac{y_2 - y_1}{x_2 - x_1} \pmod p
    $$
    结果点 $R = (x_3,y_3)$：
    $$
    x_3 = \lambda^2 - x_1 - x_2 \pmod p
    $$
    $$
    y_3 = \lambda(x_1 - x_3) - y_1 \pmod p
    $$
#### 倍点 $2P$
> 若 `A` 与 `B` 重合，其结果等同 `2A`（`Point Doubling`），即与曲线作切线求交点。

如果 $P=(x_1,y_1)$，则：
  • 若 $y_1 = 0$ → $2P = \mathcal{O}$
  • 否则：
$$
\lambda = \frac{3x_1^2 + a}{2y_1} \pmod p
$$
$$
x_3 = \lambda^2 - 2x_1 \pmod p
$$
$$
y_3 = \lambda(x_1 - x_3) - y_1 \pmod p
$$
#### 标量乘法 / Scalar Multiplication $kP$
> 这是 ECC 的核心运算，也是公钥生成和加密的基础。ECDLP 的难题：已知 P 和 Q = kP，求 k 在大素数域下是指数级困难。

定义为 $P$ 自加 $k$ 次，即：
$$
kP = P + P + … + P \quad (k\ \text{次})
$$

实际计算中用双倍-加法算法 (double-and-add)：
- 把 $k$ 写成二进制
- 每次循环做“倍点”和“条件加法”
- 时间复杂度 $O(\log k)$

由此可见 `ECC` 的加密与解密本质上都是标量乘法运算，可通过 `double-and-add` 或 `windowing` 等算法以 `O(log k)` 复杂度完成。这也引出了其与 `RSA` 的差异：
- `RSA`：加密快，解密慢
- `ECC`：加密与解密同速

如下为基点在有限域内经过多次点加后的结果。
![image.png](http://imgur.41an.com/picgo/2025-08/point_multi_demo-1755619357876.png)
### 1.4. 解密详解
解密者有私钥 d。
1.  收到 $(C_1, C_2)$。
2.  计算：

$$
dC_1 = d (kG) = (dk) G = k (dG) = kQ
$$

3.  恢复明文：

$$
P_m = C_2 - dC_1
$$
原因：
$$
C_2 - dC_1 = (P_m + kQ) - kQ = P_m
$$
## 2. SM2
Loading...


---
## 3. Ref
-  [ECC加密算法](https://www.bilibili.com/video/BV1v44y1b7Fd/?spm_id_from=333.337.search-card.all.click&vd_source=2898185808023cdd085691a99b81da2f)
-  [信息安全技术 SM2椭圆曲线公钥密码算法](https://openstd.samr.gov.cn/bzgk/gb/newGbInfo?hcno=3EE2FD47B962578070541ED468497C5B)
-  [Elliptic curves are quantum dead, long live elliptic curves - COSIC](https://www.esat.kuleuven.be/cosic/blog/elliptic-curves-are-quantum-dead-long-live-elliptic-curves/)
-  [Elliptic-Curve Cryptography](https://medium.com/coinmonks/elliptic-curve-cryptography-6de8fc748b8b)

[^1]: 有限域，即伽罗瓦域（Galois Field）：一个有限域，记作 $\mathbb{F}_q$ 或 $GF(q)$，其中包含有限个元素 $q = p^m$（p 为素数，m 为正整数），并且加法、乘法封闭且满足域的运算规律。