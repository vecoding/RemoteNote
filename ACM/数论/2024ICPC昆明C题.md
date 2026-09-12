# Problem C. 金币

海盗们刚刚夺得了一枚巨大的金币！

为了决定这枚金币的归属，他们决定采用以下方法选出金币的拥有者：

设目前剩余的海盗数量为 $n$。所有海盗排成一队，然后位于第 $1, 1+k, 1+2k, \\ldots, 1+(1\\le i-1)k$ 个的海盗被淘汰。重复这一操作，直到仅剩下一位海盗。最终剩下的海盗获得该枚金币。

Charlie 是海盗中最聪明的。他想知道，要想最终获得金币，一开始他应该站在第几位？

## Input

每个测试文件包含多组测试数据。第一行包含测试数据的组数 $T (1 \\le T \\le 100)$，每组测试数据的格式如下：

第一行包含两个整数 $n$ 和 $k (2 \\le n, k \\le 10^{12})$，表示初始海盗的数量以及测试时用到的参数。

在每个测试文件中，保证所有测试数据的 $n$ 之和不超过 $10^{12}$，所有测试数据的 $k$ 之和不超过 $10^{12}$。

## Output

对于每组数据，输出一行一个整数，表示最终获得金币的海盗在初始队列中的位置。

### Example

| standard input | standard output |
| -------------- | --------------- |
| 4              | 4               |
| 6 2            | 8               |
| 8 3            | 8192            |
| 10000 2        | 1919805         |
| 1919810 114514 |                 |

---

## Note

对于样例的第一组测试数据，每轮剩下的海盗在原始序列中的位置为：

- 初始状态：$1, 2, 3, 4, 5, 6$
- 第一轮后：$2, 4, 6$
- 第二轮后：$4$

对于样例的第二组测试数据，每轮剩下的海盗在原始序列中的位置为：

- 初始状态：$1, 2, 3, 4, 5, 6, 7, 8$

- 第一轮后：$2, 3, 5, 6, 8$

- 第二轮后：$3, 5, 8$

- 第三轮后：$5, 8$

- 第四轮后：$8$

# 思路：

设最后一个数为a

暴力：考虑a在每一次循环的位置关系。发现a的位置一直在减少一个数，且这个数可以推导出来。也就是说我们只需要知道循环层数，即可从a在最后一层的位置1，推导回第一层的位置。

```C++
#include <bits/stdc++.h>
using namespace std;
#define int long long
#define pii pair<int, int>
#define tiii tuple<int, int, int>
constexpr int inf = 1e18;
void yes() { cout << "Yes" << endl; };
void no() { cout << "No" << endl; };
void solve()
{
    int n,k,n1,ans=1,dep=0;
    cin>>n>>k;
    n1=n;
    while (n1)
    {
        n1-=(n1-1)/k+1;
        //计算层数，比较好推不多介绍
        dep++;
    }
    dep--;
    while (dep--)
    {
        ans+=(ans+k-2)/(k-1);     //k-2为向上取整
        //从下往上推
        //发现增加的个数就是当前序列里有多少k-1组数，他们在上一层必然构成一个k倍
        //加1的情况和刚好整除的情况通过向上取整达到统一，所以没有分类讨论
    }
    cout<<ans<<'\n';
}
signed main()
{
    cin.tie(0), cout.tie(0), ios::sync_with_stdio(false);
    int _ = 1;
    cin >> _;
    while (_--)
    {
        solve();
    }
    return 0;
}
```



考虑优化：
发现对于无论是层数的计算还是位置的反推，他们都有减或加很多相同的数，且对于每组数的多少可以统计得到，考虑数论分块优化，故时间复杂度为$O(\sqrt{n})$。

```C++
#include <bits/stdc++.h>
using namespace std;
#define int long long
#define pii pair<int, int>
#define tiii tuple<int, int, int>
constexpr int inf = 1e18;
void yes() { cout << "Yes" << endl; };
void no() { cout << "No" << endl; };
void solve()
{
    int n,k,pos=1,dep=0;
    cin>>n>>k;
    while (n)
    {
        int to=(n-1)/k+1;
        int num=n-(to-1)*k;
        int fre=num/to+(num%to!=0);//整除的特判
        n-=to*fre;
        dep+=fre;
    }
    dep--;
    while (dep>0)
    {
        int add=(pos+k-2)/(k-1);
        int num=add*(k-1)-pos;//取模改为减法优化，减少常数
        int fre=num/add+1;
        pos+=add*min(dep,fre);
        dep-=fre;
    }
    cout<<pos<<'\n';
}
signed main()
{
    cin.tie(0), cout.tie(0), ios::sync_with_stdio(false);
    int _ = 1;
    cin >> _;
    while (_--)
    {
        solve();
    }
    return 0;
}
```

注：

$\sum_{i=1}^{n} f(i)g\left( \left\lfloor \frac{n}{i} \right\rfloor \right)$

为整除分块形式，只要f的前缀和可以$O(1)$得到，那么对于关于n的除法上的求和时间复杂度为$O(\sqrt{n})$。
