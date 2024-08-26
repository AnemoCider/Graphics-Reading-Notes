# Rendering techniques in Toy Story 3

## SSAO based on line integrals

Related to this pub [Volumetric Obscurance
, 2010 Loos and Sloan](https://www.ppsloan.org/publications/vo.pdf)

### Overview

Obscurance: 

$$A(p) = \frac{1}{\pi}\int_{\Omega}\rho(d(p, \omega))\cos\theta d\omega$$

From the point $p$ and along direction $\omega$, $\rho(d(p, \omega))$ is $0$ until it reaches a surface (an occluder).

This requires shooting rays from point $p$, which is costly in traditional rasterization pipeline.

Volumetric obscurance:

$$V(p) = \int_X\rho(d(p, \omega))O(x) dx$$

, in which we sample neighboring points $x$ around $p$. $O(x)$ is $1$ if $x$ is within an object. $\rho$ will simply be $1$, and may fall off to $0$ if $x$ is too far away. We just assume it is $1$.

In reality (SSAO), the inside-outside test is approximated by depth comparison.

Essentially we are approximating this integral.

