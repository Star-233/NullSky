---
tags:
  - 高等数学
  - 题目
---

> [!NOTE] 题目
> 求微分方程的通解
> $$y\ln y dx + (x-\ln y)dy=0$$


## 过程


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
