---
title: "第2章 极限与连续性"
date: 2019-04-16T16:22:46+08:00
draft: false
mathjax : ture
toc: true
---

### 2.2.2 极限法则



**定理1（极限法则）**若$L, M, c$和$k$是实数，并且

$$
\lim _{x \rightarrow c} f(x)=L, \lim _{x \rightarrow c} g(x)=M
$$
则有

（1）和法则

$$
\lim _{x \rightarrow c}(f(x)+g(x))=L+M
$$

两个函数之和的极限等于它们的极限之和。

（2） 差法则

$$
\lim _{x \rightarrow c}(f(x)-g(x))=L-M
$$

两个函数之差的极限等于它们的极限之差。

（3）积法则

$$
\lim _{x \rightarrow c}(f(x) \cdot g(x))=L \cdot M
$$

两个函数乘积的极限等于它们的极限的乘积。

（4）常数倍法则

$$
\lim _{x \rightarrow c}(k \cdot f(x))=k \cdot L
$$

常数与函数相乘的极限等于常数与函数极限的相乘。

（5）商法则

$$
\lim _{x \rightarrow c} \frac{f(x)}{g(x)}=\frac{L}{M}, \quad M \neq 0
$$

两个函数之商的极限等于它们的极限之商，假定分母的极限不为零。

（6）幂法则

若$r$和$s$是不含公因数的整数，且$s\neq 0$,则

$$
\lim _{x \rightarrow c}(f(x))^{r / s}=L^{r / s}
$$

假定$L^{r/s}$是实数，函数的有理数幂的极限等于函数极限的幂，假定极限幂是实数。


**定理2（多项式的极限）** 若$P(x)=a\_{n}x^{n}+a\_{n-1}x^{n-1}+\cdots+a_{0}$
则

$\lim \_{x \rightarrow c}P(x)=P\(c)=a\_{n} c^{n}+a\_{n-1} c^{n-1}+\cdots+a_{0}$

**定理3（有理数的极限）** 若$P(x)$和$Q(x)$是多项式，且$Q(c)\neq 0$，则
$$
\lim _{x \rightarrow c} \frac{P(x)}{Q(x)}=\frac{P( c )}{Q( c )}
$$


### 2.2.3 用代数方法消去零分母



\begin{aligned}
\lim _{x \rightarrow 1} \frac{x^{2}+x-2}{x^{2}-x} &= \frac{x^{2}+x-2}{x^{2}-x} \newline
 &= \frac{(x-1)(x+2)}{x(x-1)} \newline
 &=\frac{x+2}{x} \newline
\end{aligned}

### 2.2.5 夹层定理

**定理4（夹层定理）**假设对于包含$c$的某个开区间内的所有$x$（$x=c$本身可能除外），有$g(x) \leqslant f(x) \leqslant h(x)$，同时假设$\lim _{x \rightarrow c} g(x)=\lim _{x \rightarrow c} h(x)=L$，那么$\lim _{x \rightarrow c} f(x)=L$。

**定理5** 如果在包含$c$的某个开区间内（$x=c$本身可能除外）有$f(x) \leqslant g(x)$,且当$x \rightarrow c$时$f$和$g$都存在极限，那么
$$
\lim _{x \rightarrow c} f(x) \leqslant \lim _{x \rightarrow c} g(x)
$$

## 2.3极限的精确定义

### 2.3.1 极限的定义

**定义**令$f(x)$在围绕$x_{0}$的一个开区间上定义，在本身可能除外，如果对于每个数$\varepsilon>0$存在一个对应的数$\delta>0$使得对于所有$x$，

$$
0<\left|x-x_{0}\right|<\delta \Rightarrow|f(x)-L|<\varepsilon
$$
我们就说$f(x)$当$x \rightarrow x$时的极限$L$，并写成

$$
\lim \_{x \rightarrow x\_{0}} f(x)=L
$$




