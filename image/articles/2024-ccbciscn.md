---
title: 2024-ccbciscn
date: 2024-12-16 12:20:51
tags:
---

# 12.15 ciscn&“长城杯”铁人三项 部分题解

### 前言

做出来的两道密码到最后都只有五十分了（悲）

先把做出来的题分析一下，之后有时间了把剩下两道密码题补一下。

### rasnd

一开始看标题以为是什么随机数（？），折腾老半天

##### 题目内容：

> 怎么全是114514啊，他有什么用呢

##### 题目附件：

```python
from Crypto.Util.number import getPrime, bytes_to_long
from random import randint
import os

FLAG = os.getenv("FLAG").encode()
flag1 = FLAG[:15]
flag2 = FLAG[15:]

def crypto1():
    p = getPrime(1024)
    q = getPrime(1024)
    n = p * q
    e = 0x10001
    x1=randint(0,2**11)
    y1=randint(0,2**114)
    x2=randint(0,2**11)
    y2=randint(0,2**514)
    hint1=x1*p+y1*q-0x114
    hint2=x2*p+y2*q-0x514
    c = pow(bytes_to_long(flag1), e, n)
    print(n)
    print(c)
    print(hint1)
    print(hint2)


def crypto2():
    p = getPrime(1024)
    q = getPrime(1024)
    n = p * q
    e = 0x10001
    hint = pow(514*p - 114*q, n - p - q, n)
    c = pow(bytes_to_long(flag2),e,n)
    print(n)
    print(c)
    print(hint)
print("==================================================================")
crypto1()
print("==================================================================")
crypto2()
print("==================================================================")

```

其实是很简单的解方程

##### Part.1

$$
\left\{\begin{matrix}
h_1 &=& x_1 p + y_1 q - \text{0x114}\\
h_2 &=& x_2 p + y_2 q - \text{0x514}\\
n &=& pq
\end{matrix}\right.
$$

$3$ 个方程，$6$ 未知元 ($x_1, x_2$ 可爆破，$y_1$ 难以爆破，$y_2, p,q$ 不可爆破)

直接解肯定是不行的，得想想办法

最开始想直接碰撞 $x_1 = x_2$ 的情况来着，但是下发的容器太慢了，跟有工作量证明一样慢（

尝试消元

先记 $h_1' = h_1 + \text{0x114}, h_2' = h_2 + \text{0x514}$

感觉像做齐次化一样：
$$
\left\{\begin{matrix}
x_2 h_1' &=& x_2 x_1 p + x_2 y_1 q\\
x_1 h_2' &=& x_1 x_2 p + x_1 y_2 q\\
\end{matrix}\right.
$$
这样就好办了
$$
x_2 h_1' - x_1 h_2' = (x_2 y_1 - x_1 y_2) q\\
q = \gcd(n, x_2 h_1' - x_1 h_2')
$$

```python
def solve1():
    n = int(r.recvline().decode().strip())
    c = int(r.recvline().decode().strip())
    hint1 = int(r.recvline().decode().strip())
    hint2 = int(r.recvline().decode().strip())
    hint1 += 0x114
    hint2 += 0x514
    for x1 in tqdm(range(0, 2**11)):
        for x2 in range(0, 2**11): 
            q = gcd(hint2*x1 - hint1*x2, n)
            p = n // q
            if p != 1 and q != 1:
                print("p =", p)
                print("q =", q)
                print("n =", n)
                phi = (p-1)*(q-1)
                d = int(pow(0x10001, -1, phi))
                m = int(pow(c, d, n))
                flag1 = long_to_bytes(m)
                print(flag1)
                return flag1

```

##### Part.2

关键是 `hint = pow(514*p - 114*q, n - p - q, n)`
$$
h = (514p-114q)^{n-p-q} \mod{n}
$$
注意到
$$
n-p-q=  \varphi(n) - 1
$$
则根据欧拉定理有
$$
h \equiv (514p-114q)^{n-p-q} \pmod{n}\\
h \equiv (514p-114q)^{\varphi(n) - 1} \pmod{n}\\
h \equiv (514p-114q)^{-1} \pmod{n}\\
514p-114q \equiv h^{-1} \pmod{n}  
$$
联立 $n = pq$ 解之

```python
def solve2():
    # problem 2
    n = int(r.recvline().decode().strip())
    c = int(r.recvline().decode().strip())
    hint = int(r.recvline().decode().strip())

    koiship_114q = int(pow(hint, -1, n))

    print(koiship_114q)
    p, q = var('p q')

    equ = [koiship_114q - 514*p+114*q, p*q - n]

    res = solve([*equ], p, q)
    print(res)

    p = res[0][0].right()
    q = res[0][1].right()
    assert n == p * q

    d = int(pow(0x10001, -1, n - p - q + 1))
    m = int(pow(c, d, n))
    print(long_to_bytes(m))
    return long_to_bytes(m)
```

完整脚本：
```python
from Crypto.Util.number import getPrime, bytes_to_long, long_to_bytes
from random import randint
import os
from gmpy2 import gcd
from tqdm import tqdm
from Pwn4Sage.pwn import * # 为什么 sagemath 不能直接用 pwntools 啊 QAQ


def solve1():
    n = int(r.recvline().decode().strip())
    c = int(r.recvline().decode().strip())
    hint1 = int(r.recvline().decode().strip())
    hint2 = int(r.recvline().decode().strip())
    hint1 += 0x114
    hint2 += 0x514
    for x1 in tqdm(range(0, 2**11)):
        for x2 in range(0, 2**11): 
            q = gcd(hint2*x1 - hint1*x2, n)
            p = n // q
            if p != 1 and q != 1:
                print("p =", p)
                print("q =", q)
                print("n =", n)
                phi = (p-1)*(q-1)
                d = int(pow(0x10001, -1, phi))
                m = int(pow(c, d, n))
                print(long_to_bytes(m))
                return long_to_bytes(m)

def solve2():
    # problem 2
    n = int(r.recvline().decode().strip())
    c = int(r.recvline().decode().strip())
    hint = int(r.recvline().decode().strip())

    koiship_114q = int(pow(hint, -1, n))

    print(koiship_114q)
    p, q = var('p q')

    equ = [koiship_114q - 514*p+114*q, p*q - n]

    res = solve([*equ], p, q)
    print(res)

    p = res[0][0].right()
    q = res[0][1].right()
    assert n == p * q

    d = int(pow(0x10001, -1, n - p - q + 1))
    m = int(pow(c, d, n))
    print(long_to_bytes(m))
    return long_to_bytes(m)

r = remote("39.105.45.179", 28774)
print(r.recvline())
flag1 = solve1()
print(r.recvline())
flag2 = solve2()
flag = flag1 + flag2
print(flag)
r.close()

# [94m[INFO][0m Connected to 39.105.45.179:28774
# b'==================================================================\n'
#  79%|███████▉  | 1622/2048 [01:00<00:15, 26.66it/s]p = 113931722055442701446227924225349342923035624336145681834183562625057779377412924188573492959081360857781622186384116214966258654184221455843401118314188230036225081356507628890053642334996763558081421549633854635489449005191128321348912176542344609801537585688185055218829569712481850220492418068455562521829
# q = 149687346278749733067771297796945134099573873882567708652528570721488495821734755104637927800569308648962222037872291706921428733233538343781900144390625108429477113911807232022408337280198004423317446408070180889503468830562635209615088067864562884988577416076468399790923049267827295132481119445792416795149
# n = 17054137131447319945462384577254586097818250280780440320505467706897381254398747857859611806051972107698572439639775195808276084624856874341810622198050944075029080669224262914186173634532945387572640585931189004598187516938998086186995618166426870493457572040586964035600503428663382122178741406778161779710515037520476192110503002807973029096714007655422047760141304087057135939835116788750148377067980861167227866440162107916225595534428842407827591690743865754799172493177485957066320753065436091614874565684831073947206929492837357883825511923747732779932353056073142795755044524329528614552186610574732533807521
# b'flag{ee62271a-b54b-'
# b'==================================================================\n'
# 35610761698930940955771524558264010431522232657139038026752170697961012150702331649488019804859068437215686128408212178056949158912363869075711016212204864344389859925409597955876297487978180421187133658719883091149063748060508765695828722993056359306928172621966424898685428613654429194960129005375870671933276
# [
# [p == 91296421208611922009772689366803267564928168341171916989523970204539467723847488273321200980271550391186875290590716338929702908310241534732665172498948370179431797142127466879844213504727818818087679198845566652075146859497667657189899764407074267181251805245319917664997604618612003883760286995861012937971, q == 99259638616627955765365243651516395586410928686169537770729385852388370696098923885956820166671126875915506762766807194323843297886844559446305986423285946735421787768806315968102002223262442555350293416550334807171594190537652720173506630808594859861712765562526427904590703160632814046426829039444648931437],
# [p == (-5657799401147793478625818888136434548425422935111663652931574993586137129677638661499538749500254231927183885477708010076459067979550139888439441226127298963919041902821960010181814126725959225654966724743369084008780868860646205049889877956089907012117627637064006390561670080156070400646329255248344989091909/257), q == (-23463180250613263956511581167268439764186539263681182666307660342566643205028804486243548651929788450535026949681814099104933647435732074426294949332229731136113971865526758988119962870715049436248533554103310629583312742890900587897804239452618086665581713948047218839904384386983284998126393757936280325058547/57)]
# ]
# b'468f-87e0-78730eebf8b2}'
# b'flag{ee62271a-b54b-468f-87e0-78730eebf8b2}'
# [94m[INFO][0m Connection closed
```

### fffffhash

##### 题目内容

> 听说hash函数不可逆？我不信

##### 题目附件

```python
import os
from Crypto.Util.number import *
def giaogiao(hex_string):
	base_num = 0x6c62272e07bb014262b821756295c58d
	x = 0x0000000001000000000000000000013b
	MOD = 2**128
	for i in hex_string:
		base_num = (base_num * x) & (MOD - 1) 
		base_num ^= i
	return base_num


giao=201431453607244229943761366749810895688

print("1geiwoligiaogiao")
hex_string = int(input(),16)
s = long_to_bytes(hex_string)

if giaogiao(s) == giao:
	print(os.getenv('FLAG'))
else:
	print("error")

```

##### 解题

本题要实现一个哈希碰撞

经过查询魔数，可以搜到这个先乘再异或是一种 FNV hash

参考 https://github.com/DownUnderCTF/Challenges_2023_Public/blob/main/crypto/fnv/solve/solution_joseph_LLL.sage

每一步的异或，都可以等价为加减某个数，因此可以写出式子


$$
G \oplus r = G \pm s
$$
其中 $r, s$ 都是小量，满足
$$
\left \lceil \lg r_{max} \right \rceil = \left \lceil \lg s_{max} \right \rceil = \left \lceil \lg 2^{8} \right \rceil = 8
$$
可以构造多项式
$$
TargetHash = Hash_0 p^n + s_0 p^{n-1} + s_1 p^{n-2} + s_2 p^{n-3} + \cdots + s_{n-2} p^{1} + s_{n-1} p^{0} = Hash_0 p^n + \sum _{i=0}^{n-1} s_i p^{n-1-i}
$$
构造格
$$
\begin{bmatrix}
s_0, s_1, s_2, \cdots, s_{n-2}, s_{n-1}, 1, k
\end{bmatrix}
\begin{bmatrix}
p^{n-1}  & 1 &  &  &  &  &  & \\
p^{n-2}  &  & 1 &  &  &  &  &  \\
p^{n-3}  &  &  & 1 &  &  &  &  \\
\vdots  &  &  &  & \ddots &  &  &  \\
p^{1}  &  &  &  &  & 1 &  &\\
p^{0}  &  &  &  &  &  & 1 &\\
h_0p^n - t  &  &  &  &  &  &  & 1 & \\
N  & 0 & 0 & 0 & \cdots & 0 & 0 & 0 & 
\end{bmatrix}
= 
\begin{bmatrix}
0, s_0, s_1, \cdots, s_{n-3}, s_{n-2}, s_{n-1}, 1
\end{bmatrix}
$$

```python
TARGET = 201431453607244229943761366749810895688
h0 = 0x6c62272e07bb014262b821756295c58d
p = 0x0000000001000000000000000000013b
MOD = 2^128

n = len("1geiwoligiaogiao")
M = Matrix.column([p^(n - i - 1) for i in range(n)] + [-(TARGET - h0*p^n), MOD])
M = M.augment(identity_matrix(n+1).stack(vector([0] * (n+1))))
Q = Matrix.diagonal([2^128] + [2^4] * n + [2^8]) # 这里进行了一点缩放，实测去掉不影响
M *= Q
M = M.BKZ()
M /= Q
for r in M:
    if r[0] == 0 and abs(r[-1]) == 1:
        r *= r[-1]
        good = r[1:-1]
        print(good)
        break
inp = []
y = int(h0*p)
t = (h0*p^n + good[0] * p^(n-1)) % MOD
for i in range(n):
    for x in range(256):
        y_ = (int(y) ^^ int(x)) * p^(n-i-1) % MOD
        if y_ == t:
            print('good', i, x)
            inp.append(x)
            if i < n-1:
                t = (t + good[i+1] * p^(n-i-2)) % MOD
                y = ((int(y) ^^ int(x)) * p) % MOD
            break
    else:
        print('bad', i)
print(bytes(inp).hex())
```

格基规约后再把 `+` 转为 `^` 即可

### Kiwi

很有意思的题*~~（因为是唯一拿到的高分题目）~~*

给了一段流量包和病毒样本，解密 Windows 密码

感谢我们的[逆向大神](https://txpoki.github.io/)，tql！

IDA：

![image-20241216143504496](/image/articles/2024-ccbciscn.assets/image-20241216143504496.jpg)

程序首先隐藏了自己的窗口，使用 `mimikatz` 来获取 `SAM` 等敏感信息，加密后通过 `upload` 函数上传，上传流量可以在流量包附件中查看。

流量包中有超级多的请求，导致一开始分析了老半天上传方式。只找到一个可疑的流量包

![image-20241216144139488](/image/articles/2024-ccbciscn.assets/image-20241216144139488.jpg)

```
l1Mvs8wZ1LI/v3Vup1zF8bzdp1B51zz0e0xdfIXNBQMOe1wFEg+Z03ljczfC1qGdp0Y6bWnJ7rUqnQrZmVT9nFPRXqYpURBxuBKInjI5Q2xVgs56q4VRCQWbiyv00Aw7D0CKEotHSy6sQAC1x3T9wDx6xPCioqx/0nwNgrvJnF1Oq7NFZsVpnAxaZC5BVfKSEttFPjYgv3uSfmtxeJg7pPCHmJ8qf/Sd7W7n3gKSB2BELb==
```

像一段 base64，但实际上解不出来任何东西

通过瞎翻 IDA，发现上传数据前，先进行了一段加密，再进行了换表 base64 编码



![1734331451903](/image/articles/2024-ccbciscn.assets/1734331451903.jpg)



![base64 表](/image/articles/2024-ccbciscn.assets/1734331592986.png)

表：`d+F3DwWj8tUckVGZb57S1XsLqfm0vnpeMEzQ2Bg/PTrohxluiJCRIYAyH6N4aKO9`

*~~像我这种不熟悉逆向和 IDA 的 crypto 手来说，我一开始就是没看出来这个表是从 `64h` 开始而非从 `28h` 开始的（~~*

加密后会在下一个函数中上传数据

![1734331925670](/image/articles/2024-ccbciscn.assets/1734331925670.jpg)

到这里差不多就理清了，解换表 base64 后再解密

解换表

```python
# By MetaMiku
import base64

enc = "l1Mvs8wZ1LI/v3Vup1zF8bzdp1B51zz0e0xdfIXNBQMOe1wFEg+Z03ljczfC1qGdp0Y6bWnJ7rUqnQrZmVT9nFPRXqYpURBxuBKInjI5Q2xVgs56q4VRCQWbiyv00Aw7D0CKEotHSy6sQAC1x3T9wDx6xPCioqx/0nwNgrvJnF1Oq7NFZsVpnAxaZC5BVfKSEttFPjYgv3uSfmtxeJg7pPCHmJ8qf/Sd7W7n3gKSB2BELb=="
key = "d+F3DwWj8tUckVGZb57S1XsLqfm0vnpeMEzQ2Bg/PTrohxluiJCRIYAyH6N4aKO9"
assert len(key) == 64


STANDARD_ALPHABET = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
CUSTOM_ALPHABET = key
ENCODE_TRANS = str.maketrans(STANDARD_ALPHABET, CUSTOM_ALPHABET)
DECODE_TRANS = str.maketrans(CUSTOM_ALPHABET, STANDARD_ALPHABET)

def encode(input):
  return input.translate(ENCODE_TRANS)

def decode(input):
  return input.translate(DECODE_TRANS)


b64 = decode(enc)
print(b64)
# uUgcWIFPUX0ncDNveUiCIQiAeUlRUiibfbtAZ0V6ljg+fUFChmBPbDuHLiZyUYOAeb15QGdxSqKYdjqPaNp/dCozVY1eKzltvl90dH0RjktNmWR5Y7NzyjGQw3cbb2FSEby9hrJ4T35Wj2yUtDp/FEt5toywrYtnbdF6mqcxdCU+YS6CPWNed2t8PyRlNZ9ThJJCoH1mcDvTZaJtfxmSeoy4axIYZnTASGSdDm9TlklhXQ==

hex = ', '.join([hex(b) for b in base64.b64decode(b64)])
print(hex)
# 0xb9, 0x48, 0x1c, 0x58, 0x81, 0x4f, 0x51, 0x7d, 0x27, 0x70, 0x33, 0x6f, 0x79, 0x48, 0x82, 0x21, 0x8, 0x80, 0x79, 0x49, 0x51, 0x52, 0x28, 0x9b, 0x7d, 0xbb, 0x40, 0x67, 0x45, 0x7a, 0x96, 0x38, 0x3e, 0x7d, 0x41, 0x42, 0x86, 0x60, 0x4f, 0x6c, 0x3b, 0x87, 0x2e, 0x26, 0x72, 0x51, 0x83, 0x80, 0x79, 0xbd, 0x79, 0x40, 0x67, 0x71, 0x4a, 0xa2, 0x98, 0x76, 0x3a, 0x8f, 0x68, 0xda, 0x7f, 0x74, 0x2a, 0x33, 0x55, 0x8d, 0x5e, 0x2b, 0x39, 0x6d, 0xbe, 0x5f, 0x74, 0x74, 0x7d, 0x11, 0x8e, 0x4b, 0x4d, 0x99, 0x64, 0x79, 0x63, 0xb3, 0x73, 0xca, 0x31, 0x90, 0xc3, 0x77, 0x1b, 0x6f, 0x61, 0x52, 0x11, 0xbc, 0xbd, 0x86, 0xb2, 0x78, 0x4f, 0x7e, 0x56, 0x8f, 0x6c, 0x94, 0xb4, 0x3a, 0x7f, 0x14, 0x4b, 0x79, 0xb6, 0x8c, 0xb0, 0xad, 0x8b, 0x67, 0x6d, 0xd1, 0x7a, 0x9a, 0xa7, 0x31, 0x74, 0x25, 0x3e, 0x61, 0x2e, 0x82, 0x3d, 0x63, 0x5e, 0x77, 0x6b, 0x7c, 0x3f, 0x24, 0x65, 0x35, 0x9f, 0x53, 0x84, 0x92, 0x42, 0xa0, 0x7d, 0x66, 0x70, 0x3b, 0xd3, 0x65, 0xa2, 0x6d, 0x7f, 0x19, 0x92, 0x7a, 0x8c, 0xb8, 0x6b, 0x12, 0x18, 0x66, 0x74, 0xc0, 0x48, 0x64, 0x9d, 0xe, 0x6f, 0x53, 0x96, 0x49, 0x61, 0x5d

```

解密

```c
// By txpoki
#include <stdio.h>
#include <stdlib.h>
int main(int argc, char const *argv[])
{
    char  input[] ={0xb9,0x48,0x1c,0x58,0x81,0x4f,0x51,0x7d,0x27,0x70,0x33,0x6f,0x79,0x48,0x82,0x21,0x08,0x80,0x79,0x49,0x51,0x52,0x28,0x9b,0x7d,0xbb,0x40,0x67,0x45,0x7a,0x96,0x38,0x3e,0x7d,0x41,0x42,0x86,0x60,0x4f,0x6c,0x3b,0x87,0x2e,0x26,0x72,0x51,0x83,0x80,0x79,0xbd,0x79,0x40,0x67,0x71,0x4a,0xa2,0x98,0x76,0x3a,0x8f,0x68,0xda,0x7f,0x74,0x2a,0x33,0x55,0x8d,0x5e,0x2b,0x39,0x6d,0xbe,0x5f,0x74,0x74,0x7d,0x11,0x8e,0x4b,0x4d,0x99,0x64,0x79,0x63,0xb3,0x73,0xca,0x31,0x90,0xc3,0x77,0x1b,0x6f,0x61,0x52,0x11,0xbc,0xbd,0x86,0xb2,0x78,0x4f,0x7e,0x56,0x8f,0x6c,0x94,0xb4,0x3a,0x7f,0x14,0x4b,0x79,0xb6,0x8c,0xb0,0xad,0x8b,0x67,0x6d,0xd1,0x7a,0x9a,0xa7,0x31,0x74,0x25,0x3e,0x61,0x2e,0x82,0x3d,0x63,0x5e,0x77,0x6b,0x7c,0x3f,0x24,0x65,0x35,0x9f,0x53,0x84,0x92,0x42,0xa0,0x7d,0x66,0x70,0x3b,0xd3,0x65,0xa2,0x6d,0x7f,0x19,0x92,0x7a,0x8c,0xb8,0x6b,0x12,0x18,0x66,0x74,0xc0,0x48,0x64,0x9d,0x0e,0x6f,0x53,0x96,0x49,0x61,0x5d};
    char output[180] = {};
    int v6 = 0,v8 = 1,v5 = 0x46,v7 = 0;
    char byte_140111152[] = "ixedSeed";
 do
  {
    v6 = (v6 + v8 * v5) % 256;
    v5 = byte_140111152[v7++];                  // ixedSeed
    ++v8;
  }
  while ( v5 );
    srand(v6);
    for (int i = 0; i < 178 ; i++)
    {
        output[i] = (input[i] - (rand()%128)) ^ v6;
    }
    for (int i = 0; i < 178; i++)
    {
        printf("%c",output[i]);
    }
    
    return 0;
}
```

拿到明文

```
User=Administrator
NTLM=
User=DefaultAccount
NTLM=
User=Guest
NTLM=
User=Lihua
NTLM=23d1e086b85cc18587bbc8c33adefe07
User=WDAGUtilityAccount
NTLM=d3280b38985c05214dcc81b74dd98b4f
```

最后的 `NTLM=23d1e086b85cc18587bbc8c33adefe07` 找在线网站破解

破解 https://ntlm.pw/

检验 https://www.xdtool.com/ntlm

300分，get (｀・ω・´)
