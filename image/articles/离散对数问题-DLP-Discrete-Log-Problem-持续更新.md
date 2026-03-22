---
title: 离散对数问题(DLP,Discrete Log Problem)持续更新
date: 2024-05-05 20:59:44
tags:
---

## 写在前面

通常习惯把各种链接资料放在文章末尾，但是这次有很多很多很好的资料，想先写在前面

- 对sage各种DLP方法的介绍[离散对数](https://lazzzaro.github.io/2020/05/07/crypto-%E7%A6%BB%E6%95%A3%E5%AF%B9%E6%95%B0/)

- pohlig-hellman算法(适用于p-1光滑)很细致的例子 [离散对数问题——pohlig-hellman算法讲解(有例子)](https://blog.csdn.net/oampamp1/article/details/104061969)

目前我学习的ECC相关还少，下面的内容基本只涉及普通有限域的内容，

## [sage函数](https://lazzzaro.github.io/2020/05/07/crypto-%E7%A6%BB%E6%95%A3%E5%AF%B9%E6%95%B0/#%E5%B8%B8%E8%A7%84Sage%E5%87%BD%E6%95%B0)

参数说明：`base`底，`a`的对数，`ord`为`base`的阶，可以缺省，`operation`可以是`+`与`*`，默认为`*`；`bounds`是一个区间`(ld, ud)`需要保证所计算的对数在此去区间内。

- 通用的方法： `discrete_log(a,base,ord,operation)`
- Pollard-Rho算法：`discrete_log_rho(a,base,ord,operation)`
- Pollard-kangaroo算法（lambda算法）：`discrete_log_lambda(a,base,bounds,operation)`
- 小步大步法(Baby Step Giant Step)：`bsgs(base,a,bounds,operation)`

## 一些🌰

### 秒解的dlp

##### [XYCTF 2024]Complex_dlp

**题目附件**

```python
from Crypto.Util.number import *
from secrets import flag


class Complex:
    def __init__(self, re, im):
        self.re = re
        self.im = im

    def __mul__(self, c):
        re_ = self.re * c.re - self.im * c.im
        im_ = self.re * c.im + self.im * c.re
        return Complex(re_, im_)

    def __str__(self):
        if self.im == 0:
            return str(self.re)
        elif self.re == 0:
            if abs(self.im) == 1:
                return f"{'-' if self.im < 0 else ''}i"
            else:
                return f"{self.im}i"
        else:
            return f"{self.re} {'+' if self.im > 0 else '-'} {abs(self.im)}i"


def complex_pow(c, exp, n):
    result = Complex(1, 0)
    while exp > 0:
        if exp & 1:
            result = result * c
            result.re = result.re % n
            result.im = result.im % n
        c = c * c
        c.re = c.re % n
        c.im = c.im % n
        exp >>= 1
    return result


flag = flag.strip(b"XYCTF{").strip(b"}")
p = 1127236854942215744482170859284245684922507818478439319428888584898927520579579027
g = Complex(3, 7)
x = bytes_to_long(flag)
print(complex_pow(g, x, p))
# 5699996596230726507553778181714315375600519769517892864468100565238657988087817 + 198037503897625840198829901785272602849546728822078622977599179234202360717671908i
```

复数域下的`DLP`问题

$$
(3+7i)^x \equiv y  \pmod{p}
$$

尝试过从模**长**、辐角和实部虚部自身关系入手，实在难以处理 $\mod{p}$ 这个条件:

自己写一段小数尝试找规律：

```python
g = Complex(3,7)
p = 17
for x in range(64):
    y = complex_pow(g,x,p)
    re,im = y.re, y.im
    print(f"{x = } \t {str(y)} \t {re**2 + im**2} \t {(re**2 + im**2) % p}")

# x = 0    1               1       1
# x = 1    3 + 7i          58      7
# x = 2    11 + 8i         185     15
# x = 3    11 + 16i        377     3
# x = 4    6 + 6i          72      4
# x = 5    10 + 9i         181     11
# x = 6    1 + 12i         145     9
# x = 7    4 + 9i          97      12
# x = 8    4i              16      16
# x = 9    6 + 12i         180     10
# x = 10   2 + 10i         104     2
# x = 11   4 + 10i         116     14
# x = 12   10 + 7i         149     13
# x = 13   15 + 6i         261     6
# x = 14   3 + 4i          25      8
# x = 15   15 + 16i        481     5
# x = 16   1               1       1
# x = 17   3 + 7i          58      7
# x = 18   11 + 8i         185     15
# x = 19   11 + 16i        377     3
# x = 20   6 + 6i          72      4
# x = 21   10 + 9i         181     11
# x = 22   1 + 12i         145     9
# x = 23   4 + 9i          97      12
# x = 24   4i              16      16
# x = 25   6 + 12i         180     10
# x = 26   2 + 10i         104     2
# x = 27   4 + 10i         116     14
# x = 28   10 + 7i         149     13
# x = 29   15 + 6i         261     6
# x = 30   3 + 4i          25      8
# x = 31   15 + 16i        481     5
# x = 32   1               1       1
# x = 33   3 + 7i          58      7
# x = 34   11 + 8i         185     15
# x = 35   11 + 16i        377     3
# x = 36   6 + 6i          72      4
# x = 37   10 + 9i         181     11
# x = 38   1 + 12i         145     9
# x = 39   4 + 9i          97      12
# x = 40   4i              16      16
# x = 41   6 + 12i         180     10
# x = 42   2 + 10i         104     2
# x = 43   4 + 10i         116     14
# x = 44   10 + 7i         149     13
# x = 45   15 + 6i         261     6
# x = 46   3 + 4i          25      8
# x = 47   15 + 16i        481     5
# x = 48   1               1       1
# x = 49   3 + 7i          58      7
# x = 50   11 + 8i         185     15
# x = 51   11 + 16i        377     3
# x = 52   6 + 6i          72      4
# x = 53   10 + 9i         181     11
# x = 54   1 + 12i         145     9
# x = 55   4 + 9i          97      12
# x = 56   4i              16      16
# x = 57   6 + 12i         180     10
# x = 58   2 + 10i         104     2
# x = 59   4 + 10i         116     14
# x = 60   10 + 7i         149     13
# x = 61   15 + 6i         261     6
# x = 62   3 + 4i          25      8
# x = 63   15 + 16i        481     5
```

复数$$y_i$$有明显的循环结构，但是其阶和 $p$ 的关系不明确，瞎猜与 $\varphi(p) = p^2 - 1$ 有关（？）

但是模**长**平方再模**除**的循环结构很明确，不完全归纳猜测其阶即为 $p-1$

简单证明一下：

记某复数 $c$ 及其共轭为：

$$
c = (a+bi)\\ 
\bar{c} = (a-bi)
$$

则二者与模**长**平方的关系有：

$$
c \bar{c} = \bar{c} c = a^2 - b^2 i^2 = a^2 + b^2
$$

取正整数 $n$ 作为指数：

$$
(c\bar{c})^n = (\bar{c} c)^n =c^n \bar{c} ^ n = \bar{c} ^ n c^n = (a^2 + b^2)^n
$$

关键的一个关系：

$$
(c\bar{c})^n =c^n \bar{c} ^ n
$$

其中  $(c\bar{c}),(c^n \bar{c}^n) \in \mathbb{Z}$，就可以应用模**除**了

```Python
p = 1127236854942215744482170859284245684922507818478439319428888584898927520579579027
re = 5699996596230726507553778181714315375600519769517892864468100565238657988087817
im = 198037503897625840198829901785272602849546728822078622977599179234202360717671908
Zp = Zmod(p)
g = Zp(3^2 + 7^2)
y = Zp(re ^ 2 + im ^ 2)
print("g =", g)
print("y =", y)
print("g ^ x = y")
# g = 58
# y = 440338354863477089588878295048682566041450520053821907911657559153659930236947297
# g ^ x = y
```



最后扔给`discrete_log`解之：

```Python
from Crypto.Util.number import *
p = 1127236854942215744482170859284245684922507818478439319428888584898927520579579027
Zp = Zmod(p)
g = Zp(58)
y = Zp(440338354863477089588878295048682566041450520053821907911657559153659930236947297)
# g ^ x = y
x = discrete_log(y,g)
print("x =",x)
print(long_to_bytes(x))
# x = 11043386733210093783917591062922767739440701223101839768310052590794700044066655
# b'___c0mp13x_d1p_15_3@5y_f0r_y0u___'
```

**秒解的原因猜测** —— $(p-1)$ 光滑，可用pohlig-hellman算法

```Python
p = 1127236854942215744482170859284245684922507818478439319428888584898927520579579027
print(factor(p-1))
# 2 * 269 * 1787 * 12923 * 13513 * 25127 * 26267 * 31957 * 49921 * 57163 * 60101 * 71353 * 73039 * 73091 * 75193 * 76163 * 88001 * 96233 * 100469
```

### 需要
