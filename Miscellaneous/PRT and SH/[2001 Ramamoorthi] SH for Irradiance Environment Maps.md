# An Efficient Representation for Irradiance Environment Maps

## Abstract

- Omit light transport
- Diffuse object + environment map
- Precompute environment map to SH3

## Steps

The irradiance $E$ at direction $n$ is given by

$$E(n) = \int_{\Omega^+(n)} L(\omega)(n\cdot\omega)d\omega$$

, and the final radioasity is $B(p, n) = \text{albedo}(p)E(n)$

Therefore, we can precompute $L$ and $n\cdot\omega$ (denoted by $A$) to SH to reconstruct $E$ at runtime.

### cosine term to SH

We use this representation:

$$(x, y, z) = (\sin\theta\cos\phi, \sin\theta\sin\phi, \cos\theta)$$

Denote $A(\theta) = \max(\cos\theta, 0)$. Note that $A$ is independent of the azimuth angle $\phi$, so $A(\theta) = \sum_l A_lY_{l0}(\theta)$ is only relevant to SH with $m = 0$.

### Prefilter environment map

As $d\omega = sin\theta d\theta d\phi$, we have

$$L_{lm} = \int\int L(\theta, \phi)Y_{lm}(\theta, \phi)sin\theta d\theta d\phi$$

In practice, the integration is done by sampling over every pixel of the environment map.

### Rendering

The formula is given above, which is simply the reconstruction of SH.

Since $E$ varies slowly, it is sufficient to reconstruct the irradiance per-vertex and interpolate across the triangle.

## Future work

- Frequency-space methods for specular BRDF