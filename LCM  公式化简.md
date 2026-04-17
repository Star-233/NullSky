---
tags:
  - 算法
  - 题目
  - 蓝桥杯
---

[宝石组合](https://www.lanqiao.cn/problems/19711/learning/)

题目给出一个公式
$$ S = H_a H_b H_c \cdot \frac{\text{LCM}\left(H_a, H_b, H_c\right)}{\text{LCM}\left(H_a, H_b\right) \cdot \text{LCM}\left(H_a, H_c\right) \cdot \text{LCM}\left(H_b, H_c\right)} $$
这非常复杂，我们没法直接处理LCM

**就像我们遇到遇到括号会尝试拆括号一样**，我们尝试将其使用**质因数分解表示**
式子里面有多个LCM，而 $H_a,H_b,H_c$ 的质因数分解不一定相同，我们**没法直接拆开**。
因此，我们仅看看单个质因数的情况，对于一个质因数 $p$ ，假设 $H_a,H_b,H_c$ 的质因数 $p$ 的幂分别是 $x,y,z$ 。
记 $S$ 的质因数 $p$ 的幂为 $v_p(S)$，那么
$$
v_p(S)=x+y+z+\max(x,y,z)-\max(x,y)-\max(x,z)-\max(y,z)
$$
字母的顺序并不重要，我们可以假设 $x\le y\le z$ 。于是
$$
v_p(S)=x+y+z+z-y-z-z=x
$$
由于字母顺序并不重要，这代表着对于任意质因数的幂，$v_p(S)$ 总是取 $x,y,z$ 的最小值，这意味着
$$
S=p_1^{\min(最小幂)}p_2^{\min(最小幂)}\cdots p_n^{\min(最小幂)}
$$
这和最大公约数的定义完全一样，即
$$
S=\gcd(H_a,H_b,H_c)
$$