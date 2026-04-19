---
tags:
  - 高等数学
  - 题目
---

> [!NOTE] 题目
> 求微分方程的通解
> $$y\ln y dx + (x-\ln y)dy=0$$


## 解题过程


> [!fail] 尝试
> 试图直接化为线性微分方程的形式

整理一下，得到
$$
\begin{aligned}

\frac{y\ln y}{\ln y - x} &= \frac{dy}{dx}

\end{aligned}
$$
但这没什么用，变量仍然混杂在一起。


> [!check] 正解
> 线性要求 $y$ 不能出现在超越函数里面，这根本就不是线性微分方程。但我们可以反过来看，那么他就是线性的了。

反过来看，可以整理得到
$$
\frac{dx}{dy} + \frac{1}{y\ln y}x=\frac{1}{y}
$$
这是标准的一阶线性微分方程，其中
$$
\begin{cases}
P(y) = \frac{1}{y\ln y} \\
Q(y) = \frac{1}{y}
\end{cases}
$$
那么通解
$$
x = e^{-\int P(y)dy}[\int Q(y)e^{\int P(y) dy}dy+C]
$$
其中
$$
\int P(y)dy = \int \frac{1}{y\ln y} dy
$$
令 $u=\ln y$，有 $du = \frac{1}{y}dy$，则
$$
\int P(y)dy = \int u^{-1}du= \ln u + C = \ln \ln y + C
$$
则
$$
\begin{aligned}
x &= e^{-\ln\ln y}[\int \frac{1}{y}e^{\ln\ln y}dy + C] \\
&= \frac{1}{\ln y} [\int \frac{\ln y}{y}dy + C] \\
&= \frac{1}{\ln y}[\frac{\ln^2 y}{2} + C] \\
&= \frac{\ln y}{2} + \frac{C}{\ln y}
\end{aligned}
$$

## 总结


> [!NOTE] idea
> 一个一阶微分方程，对于某个变量来说，可能不是线性的，但对另一个变量来说，可能就是线性的。可以考虑从非线性的那一个入手。
