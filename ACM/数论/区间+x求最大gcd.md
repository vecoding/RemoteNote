# Problem K. 最大gcd

**Input file:** standard input  
**Output file:** standard output  
**Time limit:** 2 seconds  
**Memory limit:** 256 megabytes  

## 题目描述

给定一个长度为 \( n \) 的序列 \( a \)，你需要执行恰好一次操作：选择一个区间，以及一个非负整数 \( X \)，将区间中的所有数加上 \( X \)，目标是最大化序列中所有数的最大公约数（gcd）。输出所有数的 gcd 的最大值，如果可以为无穷大，则输出 0。

## 输入格式

本题有多组输入数据。  
第一行输入一个正整数 $T(1 \leq T \leq 10^5)$，表示输入数据的组数。  
接下来的每组输入数据：  

- 首先输入一个正整数 $n(1 \leq n \leq 10^5)$，表示序列的长度。  
- 然后输入 $n$ 个正整数，表示序列 $a(1 \leq a_i \leq 10^5)$。  

保证所有输入数据的 $\sum n \leq 2 \cdot 10^5$。

## 输出格式

对于每组输入数据，输出一行一个整数，表示答案。

## 示例

| standard input | standard output |
| -------------- | --------------- |
| 5              | 514             |
| 2              | 0               |
| 114 514        | 1               |
| 1              | 4               |
| 2              | 3               |
| 5              |                 |
| 1 2 3 5 8      |                 |
| 5              |                 |
| 4 3 3 3 4      |                 |
| 5              |                 |
| 6 1 4 7 9      |                 |

#  思路

## 全局加的情况分析

1. **特殊情况**：若所有数字均相等，则答案可以为无穷大（输出0）。
2. **一般情况**：利用gcd的性质，对于任意 $a \geq b$，有 $\gcd(a, b) = \gcd(a - b, b)$。
   
   - 序列的gcd可以表示为：
     $$
     \gcd(a_1, a_2, \ldots, a_n) = \gcd(a_1, a_2 - a_1, \ldots, a_n - a_{n-1})
     $$
   - 因此，全局加 $k$ 后的gcd为：
     $$
     \gcd(a_1 + k, a_2 - a_1, \ldots, a_n - a_{n-1})
     $$
3. **差分数组**：定义差分数组 $b_i = a_i - a_{i-1}$（$2 \leq i \leq n$）。
   - 答案不超过 $g = \gcd(b_2, b_3, \ldots, b_n)$。
   - 通过选择合适的 $k$ 使得 $g \mid (a_1 + k)$，可以使得整个序列的gcd为 $g$。

## 非全局加的情况分析

1. **关键观察**：操作后至少有一个端点（$a_1$ 或 $a_n$）保持不变。假设 $a_1$ 不变，则答案必须是 $a_1$ 的一个因子。
2. **枚举因子**：枚举 $a_1$ 的所有因子 $d$，判断是否存在操作使得序列中所有数均为 $d$ 的倍数。
   - 这等价于差分数组 $b$ 中所有数均为 $d$ 的倍数。
3. **区间操作对差分数组的影响**：
   - 对区间 $[i, j]$ 加 $k$：
     - 若 $j < n$，则 $b_i \leftarrow b_i + k$ 且 $b_{j+1} \leftarrow b_{j+1} - k$。
     - 若 $j = n$，则仅 $b_i \leftarrow b_i + k$。
   - 条件：
     - 若 $j < n$，需要 $b$ 中不为 $d$ 的倍数的数字个数为 0 或 2。
     - 若 $j = n$，需要 $b$ 中不为 $d$ 的倍数的数字个数不超过 1。

## 时间复杂度

- 时间复杂度为 $O(n \times \sigma_0(a))$，其中 $\sigma_0(a)$ 为 $a$ 的因子个数。

# 标程

```C++
#include<bits/stdc++.h>
#define ll long long
using namespace std;
ll t;
int main()
{
    // ios::sync_with_stdio(0);
    // cin.tie(0);
    cin>>t;
    while (t--)
    {
        ll n,ans=0;
        cin>>n;
        vector<ll > a(n+2),b(n+2);
        for (ll i=1;i<=n;i++)
            cin>>a[i];
        for (ll i=1;i<n;i++)
        {
            b[i]=abs(a[i+1]-a[i]);
            ans=__gcd(ans,b[i]);
        }//全局加数的情况
        if (ans==0) {cout<<ans<<'\n';continue;}//无穷的情况
        for (ll i=1;i*i<=a[1];i++)
        {
            if (a[1]%i==0)
            {
                ll sum=0;
                for (ll j=1;j<n;j++)
                    if (b[j]%i!=0) sum++;
                if (sum<=1) ans=max(ans,i);
                if (sum==2&&a[n]%i==0) ans=max(ans,i);
                sum=0;
                for (ll j=1;j<n;j++)
                    if (b[j]%(a[1]/i)!=0) sum++;
                if (sum<=1) ans=max(ans,a[1]/i);
                if (sum==2&&a[n]%(a[1]/i)==0) ans=max(ans,a[1]/i);//attention！
                /*由于gcd性质gcd(a,b)=gcd(a-b,b)可知
                gcd(a,b,c)=gcd(a-b,b-c,c)=gcd(a,a-b,b-c)--------->(1)
                要保证得到答案为gcd，除了要判断a-b,b-c,还要判断c(或a)枚举因子的原因
                由于此部分sum==2保证+x的区间端点不在总序列端点
                故序列可以看成为a*gcd,b*gcd,c*gcd,d*gcd-x,e*gcd-x,f*gcd-x,g*gcd,h*gcd,i*gcd这种通式
                又由于+x的区间端点的影响，使得我们无法得到c*gcd-d*gcd
                (即等差数组中sum==2使得我们的答案不够严谨)
                因此我们只能把序列拆分成3段序列分别求gcd再求总gcd
                由(1)可知，因此产生了3个数，即a[1],a[n](判断要取模的原因),以及一个+x区间的端点上的数(设为k)
                又由于+x的x没有限制,很显然k+x可以表示任意数，不会对答案产生影响，即此时gcd即是答案
                */
            }
        }//对于a[1],枚举其因子，判断是否为全局gcd
        for (ll i=1;i*i<=a[n];i++)
        {
            if (a[n]%i==0)
            {
                ll sum=0;
                for (ll j=1;j<n;j++)
                    if (b[j]%i!=0) sum++;
                if (sum<=1) ans=max(ans,i);
                if (sum==2&&a[1]%i==0) ans=max(ans,i);
                sum=0;
                for (ll j=1;j<n;j++)
                    if (b[j]%(a[n]/i)!=0) sum++;
                if (sum<=1) ans=max(ans,a[n]/i);
                if (sum==2&&a[1]%(a[n]/i)==0) ans=max(ans,a[n]/i);
            }//对于a[n],枚举其因子，判断是否为全局gcd，内容同上
        }
        cout<<ans<<'\n';
    }
}
```

