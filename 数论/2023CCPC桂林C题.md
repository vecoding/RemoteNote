# C. Master of Both IV

**时间限制：** 每测试 1 秒  
**内存限制：** 每测试 1024 MB

陈教授是算术运算和二进制运算的大师。今天他给学生 Putata 和 Budada 布置的作业是：给定一个序列 $\{a_n\}$，求序列 $\{1,2,\dots,n\}$ 中满足以下条件的非空子序列 $\{i_1,i_2,\dots,i_m\}$ ($1 \le i_1 < i_2 < i_3 \dots < i_m \le n$, $1 \le m \le n$) 的数量：对于 $\forall x \in [1,m]$，都有 $a_{i_x} \mid \bigoplus_{j=1}^m a_{i_j}$。

这里 $\oplus$ 表示按位异或运算，$\bigoplus_{j=1}^m a_{i_j}$ 等于所有元素 $a_{i_j}$（$1 \le j \le m$）的按位异或。当且仅当存在一个非负整数 $k$ 使得 $s = k \cdot x$ 时，我们说 $x \mid s$。

请帮助 Putata 和 Budada 完成他们的作业。为了打破传说，请输出对 998244353 取模后的答案。

## 输入格式

第一行包含一个整数 $t$ ($1 \le t \le 2 \cdot 10^5$)，表示测试用例的数量。

对于每个测试用例：
- 第一行包含一个整数 $n$ ($1 \le n \le 2 \cdot 10^5$)，表示序列的长度。
- 第二行包含 $n$ 个整数，第 $i$ 个整数是 $a_i$ ($1 \le a_i \le n$)，表示序列中的第 $i$ 个元素。可能存在 $i \ne j$ 时 $a_i = a_j$ 的情况。

保证所有测试用例的 $n$ 的总和不超过 $2 \cdot 10^5$。

## 输出格式

对于每个测试用例，输出一行一个整数，表示答案。

## 输入样例

```
2
3
1 2 3
5
3 3 5 1 1
```

## 输出样例

```
4
11
```





# 思路：

题目描述有点抽象，整除包括k=0的情况。
手玩样例发现性质：为了满足$a_{i_x} \mid \bigoplus_{j=1}^m a_{i_j}$，当且仅当$\bigoplus_{j=1}^m a_{i_j}$=max($a_{i_j}$)或0。（比较好推，不多赘述）
然后就是：
1、当 $\bigoplus_{j=1}^m a_{i_j}=0$ 的情况等价于询问一个集合有多少子集异或和为 $0$，设这个集合异或线性基的秩为 $r$，则答案为 $2^{n-r}$。
2、当 $\bigoplus_{j=1}^m a_{i_j} = \max\{a\}$ 时，枚举 $s$，然后枚举所有 $a_i \mid s$ 的元素加入线性基计算即可。
时间复杂度 $O(n \log^2 n)$。

原因是线性基可以构造除了0以外的数（线性基所包含的01位数下的），然后再随便加一个线性基能够造的数就=0了。即除去用来构造线性基的数，其他的都是可以通过线性基构造出来，故答案为$2^{总个数-构造线性基所用的个数}$

对于情况1需要注意空集不能算，要减1。

对于情况2和情况1代码形式相同的原因解释：线性基构造成功后，该线性基可以表示其范围内的任意数，当然也可以构造出$\max\{a\}$。对于线性基构造所用的元素外的其他元素，无论其选或不选，产生的新数必然可以通过线性基构造出另一个数使它俩异或为$\max\{a\}$。

# 代码：

```c++
#include <bits/stdc++.h>
using namespace std;
#define int long long
const int mod = 998244353;
const int B = 31; 

int pow2[200010];

struct Lb
{
    int b[B];
    int rank;
    Lb()
    {
        memset(b, 0, sizeof(b));
        rank = 0;
    }
    void insert(int cur)
    {
        for (int i = B-1; i >= 0; i--) 
        {
            if (cur >> i & 1)
                if (b[i])
                    cur ^= b[i];
                else
                {
                    b[i] = cur;
                    rank++;
                    return ;
                }
        }
    }
};

void solve()
{
    int n, ans = 0;
    cin >> n;
    vector<int> a(n + 3), mp(n + 3), cnt(n + 3);
    vector<Lb> b(n + 3);
    
    for (int i = 1; i <= n; i++)
    {
        cin >> a[i];
        mp[a[i]]++;
    }
    
    for (int i = 1; i <= n; i++)
    {
        if (mp[i])
            for (int j = 0; i * j <= n; j++)
            {
                cnt[i * j] += mp[i];
                b[i * j].insert(i);
            }
    }
    
    for (int i = 0; i <= n; i++)
    {
        if (!i || mp[i])
        {
            ans = (ans + pow2[cnt[i] - b[i].rank]) % mod;
        }
    }
    cout << (ans - 1 + mod) % mod << '\n';
}

signed main()
{
    // 预计算幂数组
    pow2[0] = 1;
    for (int i = 1; i <= 200005; i++)
        pow2[i] = pow2[i - 1] * 2 % mod;
        
    ios::sync_with_stdio(false);
    cin.tie(0);
    int _ = 1;
    cin >> _;
    while (_--)
    {
        solve();
    }
    return 0;
}
```

