
> [!question]
> $$
> yy''-y'^2=0
> $$
> 求通解

> [!NOTE]
> 解决这类方程的核心想法是，不直接找 $y$ 和 $x$ 的关系，而是先找 $y$ 和 $y'$ 的关系，进而找$y$ 和 $x$ 的关系。这么做的理由是，这样可以改变方程的形态，我们就可以化用常规方程的解法。

令 $p=y'$ ，原式变成
$$
y\frac{dp}{dx}-p^2=0
$$
式子仍然还包含有 $x$，让我们用多元函数求导法则将其化掉
$$
\begin{aligned}
y\frac{dp}{dy}\cdot \frac{dy}{dx}-p^2&=0 \\
yp\frac{dp}{dy}-p^2&=0 \\
\frac{dp}{p}&=\frac{dy}{y}
\end{aligned}
$$
积分得到
$$
\ln p=\ln cy
$$
$$p = C_1 y$$
随后把 $p=y'$ 代回
$$
\frac{dy}{dx} = C_1 y
$$
分离变量然后积分
$$
\frac{dy}{C_1y}=dx
$$
$$
C_1\ln y+C_2=x
$$
$$
\boxed{ y=C_2e^{C_1x} }
$$
