---
title: 快速数论变换FNTT学习笔记
date: 2024-03-10 13:41:31
tags: 
mathjax: true
---

# 快速数论变换(fast number-theoretic transform,FNTT)

‎2024‎年‎1‎月‎15‎日 12:55:59 记

### 大数乘法与多项式乘法(Polynomial Multiplication)

两个大数如$12345$和$67890$，用多项式表达：

$$
f(x) = 5 x^0 +4 x^1 +3 x^2 +2 x^3 +1 x^4\\
g(x) = 0 x^0 +9 x^1 +8 x^2 +7 x^3 +6 x^4
$$

在进行多项式乘法的时候，得到$h(x)$的系数便可以变相表示为两个大整数之积

$$
h(x) = f(x) \cdot g(x)\\
h(x) = (5 x^0 +4 x^1 +3 x^2 +2 x^3 +1 x^4)(0 x^0 +9 x^1 +8 x^2 +7 x^3 +6 x^4)\\
h(x) = 0 x^0 + 45 x^1 + 76 x^2 + 94 x^3 + 100 x^4 + 70 x^5 +40 x^6 + 19 x^7 + 6 x^8
$$

转为列表

$$
[0,45,76,94,100,70,40,19,6]
$$

逆序一下，符合高位在左的书写习惯

$$
[6,19,40,70,100,94,76,45,0]
$$

进位

$$
[8,3,8,1,0,2,0,5,0]
$$

### 快速傅里叶变换(Fast Fourier Transform,FFT)

回到开始的多项式，估算原积不超过$10$位，不足$2^k$位数则用$0$补齐（方便后续递归分治）

$$
f(x) = 5 x^0 +4 x^1 +3 x^2 +2 x^3 +1 x^4 +0 x^5 + 0 x^6 +\cdots 0 x^{14} +0 x^{15}   \\
g(x) = 0 x^0 +9 x^1 +8 x^2 +7 x^3 +6 x^4 +0 x^5 + 0 x^6 +\cdots 0 x^{14} +0 x^{15}
$$

正常的乘法，需要对$f(x)$和$g(x)$的每一项都两两相乘，复杂度$O(n^2)$很高

为了描述这些多项式方程，除了采用系数表示法($n$个系数)，还可以使用描点表示法($n$个点)，例如方程$y=1x+0$可以用点$(0,0),(1,1)$唯一确定，方程$y=1x^2+2x+3$可以用点$(0,3),(1,6),(2,11)$唯一确定，即**具有$n$个系数的$(n-1)$次多项式方程可以由$n$个不同的点唯一确定**

那么对于$7$次多项式方程

$$
f(x) = a_0x^0 + a_1x^1 + a_2x^2 + a_3x^3 + a_4x^4 + a_5x^5 + a_6x^6 + a_7x^7
$$

任取$7$个

$$
x \in (x_0,x_1,x_2,x_3,x_4,x_5,x_6,x_7)
$$

得到$7$个函数值

$$
f(x_0),f(x_1),f(x_2),f(x_3),f(x_4),f(x_5),f(x_6),f(x_7)
$$

为了求这$7$个函数值，可以用矩阵方法表示(用列向量主要是因为横向量太长了Typora写不下)

$$
\begin{bmatrix}
f(x_0) \\
f(x_1) \\
f(x_2) \\
f(x_3) \\
f(x_4) \\
f(x_5) \\
f(x_6) \\
f(x_7)
\end{bmatrix}=\begin{bmatrix}
x_0^0& x_0^1& x_0^2& x_0^3& x_0^4& x_0^5& x_0^6& x_0^7\\
x_1^0& x_1^1& x_1^2& x_1^3& x_1^4& x_1^5& x_1^6& x_1^7\\
x_2^0& x_2^1& x_2^2& x_2^3& x_2^4& x_2^5& x_2^6& x_2^7\\
x_3^0& x_3^1& x_3^2& x_3^3& x_3^4& x_3^5& x_3^6& x_3^7\\
x_4^0& x_4^1& x_4^2& x_4^3& x_4^4& x_4^5& x_4^6& x_4^7\\
x_5^0& x_5^1& x_5^2& x_5^3& x_5^4& x_5^5& x_5^6& x_5^7\\
x_6^0& x_6^1& x_6^2& x_6^3& x_6^4& x_6^5& x_6^6& x_6^7\\
x_7^0& x_7^1& x_7^2& x_7^3& x_7^4& x_7^5& x_7^6& x_7^7
\end{bmatrix}\begin{bmatrix}
a_0 \\
a_1\\
a_2 \\
a_3 \\
a_4 \\
a_5 \\
a_6 \\
a_7
\end{bmatrix}
$$

如若使用编程实现该功能，复杂度$O(n^2)$依然没有优势

重新回到$f(x)$，[wiki上对奇偶函数的代数结构有这样的描述](https://zh.wikipedia.org/zh-hans/%E5%A5%87%E5%87%BD%E6%95%B8%E8%88%87%E5%81%B6%E5%87%BD%E6%95%B8)：

> 偶函数的任何线性组合皆为偶函数，且偶函数会形成一个实数上的向量空间。相似地，奇函数的任何线性组合皆为奇函数，且奇函数亦会形成一个实数上的向量空间，实际上，”所有“实值函数之向量空间为偶函数和奇函数之子空间的直和。换句话说，**每个定义域关于原点对称的函数都可以被唯一地写成一个偶函数和一个奇函数的相加**：
> $$
> f(x) = f_{even}(x) + f_{odd}(x) = \frac{f(x)+f(-x)}{2} + \frac{f(x)-f(-x)}{2}
> $$
>

类似地，将原式$f(x)$奇偶性分解
$$
f(x) = a_0x^0 + a_1x^1 + a_2x^2 + a_3x^3 + a_4x^4 + a_5x^5 + a_6x^6 + a_7x^7\\
f(x) = (a_0x^0+a_2x^2 + a_4x^4 + a_6x^6) + (a_1x^1+a_3x^3 + a_5x^5 + a_7x^7)\\
f(x) = (a_0x^0+a_2x^2 + a_4x^4 + a_6x^6) + x(a_1x^0+a_3x^2 + a_5x^4 + a_7x^6)\\
$$
记
$$
q_e(x) = a_0 x^0 + a_2 x^1 + a_4 x^2 + a_6 x^3,q_o(x) = a_1 x^0 + a_3x^1 + a_5 x^2 + a_7 x^3
$$

则有
$$
f(x) = q_e(x^2) + x q_o(x^2)\\
f(-x) = q_e(x^2) - x q_o(x^2)
$$
如此一来计算$f(x_i)$的值只需计算$q_e(x_i),q_o(x_i)$的值即可，观察到$q(x)$与$f(x)$形似，也可以使用同样的逻辑进一步拆分**分治**，使用**递归**的逻辑连接起来。

为了更好的利用奇偶性质，$x_i$的值不任取，如在本例$n=8$中选取
$$
x \in \{e^{\frac{2i\pi}{n}k}|k \in \mathbb{Z}_n,n=8\}
$$
这样一来
$$
e^{\frac{2i\pi}{n}k} = - e^{\frac{2i\pi}{n}(k+\frac{n}{2})}
$$
综上写出伪代码如下:
```python
def fft(a):
	#a = [a_0,a_1,a_2,...,a_{n-1}]
	#return [f(w^0),f(w^1),f(w^2),...,f(w^{n-1})]
	if (n==1) return [a_0]
	pe = [a_0,a_2,...,a_{n-2}]
	po = [a_1,a_3,...,a_{n-1}]
	q_o = fft(p_e)
    q_e = fft(p_e)
    w = cos(2pi/n) + i sin(2pi/n)
    for j in [0,1,2,...,n/2-1]:
    	f(w^j) = q_e[j] + w^j q_o[j]
    	f(w^{j+n/2}) = q_e[j] - w^j q_o[j]
    return [f(w^0),f(w^1),f(w^2),...,f(w^{n-1})]
```

以上，实现了从系数表示到点列表示的过程，即主要是为了实现这个计算
$$
\begin{bmatrix}
f(\omega^0) \\
f(\omega^1) \\
f(\omega^2) \\
f(\omega^3) \\
f(\omega^4) \\
f(\omega^5) \\
f(\omega^6) \\
f(\omega^7)
\end{bmatrix}=\begin{bmatrix}
\omega^{0 \cdot 0}& \omega^{0 \cdot 1}& \omega^{0 \cdot 2}& \omega^{0 \cdot 3}& \omega^{0 \cdot 4}& \omega^{0 \cdot 5}& \omega^{0 \cdot 6}& \omega^{0 \cdot 7}\\
\omega^{1 \cdot 0}& \omega^{1 \cdot 1}& \omega^{1 \cdot 2}& \omega^{1 \cdot 3}& \omega^{1 \cdot 4}& \omega^{1 \cdot 5}& \omega^{1 \cdot 6}& \omega^{1 \cdot 7}\\
\omega^{2 \cdot 0}& \omega^{2 \cdot 1}& \omega^{2 \cdot 2}& \omega^{2 \cdot 3}& \omega^{2 \cdot 4}& \omega^{2 \cdot 5}& \omega^{2 \cdot 6}& \omega^{2 \cdot 7}\\
\omega^{3 \cdot 0}& \omega^{3 \cdot 1}& \omega^{3 \cdot 2}& \omega^{3 \cdot 3}& \omega^{3 \cdot 4}& \omega^{3 \cdot 5}& \omega^{3 \cdot 6}& \omega^{3 \cdot 7}\\
\omega^{4 \cdot 0}& \omega^{4 \cdot 1}& \omega^{4 \cdot 2}& \omega^{4 \cdot 3}& \omega^{4 \cdot 4}& \omega^{4 \cdot 5}& \omega^{4 \cdot 6}& \omega^{4 \cdot 7}\\
\omega^{5 \cdot 0}& \omega^{5 \cdot 1}& \omega^{5 \cdot 2}& \omega^{5 \cdot 3}& \omega^{5 \cdot 4}& \omega^{5 \cdot 5}& \omega^{5 \cdot 6}& \omega^{5 \cdot 7}\\
\omega^{6 \cdot 0}& \omega^{6 \cdot 1}& \omega^{6 \cdot 2}& \omega^{6 \cdot 3}& \omega^{6 \cdot 4}& \omega^{6 \cdot 5}& \omega^{6 \cdot 6}& \omega^{6 \cdot 7}\\
\omega^{7 \cdot 0}& \omega^{7 \cdot 1}& \omega^{7 \cdot 2}& \omega^{7 \cdot 3}& \omega^{7 \cdot 4}& \omega^{7 \cdot 5}& \omega^{7 \cdot 6}& \omega^{7 \cdot 7}
\end{bmatrix}\begin{bmatrix}
a_0 \\
a_1\\
a_2 \\
a_3 \\
a_4 \\
a_5 \\
a_6 \\
a_7
\end{bmatrix}
$$
~~*p.s:突然发现这好像就是我当时参加的[TACA](https://zhuanlan.zhihu.com/p/550939115)的原题啊喂*~~

不妨把这个大矩阵记作$W$
$$
W = (\omega^{ij})_{i,j = 0,1,\cdots,n-1},\omega = e^{\frac{2i\pi}{n}}
$$


想要实现从点列表示转成系数表示，就需要求得这个$W$矩阵的逆矩阵$W^{-1}$

### 离散傅里叶变换矩阵(Discrete Fourier Transform matrix,DFTmtx)

>  **离散傅立叶变换矩阵是将离散傅立叶变换以矩阵乘法来表达的一种表示式**

DFT矩阵定义
$$
W = (\omega^{ij})_{i,j = 0,1,\cdots,N-1}/\sqrt{N},\omega = e^{\frac{-2i\pi}{N}}\\
$$
或
$$
W=
\frac{1}{\sqrt{N}}\begin{bmatrix}
\omega ^{0 \cdot 0}& \omega ^{0 \cdot 1}& \cdots & \omega ^{0 \cdot (N-1)}\\
\omega ^{1 \cdot 0}& \omega ^{1 \cdot 1}& \cdots & \omega ^{1 \cdot (N-1)}\\
\vdots& \vdots & \ddots & \vdots \\
\omega ^{(N-1) \cdot 0}& \omega ^{(N-1) \cdot 1}& \cdots & \omega ^{(N-1) \cdot (N-1)}\\
\end{bmatrix},
\omega  = e^{\frac{-2i\pi}{N}}
$$

*注意：此处$\omega$在指数上是否存在负号区别于之前的$\omega$，同时此处矩阵$W$有正规化因数$\frac{1}{\sqrt{N}}$*

其逆矩阵是
$$
T=
\frac{1}{\sqrt{N}}\begin{bmatrix}
\tau ^{0 \cdot 0}& \tau ^{0 \cdot 1}& \cdots & \tau ^{0 \cdot (N-1)}\\
\tau ^{1 \cdot 0}& \tau ^{1 \cdot 1}& \cdots & \tau ^{1 \cdot (N-1)}\\
\vdots& \vdots & \ddots & \vdots \\
\tau ^{(N-1) \cdot 0}& \tau ^{(N-1) \cdot 1}& \cdots & \tau ^{(N-1) \cdot (N-1)}\\
\end{bmatrix},
\tau  = e^{\frac{2i\pi}{N}}=\omega ^{-1}
$$
下验证$WT=I$
$$
\begin{align}
N(WT)_{ij} 
&=  N \sum_{k=1}^{N}{W_{ik}T_{kj}}\\
&= \sum_{k=1}^{N}{\omega^{ik}\tau^{kj}}\\
&= \sum_{k=1}^{N}{\omega^{ik}\omega^{-kj}}\\
&= \sum_{k=1}^{N}{\omega^{k(i-j)}}
\end{align}
$$
当且仅当$i=j$时有
$$
\begin{align}
N(WT)_{ij} 
&= \sum_{k=1}^{N}{\omega^{k(i-j)}}\\
&= \sum_{k=1}^{N}{\omega^{0}}\\
&= N\\
\therefore (WT)_{ij}=1,i = j
\end{align}
$$
当$i \ne j$时有
$$
\begin{align}
N(WT)_{ij} 
&= \sum_{k=1}^{N}{\omega^{k(i-j)}}\\
&= \sum_{k=1}^{\frac{N}{2}}{\omega^{k(i-j)}} + \sum_{k=1}^{\frac{N}{2}}{\omega^{(k+\frac{N}{2})(i-j)}}\\
&= \sum_{k=1}^{\frac{N}{2}}{e^{\frac{-2i\pi}{N}k(i-j)}} + \sum_{k=1}^{\frac{N}{2}}{e^{\frac{-2i\pi}{N}k(i-j)-i\pi k(i-j)}}\\
&= \sum_{k=1}^{\frac{N}{2}}{e^{\frac{-2i\pi}{N}k(i-j)}} - \sum_{k=1}^{\frac{N}{2}}{e^{\frac{-2i\pi}{N}k(i-j)}}\\
&= 0\\
\therefore (WT)_{ij}=0,i \ne j
\end{align}
$$
至此$WT=I$验证完毕

### 快速傅里叶逆变换(Inverse Fast Fourier Transform,IFFT)

根据上面的式子，我们得到了
$$
\begin{bmatrix}
\omega^{0 \cdot 0}& \omega^{0 \cdot 1}& \omega^{0 \cdot 2}& \omega^{0 \cdot 3}& \omega^{0 \cdot 4}& \omega^{0 \cdot 5}& \omega^{0 \cdot 6}& \omega^{0 \cdot 7}\\
\omega^{1 \cdot 0}& \omega^{1 \cdot 1}& \omega^{1 \cdot 2}& \omega^{1 \cdot 3}& \omega^{1 \cdot 4}& \omega^{1 \cdot 5}& \omega^{1 \cdot 6}& \omega^{1 \cdot 7}\\
\omega^{2 \cdot 0}& \omega^{2 \cdot 1}& \omega^{2 \cdot 2}& \omega^{2 \cdot 3}& \omega^{2 \cdot 4}& \omega^{2 \cdot 5}& \omega^{2 \cdot 6}& \omega^{2 \cdot 7}\\
\omega^{3 \cdot 0}& \omega^{3 \cdot 1}& \omega^{3 \cdot 2}& \omega^{3 \cdot 3}& \omega^{3 \cdot 4}& \omega^{3 \cdot 5}& \omega^{3 \cdot 6}& \omega^{3 \cdot 7}\\
\omega^{4 \cdot 0}& \omega^{4 \cdot 1}& \omega^{4 \cdot 2}& \omega^{4 \cdot 3}& \omega^{4 \cdot 4}& \omega^{4 \cdot 5}& \omega^{4 \cdot 6}& \omega^{4 \cdot 7}\\
\omega^{5 \cdot 0}& \omega^{5 \cdot 1}& \omega^{5 \cdot 2}& \omega^{5 \cdot 3}& \omega^{5 \cdot 4}& \omega^{5 \cdot 5}& \omega^{5 \cdot 6}& \omega^{5 \cdot 7}\\
\omega^{6 \cdot 0}& \omega^{6 \cdot 1}& \omega^{6 \cdot 2}& \omega^{6 \cdot 3}& \omega^{6 \cdot 4}& \omega^{6 \cdot 5}& \omega^{6 \cdot 6}& \omega^{6 \cdot 7}\\
\omega^{7 \cdot 0}& \omega^{7 \cdot 1}& \omega^{7 \cdot 2}& \omega^{7 \cdot 3}& \omega^{7 \cdot 4}& \omega^{7 \cdot 5}& \omega^{7 \cdot 6}& \omega^{7 \cdot 7}
\end{bmatrix}\begin{bmatrix}
\tau^{0 \cdot 0}& \tau^{0 \cdot 1}& \tau^{0 \cdot 2}& \tau^{0 \cdot 3}& \tau^{0 \cdot 4}& \tau^{0 \cdot 5}& \tau^{0 \cdot 6}& \tau^{0 \cdot 7}\\
\tau^{1 \cdot 0}& \tau^{1 \cdot 1}& \tau^{1 \cdot 2}& \tau^{1 \cdot 3}& \tau^{1 \cdot 4}& \tau^{1 \cdot 5}& \tau^{1 \cdot 6}& \tau^{1 \cdot 7}\\
\tau^{2 \cdot 0}& \tau^{2 \cdot 1}& \tau^{2 \cdot 2}& \tau^{2 \cdot 3}& \tau^{2 \cdot 4}& \tau^{2 \cdot 5}& \tau^{2 \cdot 6}& \tau^{2 \cdot 7}\\
\tau^{3 \cdot 0}& \tau^{3 \cdot 1}& \tau^{3 \cdot 2}& \tau^{3 \cdot 3}& \tau^{3 \cdot 4}& \tau^{3 \cdot 5}& \tau^{3 \cdot 6}& \tau^{3 \cdot 7}\\
\tau^{4 \cdot 0}& \tau^{4 \cdot 1}& \tau^{4 \cdot 2}& \tau^{4 \cdot 3}& \tau^{4 \cdot 4}& \tau^{4 \cdot 5}& \tau^{4 \cdot 6}& \tau^{4 \cdot 7}\\
\tau^{5 \cdot 0}& \tau^{5 \cdot 1}& \tau^{5 \cdot 2}& \tau^{5 \cdot 3}& \tau^{5 \cdot 4}& \tau^{5 \cdot 5}& \tau^{5 \cdot 6}& \tau^{5 \cdot 7}\\
\tau^{6 \cdot 0}& \tau^{6 \cdot 1}& \tau^{6 \cdot 2}& \tau^{6 \cdot 3}& \tau^{6 \cdot 4}& \tau^{6 \cdot 5}& \tau^{6 \cdot 6}& \tau^{6 \cdot 7}\\
\tau^{7 \cdot 0}& \tau^{7 \cdot 1}& \tau^{7 \cdot 2}& \tau^{7 \cdot 3}& \tau^{7 \cdot 4}& \tau^{7 \cdot 5}& \tau^{7 \cdot 6}& \tau^{7 \cdot 7}
\end{bmatrix}=nI
$$


那么回到最先的式子
$$
\begin{bmatrix}
f(\omega^0) \\
f(\omega^1) \\
f(\omega^2) \\
f(\omega^3) \\
f(\omega^4) \\
f(\omega^5) \\
f(\omega^6) \\
f(\omega^7)
\end{bmatrix}
=
\begin{bmatrix}
\omega^{0 \cdot 0}& \omega^{0 \cdot 1}& \omega^{0 \cdot 2}& \omega^{0 \cdot 3}& \omega^{0 \cdot 4}& \omega^{0 \cdot 5}& \omega^{0 \cdot 6}& \omega^{0 \cdot 7}\\
\omega^{1 \cdot 0}& \omega^{1 \cdot 1}& \omega^{1 \cdot 2}& \omega^{1 \cdot 3}& \omega^{1 \cdot 4}& \omega^{1 \cdot 5}& \omega^{1 \cdot 6}& \omega^{1 \cdot 7}\\
\omega^{2 \cdot 0}& \omega^{2 \cdot 1}& \omega^{2 \cdot 2}& \omega^{2 \cdot 3}& \omega^{2 \cdot 4}& \omega^{2 \cdot 5}& \omega^{2 \cdot 6}& \omega^{2 \cdot 7}\\
\omega^{3 \cdot 0}& \omega^{3 \cdot 1}& \omega^{3 \cdot 2}& \omega^{3 \cdot 3}& \omega^{3 \cdot 4}& \omega^{3 \cdot 5}& \omega^{3 \cdot 6}& \omega^{3 \cdot 7}\\
\omega^{4 \cdot 0}& \omega^{4 \cdot 1}& \omega^{4 \cdot 2}& \omega^{4 \cdot 3}& \omega^{4 \cdot 4}& \omega^{4 \cdot 5}& \omega^{4 \cdot 6}& \omega^{4 \cdot 7}\\
\omega^{5 \cdot 0}& \omega^{5 \cdot 1}& \omega^{5 \cdot 2}& \omega^{5 \cdot 3}& \omega^{5 \cdot 4}& \omega^{5 \cdot 5}& \omega^{5 \cdot 6}& \omega^{5 \cdot 7}\\
\omega^{6 \cdot 0}& \omega^{6 \cdot 1}& \omega^{6 \cdot 2}& \omega^{6 \cdot 3}& \omega^{6 \cdot 4}& \omega^{6 \cdot 5}& \omega^{6 \cdot 6}& \omega^{6 \cdot 7}\\
\omega^{7 \cdot 0}& \omega^{7 \cdot 1}& \omega^{7 \cdot 2}& \omega^{7 \cdot 3}& \omega^{7 \cdot 4}& \omega^{7 \cdot 5}& \omega^{7 \cdot 6}& \omega^{7 \cdot 7}
\end{bmatrix}
\begin{bmatrix}
a_0 \\
a_1\\
a_2 \\
a_3 \\
a_4 \\
a_5 \\
a_6 \\
a_7
\end{bmatrix}\\
$$
可以有
$$
F=WA\\
W^{-1}F=W^{-1}WA\\
A = W^{-1}F\\
A = \frac{1}{n}TF
$$
即
$$
\begin{bmatrix}
a_0 \\
a_1\\
a_2 \\
a_3 \\
a_4 \\
a_5 \\
a_6 \\
a_7
\end{bmatrix}\\
=\frac{1}{n}
\begin{bmatrix}
\tau^{0 \cdot 0}& \tau^{0 \cdot 1}& \tau^{0 \cdot 2}& \tau^{0 \cdot 3}& \tau^{0 \cdot 4}& \tau^{0 \cdot 5}& \tau^{0 \cdot 6}& \tau^{0 \cdot 7}\\
\tau^{1 \cdot 0}& \tau^{1 \cdot 1}& \tau^{1 \cdot 2}& \tau^{1 \cdot 3}& \tau^{1 \cdot 4}& \tau^{1 \cdot 5}& \tau^{1 \cdot 6}& \tau^{1 \cdot 7}\\
\tau^{2 \cdot 0}& \tau^{2 \cdot 1}& \tau^{2 \cdot 2}& \tau^{2 \cdot 3}& \tau^{2 \cdot 4}& \tau^{2 \cdot 5}& \tau^{2 \cdot 6}& \tau^{2 \cdot 7}\\
\tau^{3 \cdot 0}& \tau^{3 \cdot 1}& \tau^{3 \cdot 2}& \tau^{3 \cdot 3}& \tau^{3 \cdot 4}& \tau^{3 \cdot 5}& \tau^{3 \cdot 6}& \tau^{3 \cdot 7}\\
\tau^{4 \cdot 0}& \tau^{4 \cdot 1}& \tau^{4 \cdot 2}& \tau^{4 \cdot 3}& \tau^{4 \cdot 4}& \tau^{4 \cdot 5}& \tau^{4 \cdot 6}& \tau^{4 \cdot 7}\\
\tau^{5 \cdot 0}& \tau^{5 \cdot 1}& \tau^{5 \cdot 2}& \tau^{5 \cdot 3}& \tau^{5 \cdot 4}& \tau^{5 \cdot 5}& \tau^{5 \cdot 6}& \tau^{5 \cdot 7}\\
\tau^{6 \cdot 0}& \tau^{6 \cdot 1}& \tau^{6 \cdot 2}& \tau^{6 \cdot 3}& \tau^{6 \cdot 4}& \tau^{6 \cdot 5}& \tau^{6 \cdot 6}& \tau^{6 \cdot 7}\\
\tau^{7 \cdot 0}& \tau^{7 \cdot 1}& \tau^{7 \cdot 2}& \tau^{7 \cdot 3}& \tau^{7 \cdot 4}& \tau^{7 \cdot 5}& \tau^{7 \cdot 6}& \tau^{7 \cdot 7}
\end{bmatrix}
\begin{bmatrix}
f(\omega^0) \\
f(\omega^1) \\
f(\omega^2) \\
f(\omega^3) \\
f(\omega^4) \\
f(\omega^5) \\
f(\omega^6) \\
f(\omega^7)
\end{bmatrix}
$$
由于
$$
\omega \tau=1
$$
这意味着先前得出的FFT逻辑完全可以把所有的$\omega$换成$\tau$便可完成IFFT的过程，再一次实现函数复用

现在剩余一个问题，$\omega$是一个复数，在$n$不大的情况下计算误差不大，但是当$n$很大，计算误差和三角函数计算时间都是问题，于是要引入有限域

### 有限域(finite field)

一点儿没学明白，查资料也写不出来多少$QAQ$

大致运算：

如在$\mathrm{mod} \space 5 $下，
$$
1+1 = 2\\
2+4 = 1\\
3-4 = 4\\
3\times 4 = 2\\
3\times 2 = 1\\
3^{-1} = 2\\
4/3 = 4\times 2 = 3\\
3\times 3 = 4\\
2\times 2 = 4\\
\sqrt{4} = 2,3
$$
根据此类性质，可以写出C函数如下

```c
typedef long long int lli;
lli ZmodAdd(lli a, lli b){
    a += b;
    return (a<MOD_P) ? a : (a-MOD_P);
}

lli ZmodSub(lli a, lli b){
    a -= b;
    return (a>=0) ? a : (a+MOD_P);
}

lli ZmodMul(lli a, lli b){
    return (a * b) % MOD_P;
}

lli ZmodInv(lli a) {
    //根据费马小定理，乘法逆元唯一存在
    //利用扩展欧几里得算法求乘法逆元
    lli x,y;
    gcdext(a,MOD_P,&x,&y);
    //x mod p 即为所求逆元，x的范围是正负p/2
    return (x>=0) ? x : (x+MOD_P);
}

lli ZmodDiv(lli a, lli b) {
    return (a * ZmodInv(b)) % MOD_P;
}

lli gcdext(lli a,lli b,lli *x,lli *y){  //扩展欧几里得算法，网上抄的
    if(b==0){
        *x=1;
        *y=0;
        return a;
    }
    lli ret=gcdext(b,a%b,x,y);
    lli t=*x;
    *x=*y;
    *y=t-a/b*(*y);
    return ret;
}

```



### 快速数论变换(fast number-theoretic transform,FNTT)

先前FFT中选取$\omega = e^{\frac{2i\pi}{n}}$的性质主要有三：

1. 所有计算除了系数$a_i$只需要计算$\omega ^{k}$即可，便于计算
2. $\omega ^{k+\frac{n}{2}}=-\omega ^{k},\omega^{n}=1$
3. $\omega ^{k},k\in Z_{n}$各不相同

而在一个有限域内，也同样有数可以满足这两条特性，如$p=84906529,\omega =213016,n=16$，$\omega^k \pmod{p} $的各值如下
$$
\omega^{0}\equiv 1\\
\omega^{1}\equiv 213016\\
\omega^{2}\equiv 35729770\\
\omega^{3}\equiv 76333289\\
\omega^{4}\equiv 17240421\\
\omega^{5}\equiv 23420899\\
\omega^{6}\equiv 3483873\\
\omega^{7}\equiv 37627508\\
\omega^{8}\equiv 84906528\equiv -1\\
\omega^{9}\equiv 84693513\\
\omega^{10}\equiv 49176759\\
\omega^{11}\equiv 8573240\\
\omega^{12}\equiv 67666108\\
\omega^{13}\equiv 61485630\\
\omega^{14}\equiv 81422656\\
\omega^{15}\equiv 47279021\\
\omega^{16}\equiv 1
$$
这样的$\omega$也同样符合先前的性质，因此不妨把FFT中的所有运算都转为模$p$数域下的运算，$\omega$换成整数的$\omega$

```c
int FNTT(int *pa,int w,List a,int *f){ //FNTT快速数论变换（基于FFT快速傅里叶变换）
    if (a.n == 1){
        f[0] = pa[a.begin];
        return 0;
    }
    int nOver2 = a.n>>1;
    int stepBy2 = a.step<<1;
    List pe = createList(a.begin, stepBy2, nOver2);
    List po = createList(a.begin+a.step, stepBy2, nOver2);
    int qe[nOver2];
    int qo[nOver2];
    FNTT(pa, ZmodMul(w,w), pe, qe);
    FNTT(pa, ZmodMul(w,w), po, qo);
    int t = 1;
    int qo_t;
    for (int j=0; j<nOver2; j++){
        qe_t = ZmodMul(qo[j],t);
        f[j] = ZmodAdd(qe[j] , qo_t);
        f[j+nOver2] = ZmodSub(qe[j] , qo_t);
        t = ZmodMul(w,t);
    }
    return 0;
}

```

那么如何选择$p,\omega$？

对于质数$p = q2^m+1$，原根$g$满足$g^{q2^k} \equiv 1\pmod{p}$，将$g_n \equiv g^q \pmod{p}$看作$\omega$的等价，则其满足相似的性质

~~*p.s:数论知识好多，不会证明;w;*~~

综上，写成完整C语言代码如下

```c
#include <stdio.h>
#define MOD_P 84906529
typedef long long int lli;

typedef struct {
    int begin;
    int step;
    int n;
}List;
List createList(int begin,int step,int n) {
    List newList;
    newList.begin = begin;
    newList.step = step;
    newList.n = n;
    return newList;
}

//Zmod(p)数域计算（c语言好像没有重载运算符，所以表达式看起来会比较麻烦~）
lli ZmodAdd(lli a, lli b);
lli ZmodSub(lli a, lli b);
lli ZmodMul(lli a, lli b);
lli ZmodInv(lli a);
lli ZmodDiv(lli a, lli b);
lli gcdext(lli a,lli b,lli *x,lli *y);  //扩展欧几里得算法

int FNTT(int *pa,int w,List a,int *f){ //FNTT快速数论变换（基于FFT快速傅里叶变换）
    if (a.n == 1){
        f[0] = pa[a.begin];
        return 0;
    }
    int nOver2 = a.n>>1;
    int stepBy2 = a.step<<1;
    List pe = createList(a.begin, stepBy2, nOver2);
    List po = createList(a.begin+a.step, stepBy2, nOver2);
    int qe[nOver2];
    int qo[nOver2];
    FNTT(pa, ZmodMul(w,w), pe, qe);
    FNTT(pa, ZmodMul(w,w), po, qo);
    int t = 1;
    int qo_t;
    for (int j=0; j<nOver2; j++){
        qe_t = ZmodMul(qo[j],t);
        f[j] = ZmodAdd(qe[j] , qo_t);
        f[j+nOver2] = ZmodSub(qe[j] , qo_t);
        t = ZmodMul(w,t);
    }
    return 0;
}


int main() {
    int w = 213016;
    int t = ZmodInv(w);
    int f[16]     = {5,4,3,2,1,0,0,0,0,0,0,0,0,0,0,0};
    int g[16]     = {1,2,3,4,5,0,0,0,0,0,0,0,0,0,0,0};
    int ntt_f[16] = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0};
    int ntt_g[16] = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0};
    int h[16]     = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0};
    int r[16]     = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0};
    List start = createList(0,1,16);
    FNTT(f,w,start,ntt_f);
    FNTT(g,w,start,ntt_g);
    for (int i=0; i<16; i++){
        h[i] = ZmodMul(ntt_f[i],ntt_g[i]);
    }
    FNTT(h,t,start,r);
    for (int i=0; i<16; i++){
        r[i] = ZmodDiv(r[i],16);
    }
    for (int i=0; i<16; i++){
        printf("%d ",r[i]);
    }
    return 0;
}

lli ZmodAdd(lli a, lli b){
    a += b;
    return (a<MOD_P) ? a : (a-MOD_P);
}

lli ZmodSub(lli a, lli b){
    a -= b;
    return (a>=0) ? a : (a+MOD_P);
}

lli ZmodMul(lli a, lli b){
    return (a * b) % MOD_P;
}

lli ZmodInv(lli a) {
    //根据费马小定理，乘法逆元唯一存在
    //利用扩展欧几里得算法求乘法逆元
    lli x,y;
    gcdext(a,MOD_P,&x,&y);
    //x mod p 即为所求逆元，x的范围是正负p/2
    return (x>=0) ? x : (x+MOD_P);
}

lli ZmodDiv(lli a, lli b) {
    return (a * ZmodInv(b)) % MOD_P;
}

lli gcdext(lli a,lli b,lli *x,lli *y){  //扩展欧几里得算法，网上抄的
    if(b==0){
        *x=1;
        *y=0;
        return a;
    }
    lli ret=gcdext(b,a%b,x,y);
    lli t=*x;
    *x=*y;
    *y=t-a/b*(*y);
    return ret;
}

```

### 参考学习

[超硬核FFT快速傅里叶变换讲解，高效进行高精度乘法运算！ - Bilibili](https://www.bilibili.com/video/BV1gY411d7Z1/)

[最讨厌做算法题的时候出现浮点数了，有什么方法可以避免？斐波那契数列和FFT均可适用！ - Bilibili](https://www.bilibili.com/video/BV1EM41117Dv/)

[ChatGPT](https://chat.openai.com/)

[快速数论变换 - OI wiki](https://oi-wiki.org/math/poly/ntt/)

[Finite field - Wikipedia](https://en.wikipedia.org/wiki/Finite_field/)

[离散傅里叶变换矩阵 - Wikipedia](https://zh.wikipedia.org/zh-cn/%E9%9B%A2%E6%95%A3%E5%82%85%E9%87%8C%E8%91%89%E8%AE%8A%E6%8F%9B%E7%9F%A9%E9%99%A3)

[奇函数与偶函数 - Wikipedia](https://zh.wikipedia.org/zh-cn/%E5%A5%87%E5%87%BD%E6%95%B8%E8%88%87%E5%81%B6%E5%87%BD%E6%95%B8#%E4%BB%A3%E6%95%B8%E7%B5%90%E6%A7%8B)
