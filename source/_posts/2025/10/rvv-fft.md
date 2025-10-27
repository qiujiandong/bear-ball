---
title: RISC-V Vector优化Radix-2 FFT
date: 2025-10-27 17:29:30
tags:
  - RISC-V
  - FFT
  - RVV
categories:
  - RISC-V
---

## FFT基础

离散傅里叶变换：

$$X[k] = \sum\limits_{n = 0}^{N - 1} {x[n]{e^{ - j{2\pi nk} \over N}}
= \sum\limits_{n = 0}^{N - 1} {x[n]W_N^{nk}} } $$

快速傅里叶变换(FFT)常见的可以分为DIT/DIF，即按时域分解或者按频域分解。以按频域分解为例：

![fft_dif](fft_dif.svg)

写出偶数频率项和奇数频率项：

$$X[2r] = \sum\limits_{n = 0}^{N - 1} {x[n]W_N^{2nr}} =
\sum\limits_{n = 0}^{\frac{N}{2} - 1} {x[n]W_N^{2rn}} +
\sum\limits_{n = 0}^{\frac{N}{2} - 1}
{x[n + \frac{N}{2}]W_N^{2r[n + \frac{N}{2}]}} =
\sum\limits_{n = 0}^{\frac{N}{2} - 1} {(x[n] + x[n + \frac{N}{2}])W_{\frac{N}{2}}^{rn}}$$

$$X[2r + 1] = \sum\limits_{n = 0}^{\frac{N}{2} - 1}
{(x[n] - x[n + \frac{N}{2}])W_N^nW_{\frac{N}{2}}^{rn}} $$

从这两个表达式来看，长度为N的DFT可以分解成两个长度为N/2的DFT.

而且从分解的形式看，每次都是将偶数部分和奇数部分拆分开,
结果自然就不会是连续的, 而是形成位倒序的排列,
由此也可以理解位倒序正是和radix-2息息相关，
如果是radix-3或者radix-4, 输出的顺序就不完全是位倒序了。

## 向量化处理

可以看到DIF FFT的计算中每个stage的FFT长度是逐级减小的,
这不太利于发挥SIMD指令的性能, 因此略微调整数据的摆放位置,
可以使得中间的计算结果都能连续存放。

原先的DIF FFT的计算过程有一个好处是可以原位计算,
但是经过如下的调整后, 就不能原位计算了, 但是可以实现中间每层的计算都以N/2的向量长度处理。

![fft_rvv](fft_rvv.svg)

## 实数FFT

先说结论，N点的实数FFT只两个N/4长度的序列就可以算出N/2点结果，而且因为完整的结果是共轭对称的,
所以一般剩下N/2点就不用写了。

### 正变换

实数序列x[n], n=0, 1, 2, ..., N-1，可以根据索引值的奇偶性进行分组。

偶数索引值的数组成一个新的实数序列f[u]，u=0, 1, 2, ..., N/2-1

$$f[u] = x[2u]$$

x[n]中奇数索引值的数组成新的实数序列g[u]，长度为N/2。

$$g[u] = x[2u+1]$$

把x[n]看作一个复数序列y[u]，偶数索引值的数作为实部，奇数索引值的数作为虚部。

$$y[u] = f[u] + jg[u]$$

两边都做离散傅里叶变换，根据线性性质可以得到：

$$Y[r] = F[r] + jG[r]$$

实数的离散傅里叶变换具有共轭对称性。

$$X[r] = \overline {X[N - r]} $$

只需要把公式代入下面的分析式中就可以轻松验证

$$X[r] = \sum\limits_{n = 0}^{N - 1} {x[n]{e^{ -j2\pi \frac{rn}{N}}}}$$

$F[r]$是$f[u]$的离散傅里叶变换结果，$G[r]$是$g[u]$的离散傅里叶变换结果,
$Y[r]$是$y[u]$的离散傅里叶变换结果。因为$f[u]$, $g[u]$是实数，所以：

$$\overline {F[N/2 - r]}  = F[r]$$

$$\overline {G[N/2 - r]}  = G[r]$$

根据这一信息，可以把$r=N/2-r$代入$Y[r]$中：

$$\overline {Y[N/2 - r]}  = \overline {F[N/2 - r]}  +
\overline {jG[N/2 - r]}  = F[r] - jG[r]$$

再联立两个方程，可以用$Y[r]$和$\overline {Y[N/2 - r]} $来表示$F[r]$和$G[r]$

$$F[r] = {1 \over 2}\left( {Y[r] + \overline {Y[N/2 - r]} } \right)$$

$$G[r] = {j \over 2}\left( {\overline {Y[N/2 - r]}  - Y[r]} \right)$$

到这里，把x[n]（N点实数）看作一个复数序列y[u]（N/2点复数）后, 可以计算y[u]的离散傅里叶变换，并且可以推算出x[n]中偶数索引的数组成的序列的离散傅里叶变换结果$F[r]$和奇数索引的数组成的序列的离散傅里叶变换结果$G[r]$。

再考虑到用DIT分解计算x[n]的离散傅里叶变换时，第一步就是把x[n]分成两个奇偶子序列，

$$X[r] = \sum\limits_{n = 0}^{N/2 - 1} {x[2n]{e^{ - j2\pi {r2n} \over N}}}  +
\sum\limits_{n = 0}^{N/2 - 1} {x[2n + 1]{e^{ - j2\pi {r(2n + 1)} \over N}}}$$

$$X[r] = F[r] + \omega _N^rG[r],\omega _N^r = {e^{ - j2\pi {r \over {N/2}}}}$$

这样就可以计算出x[r]的前N/2点的复数结果.

但是当$r=N/2$时，代进去会发现$X[N/2] = \overline {X[N/2]} $，只能知道X[N/2]处也是一个实数，这个奈奎斯特频点的值需要进一步推导：

N/2点的离散傅里叶变换具有周期性，即$F[r]=F[r+N/2]$, $G[r]=G[r+N/2]$

$$X[r + N/2] = F[r + N/2] + \omega _N^{r + N/2}G[r + N/2] = F[r] - \omega _N^rG[r]$$

所以：

$$X[N/2] = F[0] - G[0]$$

再结合$X[r]$的共轭对称性，可以得到

$$X[r + N/2] = \overline {X[N/2 - r]} =
F[N/2 - r] - \omega _N^rG[N/2 -r] =
F[r] - \omega _N^rG[r] $$

总结一下计算步骤, 先把实数序列当做复数序列算cfft $Y[r]$, 然后根据$Y[r]$ 计算出$F[r]$和$G[r]$,
再利用N/4长度的$F[r]$和$G[r]$计算$X[r]$, 其中$X[0]$和$X[N/2]$另外单独算.

$$Y[r] = F[r] + jG[r]$$

$$F[r] = {1 \over 2}\left( {Y[r] + \overline {Y[N/2 - r]} } \right)$$

$$G[r] = {j \over 2}\left( {\overline {Y[N/2 - r]}  - Y[r]} \right)$$

$$X[r] = F[r] + \omega _N^rG[r]$$

$$X[N/2 - r] = \overline {F[r] - \omega _N^rG[r]}$$

### 逆变换

如果同样想用复数IFFT来计算实数IFFT（经过IFFT计算后结果为实数），这是一个相反的过程。

$$Y[r] = F[r] + jG[r]$$

目前已知$X[r]$，只需要用$X[r]$表示出$F[r]$和$G[r]$，就可以用IFFT计算出$Y[r]$了。

$$X[r] = F[r] + \omega _N^rG[r]$$

$$X[r + N/2] = F[r] - \omega _N^rG[r]$$

联立上面两个式子可以得到：

$$F[r] = {1 \over 2}\left( {X[r] + X[r + N/2]} \right)$$

$$G[r] = \frac{\omega _N^{ - r}}{2}\left( {X[r] - X[r + N/2]} \right)$$

$X[r]$具有共轭对称性，所以:

$$X[r] = \overline {X[N - r]} $$

计算IFFT分为两步，首先需要将长度为（N/2 + 1）的复数序列转换为长度为N/2的复数序列。

$$F[r] = {1 \over 2}\left( {X[r] + \overline {X[N/2 - r]} } \right)$$

$$G[r] = \frac{\omega _N^{ - r}}{2}\left( {X[r] - \overline {X[N/2 - r]} } \right)$$

观察对称性：

$$F[N/2 - r] = {1 \over 2}\left( {X[N/2 - r] + \overline {X[r]} } \right)$$

$$G[N/2 - r] = \frac{\omega _N^{ - r}}{2}\left( {X[N/2 - r] - \overline {X[r]} } \right)$$

所以在计算那种知道结果是实数的IFFT的时候，是可以类似计算FFT的时候一样,
同时计算$F[r]$和$F[N/2 - r]$的, 因为他们所用到的输入数据是一样的。

最后，如果FFT和IFFT都不归一化，一个序列调用FFT，再调用IFFT后会增大N倍，一般都会除以归一化因子N，有的放在FFT的计算上面，有的放在IFFT的计算上面，也有的拆分成两个$1/\sqrt{N}$，分别放在FFT和IFFT上。

一般情况下，信号经过FFT-IFFT loop会增大N倍，而如果用N/2点计算的话，就会增大N/2倍。所以如果想要让两种计算方式的结果一致，需要在第二种方式时把结果乘以2。

## 参考资料

- [FFT wiki](https://zh.wikipedia.org/wiki/%E5%BA%93%E5%88%A9-%E5%9B%BE%E5%9F%BA%E5%BF%AB%E9%80%9F%E5%82%85%E9%87%8C%E5%8F%B6%E5%8F%98%E6%8D%A2%E7%AE%97%E6%B3%95)
