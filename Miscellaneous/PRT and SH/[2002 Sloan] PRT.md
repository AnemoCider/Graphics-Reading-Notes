# Precomputed Radiance Transfer

## Abstract

- A real-time method for rendering diffuse and glossy objects in low-frequency lighting environments, capturing soft shadows, interreflections, and caustics.
- Precomputes `transfer functions` over an object's surface, and samples incident lighting at runtime.

## Problem and Related Work

Problem: Real-time rendering under lighting from area sources, with effects like soft shadows, interreflections, and caustics.

Difficulties:

- must model complex BRDF
- requires integration of light at each surface point
- must account for light transport effects (which simply prefiltering the env map cannot achieve)

This paper works for diffuse and glossy objects, plus light transport over the object itself and the neighboring environment.

## Steps

### Self-transfer

#### Light sampling

First consider the incident lighting around an object $O$. With the assumption that lighting variation over $O$ (without the presence of $O$) is small, we can use a single sample at the center of $O$ to estimate lighting. To be more accurate, we can use iterated closest point (ICP) in the precompute stage, to generate several points to sample. To blend the samples, we can also the coefficients for blending for each sample. The lighting is denoted as $L_p(s)$, a function of incident direction, and the SH coefficients are $(L_p)_i$.

#### Diffuse transfer

Transfer vector at a surface point $p$, $M_p$, transforms the lighting vector to scalar exit radiance.

$$L_p' = \sum_{i=1}^{n^2}(M_p)_i(L_p)_i$$

For diffuse surface, the unshadowed diffuse transfer function is 

$$T_{DU}(L_p) = \frac{\rho}{\pi}\int L_p(s)H_{Np}(s)ds$$

where $T_{DU}$ means "Diffuse Unshadowed Transfer", and $H_{Np} = \max(N_p\cdot s, 0)$. Therefore, the transfer vector is $H_{Np}$.