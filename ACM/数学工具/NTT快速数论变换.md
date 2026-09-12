# NTT快速数论变换

由于前置知识与FTT快速傅里叶变换相同，故该篇只讲用有限域上的单位根代替复平面的的单位根部分。

## 引言

在[FFT](https://zhida.zhihu.com/search?content_id=165402667&content_type=Article&match_order=1&q=FFT&zhida_source=entity)中我们需要用到复数，由于其计算时会使用正弦函数和余弦函数，所以FFT在不断运算中无法避免地会产生精度误差。

在[离散正交变换](https://zhida.zhihu.com/search?content_id=165402667&content_type=Article&match_order=1&q=离散正交变换&zhida_source=entity)的理论中，已经证明在复数域内，具有循环卷积特性的唯一变换是**[DFT](https://zhida.zhihu.com/search?content_id=165402667&content_type=Article&match_order=1&q=DFT&zhida_source=entity)**，所以在复数域中不存在具有循环卷积性质的更简单的离散正交变换。

因此，也就提出了因此提出了以数论为基础的具有循环卷积性质的**[快速数论变换](https://zhida.zhihu.com/search?content_id=165402667&content_type=Article&match_order=1&q=快速数论变换&zhida_source=entity)**（**NTT**），用**[有限域](https://zhida.zhihu.com/search?content_id=165402667&content_type=Article&match_order=1&q=有限域&zhida_source=entity)上的[单位根](https://zhida.zhihu.com/search?content_id=165402667&content_type=Article&match_order=1&q=单位根&zhida_source=entity)**来取代**复平面上的单位根**。

## 原根与单位根性质的类比

设：
- $n$ 为大于 1 的 2 的幂
- $p$ 为素数且满足 $n \mid (p-1)$
- $g$ 为 $p$ 的一个原根

定义：
$$ \omega_n = g^{\frac{p-1}{n}} $$

### 性质推导：

1. 幂次关系：
   $$
   \begin{aligned}
   \omega_n^n &= g^{n \cdot \frac{p-1}{n}} = g^{p-1} \\
   \omega_n^{\frac{n}{2}} &= g^{\frac{p-1}{2}} \\
   \omega_{an}^{ak} &= g^{\frac{ak(p-1)}{an}} = g^{\frac{k(p-1)}{n}} = g_n^k
   \end{aligned}
   $$

2. 模运算性质：
   $$
   \begin{aligned}
   \omega_n^n &\equiv 1 \pmod{p} \\
   \omega_n^{\frac{n}{2}} &\equiv -1 \pmod{p} \\
   \omega_n^{k+\frac{n}{2}} &= \omega_n^{k}*\omega_n^{\frac{n}{2}} \equiv -\omega_n^{k} \pmod{p}\\
   (\omega_n^{k+\frac{n}{2}})^2 &= \omega_n^{2k+n} \equiv \omega_n^{2k} \pmod{p}
   \end{aligned}
   $$

### 结论：

原根 $g_n$ 在模 $p$ 条件下具有与单位根相似的性质。
而在INTT中，乘以单位根的共轭复数的操作也就会相应的变为乘以原根在模 p 意义下的逆元。

因此可以直接代用单位根进行计算。

## 原根的判定与求解（ **关于素数原根的求法问题**）

### 阶的定义：

$$\delta_p(g) = \varphi(p) = p - 1$$

设 $p$ 为素数，$g$ 为 $p$ 的原根，满足：
$$\delta_p(g) = \varphi(p) = p - 1$$

其中：
- $\delta_p(g)$ 表示 $g$ 模 $p$ 的阶
- $\varphi(p)$ 是欧拉函数，对于素数 $p$ 有 $\varphi(p) = p - 1$

**示例**：
对于 $p=2$：

- 其原根 $g=1$（因为 $1^1 \equiv 1 \pmod{2}$）

- 满足 $\delta_2(1) = 1 = \varphi(2) = 1$

**由此可知原根的充要条件：**
设 $p$ 为素数，$g$ 为 $p$ 的原根，则有：
对于 素数p，$p-1$ 的所有质因数 $p_i$，必须满足：
$$
g^{\frac{p-1}{p_i}} \not\equiv 1 \pmod{p}
且g^{p-1} \equiv 1 \pmod{p}
$$

### 求解方法：

由于原根通常较小，可以通过直接枚举验证：

1. 对 $g$ 从 2 开始枚举
2. 对每个 $g$，检查是否满足上述条件
3. 第一个满足条件的 $g$ 即为最小原根

**注**：

- 需要预先对 $p-1$ 进行质因数分解
- 原根的存在性：所有素数都有原根
- 若找到原根 $g$，则 $g^k$ (其中 $\gcd(k,p-1)=1$) 也是原根

## 总注：

一般使用NNT不需要求原根，记住几个常用模数以及他的原根即可。

## 代码：

```C++
#include <bits/stdc++.h>
#define ll long long
using namespace std;
const ll P=998244353,G=3,Gi=332748118;//以P为模数，G为其原根，Gi为原根的逆
//一般用998244353作为模数，他的原根3很小方便计算
const ll MAXN = 1e7 + 10;
ll a[MAXN], b[MAXN];
string st1, st2;
ll len1, len2, lim, k, r[MAXN];
void change(ll lim)
{
    for (ll i = 0; i < lim; i++)
        r[i] = (r[i >> 1] >> 1) | ((i & 1) << (k - 1));
}//获得位逆序置换位置关系
ll qPow(ll x,ll k)
{
    ll ret=1;
    while (k)
    {
        if (k&1) 
            ret=ret*x%P;
        x=x*x%P;
        k>>=1;
    }
    return ret;
}//快速幂
void ntt(ll *f, ll len, ll opt)
{
    for (ll i = 0; i < len; i++)
        if (i < r[i])
            swap(f[i], f[r[i]]);//保证一个数只交换一次，防止交换2次回到原位
    for (ll i = 1; i < len; i <<= 1)//当前处理的段落长度1，2，4，8....
    {
        ll wn=qPow(opt==1?G:Gi,(P-1)/(i<<1));//计算单位根
        //更新下一个长度的点值法表示
        for (ll j = 0; j < len; j += (i << 1))//枚举每个段落的起始位置
        //由于更新的是下一个长度的点值法表示，故段落长度为(i << 1)
        {
            ll n=1;
            for (ll k = j; k < j + i; k++)//蝶形优化使得只需要遍历前半部分的点值即可得到下半部分的点值
            {
                ll x = f[k], y = f[k + i];
                f[k] = (x + (n * y)%P)%P;
                f[k + i] = (x - (n * y)%P+P)%P;
                //套用计算公式
                n =(n* wn)%P;//更新单位根
            }
        }
    }
    if (opt == -1)
    {
        ll inv=qPow(len,P-2);
        for (ll i = 0; i < len; i++)
            f[i] =(f[i]*inv)%P;//若为INTT，答案为f[i]*inv
    }
}
int main()
{
    ios::sync_with_stdio(0);
    cin.tie(0);
    cin >> st1;
    for (ll i = 0; i < st1.size(); i++)
        a[st1.size() - i - 1] = st1[i] - 48;
    len1 = st1.size();
    cin >> st2;
    for (ll i = 0; i < st2.size(); i++)
        b[st2.size() - i - 1] = st2[i] - 48;
    len2 = st2.size();
    lim = 1;
    while (lim < len1+len2+1)//由于a*b位数最少为a的位数*b的位数，故保证点值法所需要的点足够多
        k++, lim <<= 1;//由于ntt只能计算长度为2^k的数，故扩展填0
    change(lim);
    ntt(a, lim, 1);//NTT
    ntt(b, lim, 1);//NTT
    for (ll i = 0; i < lim; i++)
        a[i] = a[i] * b[i];//点值法(x,y)中相同的x只需要将y相乘即可
    ntt(a, lim, -1);//INTT
    for (ll i = 0; i < lim; i++)
    {
        a[i + 1] += a[i] / 10;
        a[i] = a[i] % 10;
    }//普通进位
    bool flag = false;
    for (ll i = lim; i >= 0; i--)
    {
        if (a[i] != 0 || flag)
            flag = true;
        else
            continue;
        cout << a[i];
    }
    return 0;
}
```

