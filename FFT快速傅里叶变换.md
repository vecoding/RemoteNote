# 快速傅里叶变换

## 定义

FFT（Fast Fourier Transformation），中文名快速傅里叶变换，是离散傅氏变换的快速算法，它是根据离散傅氏变换的奇、偶、虚、实等特性，对离散傅立叶变换的算法进行改进获得的。
而在信奥中，一般用来加速多项式乘法。
朴素高精度乘法的时间为O(n^2^)，但FFT能将时间复杂度降到 O(nlog~2~n)
学习FFT之前，需要了解一些有关复数和多项式的知识。

## 多项式的两种表示方法

### 系数表示法

$$
F[x] = y = a_0 x^0 + a_1 x^1 + a_2 x^2 + \ldots + a_n x^n
$$
$$
\{a_0, a_1, a_2, \ldots, a_n\}
$$
是这个多项式每一项的系数，所以这是多项式的系数表示法。

## 点值表示法

在函数图像中，对于函数$F[x]$ (与系数表示法的$F[x]$相同)，使用 $\{(x_0, f[x_0]), (x_1, f[x_1]), \ldots, (x_n, f[x_n])\}$ n个点就可以完整描述出这个多项式(求解n元方程组a~0~,a~1~,a~2~,...,a~n~)，这就是多项式的点值表示法。

通俗来说，即将a~0~,a~1~,a~2~,...,a~n~作为未知数，通过使用 $\{(x_0, f[x_0]), (x_1, f[x_1]), \ldots, (x_n, f[x_n])\}$ n个通过$F[x]$的点求解得到。

---

## 多项式相乘

设两个多项式分别为 $f(x)$ 和 $g(x)$，我们要把这两个多项式相乘（即求卷积）。

### 如果用系数表示法：

我们需要将每一位的系数与每一位的系数相乘，多项式乘法的时间复杂度为 $O(n^2)$，这也是我们所熟知的高精度乘法的原理。

### 如果用点值表示法：

$$
f[x] = \{(x_0, f[x_0]), (x_1, f[x_1]), \ldots, (x_n, f[x_n])\}
$$
$$
g[x] = \{(x_0, g[x_0]), (x_1, g[x_1]), \ldots, (x_n, g[x_n])\}
$$
$$
f[x] * g[x] = \{(x_0, f[x_0] \cdot g[x_0]), (x_1, f[x_1] \cdot g[x_1]), \ldots, (x_n, f[x_n] \cdot g[x_n])\}
$$

可以发现，如果两个多项式取相同的 $x$，得到不同的 $y$ 值，那么只需要将 $y$ 值对应相乘即可！

#### 复杂度只有枚举的 $O(n)$

那么问题转换为将多项式从系数表示法转化成点值表示法。

#### 朴素系数转点的算法叫 DFT（离散傅里叶变换），优化后为 FFT（快速傅里叶变换），点值转系数的算法叫 IDFT（离散傅里叶逆变换），优化后为 IFFT（快速傅里叶逆变换）。之后我会分别介绍。

---

## 卷积

其实不理解卷积也没关系，但这里顺便提一下，可以跳过的卷积与傅里叶变换有着密切的关系。利用一点性质，即两函数的傅里叶变换的乘积等于它们卷积后的傅里叶变换，能使傅里叶分析中许多问题的处理得到简化。

$$
\mathcal{F}(g(x) * f(x)) = \mathcal{F}(g(x)) \cdot \mathcal{F}(f(x))
$$

其中 $\mathcal{F}$ 表示的是傅里叶变换。

## 单位根

以单位圆点为起点，单位圆的 n 等分点为终点，在单位圆上可以得到 n 个复数，设幅角为正且最小的复数为 ω~n~ ，称为 n **次单位根**，即$\omega_n=cos(2\pi/n)+sin(2\pi/n)i$

由$cos(n)+sin(n)i=e^{in}$欧拉公式可知$\omega_n^k=cos(2k\pi/n)+sin(2k\pi/n)i$

### 性质

1、显然若$\omega_n^k=1$成立，即$e^{ikn}=1$成立，当且仅当$k=2\pi/x或0$，既有$\omega_n^0=\omega_n^n=1$

2、
$$
\begin{flalign}
\omega_{rn}^{rk}&=cos(2rk\pi/rn)+sin(2rk\pi/rn)i& \\
&=cos(2k\pi/n)+sin(2k\pi/n)i& \\
&=\omega_n^k&
\end{flalign}
$$

3、
$$
\begin{flalign}
\omega_{n}^{k+n/2}&=\omega_{n}^{k}*\omega_{n}^{n/2}&\\
&=\omega_{n}^{k}*(cos(2*(n/2)\pi/n)+sin(*(n/2)\pi/n)i)&\\
&=\omega_{n}^{k}*(cos(\pi)+sin(\pi)i)&\\
&=-\omega_n^k&
\end{flalign}
$$

$$
\begin{flalign}
(\omega_{n}^{k})^{-1}&=\omega_{n}^{-k}&\\
&=(cos(2k\pi/n)-sin(2k\pi/n)i)&\\
&=(cos(2\pi-2k\pi/n)-sin(2\pi-2k\pi/n)i)&\\
&=\omega_n^{n-k}&
\end{flalign}
$$

## FFT（快速傅里叶变换）

虽然DFT能把多项式转换成点值，但它仍然是暴力代入n个数，复杂度仍然是$O(n^2)$，所以它只是快速傅里叶变换的朴素版。因此，我们需要利用单位根的性质来加速运算，从而得到FFT（快速傅里叶变换）。

对于多项式 $A(x) = a_0 + a_1 x + a_2 x^2 + \ldots + a_{n-1} x^{n-1}$，将其每一项按照下标的奇偶分成两部分：

$$
A(x) = a_0 + a_2 x^2 + \ldots + a_{n-2} x^{n-2} + x \cdot (a_1 + a_3 x^2 + \ldots + a_{n-1} x^{n-2})
$$

设两个多项式 $A_0(x)$ 和 $A_1(x)$，令：

$$
A_0(x) = a_0 x^0 + a_2 x^1 + \ldots + a_{n-2} x^{n/2-1}
$$

$$
A_1(x) = a_1 x^0 + a_3 x^1 + \ldots + a_{n-1} x^{n/2-1}
$$

显然，$A(x) = A_0(x^2) + x \cdot A_1(x^2)$。

假设 $k < n$，代入 $x = \omega_n^k$（n次单位根）：
$$
\begin{aligned}
A(\omega_n^k) &= A_0(\omega_n^{2k}) + \omega_n^k \cdot A_1(\omega_n^{2k}) \\
&= A_0(\omega_\frac{n}{2}^k) + \omega_n^k \cdot A_1(\omega_\frac{n}{2}^k) \\
\end{aligned}
$$

$$
\begin{aligned}
A(\omega_n^{k+\frac{n}{2}}) &= A_0(\omega_n^{2k+n}) + \omega_n^{k+\frac{n}{2}} \cdot A_1(\omega_n^{2k+n}) \\
&= A_0(\omega_\frac{n}{2}^k) - \omega_n^k \cdot A_1(\omega_\frac{n}{2}^k)
\end{aligned}
$$

考虑 $A_1(x)$ 和 $A_2(x)$ 分别在 $(\omega_{\frac{n}{2}}^1, \omega_{\frac{n}{2}}^2, \omega_{\frac{n}{2}}^3, \ldots, \omega_{\frac{n}{2}}^{\frac{n}{2}-1})$ 的点值表示已经求出，就可以用$O(n)$的时间复杂度求出 $A(x)$ 在 $(\omega_n^1, \omega_n^2, \omega_n^3, \ldots, \omega_n^{n-1})$ 处的点值表示。这个操作称为**蝴蝶变换**。

而 $A_1(x)$ 和 $A_2(x)$ 是规模缩小了一半的子问题，因此可以不断向下递归分治。当 $n=1$ 时直接返回。

**注意**：这个过程要求每层都能分成大小相等的两部分，所以多项式最高次项必须是2的幂。如果不是，需要在最高次项补零。时间复杂度为 $O(n \log_2 n)$。

## IFFT（快速傅里叶逆变换）

我们已经将两个多项式从系数表示法转化成点值表示法相乘后，还要将结果从点值表示法转化为系数表示法，也就是IFFT（快速傅里叶逆变换）。

首先思考一个问题，为什么要把 $\omega_n^k$（单位根）作为x代入？  
当然是因为离散傅里叶变换特殊的性质，而这也和IFFT有关。

### 一个重要结论

把多项式A(x)的离散傅里叶变换结果作为另一个多项式B(x)的系数，取单位根的倒数即 $\omega_n^0, \omega_n^{-1}, \ldots, \omega_n^{1-n}$ 作为x代入B(x)，得到的每个数再除以n，得到的是A(x)的各项系数，这就实现了傅里叶变换的逆变换了。相当于在FFT基础上再搞一次FFT。

### 证明

设 $(y_0, y_1, y_2, \ldots, y_{n-1})$ 为多项式  
$$ A(x) = a_0 + a_1 x + a_2 x^2 + \ldots + a_{n-1} x^{n-1} $$  
的离散傅里叶变换。

设多项式 $ B(x) = y_0 + y_1 x + y_2 x^2 + \ldots + y_{n-1} x^{n-1} $。  
把离散傅里叶变换的 $\omega_n^0, \omega_n^{-1}, \ldots, \omega_n^{1-n}$ 这个单位根的倒数，即 $\omega_n^0, \omega_n^{-1}, \ldots, \omega_n^{1-n}$ 作为x代入 $ B(x) $，得到一个新的离散傅里叶变换 $(z_0, z_1, z_2, \ldots, z_{n-1})$。

$$ z_k = \sum_{j=0}^{n-1} y_i (\omega_n^{-k})^i $$  

$$ = \sum_{i=0}^{n-1} \left( \sum_{j=0}^{n-1} a_j \cdot (\omega_n^i)^j \right) (\omega_n^{-k})^i $$  

$$ = \sum_{j=0}^{n-1} a_j \cdot \left( \sum_{i=0}^{n-1} (\omega_n^i)^{j-k} \right) $$

当 $ j - k = 0 $ 时，
$$ \sum_{i=0}^{n-1} (\omega_n^i)^{j-k} = n $$

否则，通过等比数列求和可知：

$$ \sum_{i=0}^{n-1} (\omega_n^i)^{j-k} = \frac{(\omega_n^{j-k})^{n}-1}{\omega_n^{j-k}-1} = \frac{(\omega_n^n)^{j-k}-1}{(\omega_n^{j-k})-1} = \frac{1-1}{(\omega_n^{j-k})-1} = 0 $$

（因为 $\omega_n^0=\omega_n^n = 0$）

所以

$$ z_k = n \cdot a_k $$

$$ a_k = \frac{z_k}{n} $$，得证。

### 怎么求单位根的倒数呢？

单位根的倒数其实就是它的共轭复数。不明白的可以看看前面共轭复数的介绍。到现在你已经完全学会FFT了，但写递归还是可能会超时，所以我们需要优化。

## 优化

### 位逆序置换

以 8 项多项式为例，模拟拆分的过程：

- 初始序列为 $\{x_0, x_1, x_2, x_3, x_4, x_5, x_6, x_7\}$
- 一次二分之后 $\{x_0, x_2, x_4, x_6\}, \{x_1, x_3, x_5, x_7\}$
- 两次二分之后 $\{x_0, x_4\}, \{x_2, x_6\}, \{x_1, x_5\}, \{x_3, x_7\}$
- 三次二分之后 $\{x_0\}, \{x_4\}, \{x_2\}, \{x_6\}, \{x_1\}, \{x_5\}, \{x_3\}, \{x_7\}$

规律：其实就是原来的那个序列，每个数用二进制表示，然后把二进制翻转对称一下，就是最终那个位置的下标。比如 $x_1$ 是 001，翻转是 100，也就是 4，而且最后那个位置确实是 4。我们称这个变换为位逆序置换（bit-reversal permutation）。

### 位逆序置换的线性时间实现

实际上，位逆序置换可以在 $O(n)$ 时间内通过递推实现。设 $len = 2^k$，其中 $k$ 表示二进制数的长度，$R(x)$ 表示长度为 $k$ 的二进制数 $x$ 翻转后的数（高位补 0）。我们需要计算序列：
$$ R(0), R(1), \cdots, R(n-1) $$

**递推基础**：
$$ R(0) = 0 $$

**递推关系**：
对于 $x > 0$，我们可以利用已知的 $R\left(\left\lfloor \frac{x}{2} \right\rfloor\right)$ 来计算 $R(x)$：
1. 将 $x$ 右移一位（即除以 2）并翻转
2. 将结果再右移一位
3. 根据 $x$ 的最低位决定是否加上 $2^{k-1}$：

$$ R(x) = \left\lfloor \frac{R\left(\left\lfloor \frac{x}{2} \right\rfloor\right)}{2} \right\rfloor + (x \bmod 2) \times \frac{len}{2} $$

**示例**（$k=5$，$len=(100000)_2$）：
要翻转 $(11001)_2$：
1. 先处理 $(1100)_2$：
   - 已知 $R((1100)_2) = R((01100)_2) = (00110)_2$
   - 右移一位得到 $(00011)_2$
2. 处理最低位：
   - 若为 1，则加上 $(10000)_2 = 2^{k-1}$
   - 若为 0，则保持不变

## 代码实现

```C++
#include <bits/stdc++.h>
using namespace std;
const double pi = acos(-1);
const int MAXN = 1e7 + 10;
complex<double> a[MAXN], b[MAXN];
string st1, st2;
int len1, len2, lim, k, r[MAXN], ans[MAXN];
void change(int lim)
{
    for (int i = 0; i < lim; i++)
        r[i] = (r[i >> 1] >> 1) | ((i & 1) << (k - 1));
}//获得位逆序置换位置关系
void fft(complex<double> *f, int len, int opt)
{
    for (int i = 0; i < len; i++)
        if (i < r[i])
            swap(f[i], f[r[i]]);//保证一个数只交换一次，防止交换2次回到原位
    for (int i = 1; i < len; i <<= 1)//当前处理的段落长度1，2，4，8....
    {
        complex<double> wn(cos(2 * pi / (i * 2)), sin(2 * pi / (i * 2)) * opt);//计算单位根
        //更新下一个长度的点值法表示
        for (int j = 0; j < len; j += (i << 1))//枚举每个段落的起始位置
        //由于更新的是下一个长度的点值法表示，故段落长度为(i << 1)
        {
            complex<double> n(1, 0);
            for (int k = j; k < j + i; k++)//蝶形优化使得只需要遍历前半部分的点值即可得到下半部分的点值
            {
                complex<double> x = f[k], y = f[k + i];
                f[k] = x + n * y;
                f[k + i] = x - n * y;
                //套用计算公式
                n *= wn;//更新单位根
            }
        }
    }
    if (opt == -1)
        for (int i = 0; i < len; i++)
            f[i] /= len;//若为IFFT，答案为f[i]/len
}
int main()
{
    ios::sync_with_stdio(0);
    cin.tie(0);
    cin >> st1;
    for (int i = 0; i < st1.size(); i++)
        a[st1.size() - i - 1] = (double)(st1[i] - 48);
    len1 = st1.size();
    cin >> st2;
    for (int i = 0; i < st2.size(); i++)
        b[st2.size() - i - 1] = (double)(st2[i] - 48);
    len2 = st2.size();
    lim = 1;
    while (lim < len1+len2+1)//由于a*b位数最少为a的位数*b的位数，故保证点值法所需要的点足够多
        k++, lim <<= 1;//由于fft只能计算长度为2^k的数，故扩展填0
    change(lim);
    fft(a, lim, 1);//FFT
    fft(b, lim, 1);//FFT
    for (int i = 0; i < lim; i++)
        a[i] = a[i] * b[i];//点值法(x,y)中相同的x只需要将y相乘即可
    fft(a, lim, -1);//IFFT
    for (int i = 0; i < lim; i++)
    {
        ans[i] += a[i].real() + 0.5;
        ans[i + 1] += ans[i] / 10;
        ans[i] = ans[i] % 10;
    }//普通进位
    bool flag = false;
    for (int i = lim; i >= 0; i--)
    {
        if (ans[i] != 0 || flag)
            flag = true;
        else
            continue;
        cout << ans[i];
    }
    return 0;
}
```

