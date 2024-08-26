# Indirect Diffuse Lighting, I

## 前言

## 前置：渲染方程

对于常见的PBR模型，渲染方程可以写作[^1]

$$L_o = \int_{\Omega^+}(k_d \frac{c}{\pi} + k_s\frac{DFG}{4\cos\theta_i\cos\theta_o})L_i\cos\theta_id\omega_i$$

其中漫反射项

$$L_{diff} = k_d \frac{c}{\pi}\int_{\Omega^+} L_i\cos\theta_id\omega_i$$

在出射的半球面内是个定值，不随观察角度而变化。Indirect diffuse就是考虑所有经过至少一次弹射的入射 $L_i$ 得到的漫反射结果。

## 前置：球谐函数

本文不会过多介绍球谐函数的数学原理，可以参考这篇文章[^6]作为入门。这里介绍一些有用的性质。

### 投影与重建

SH作为orthonormal basis，任意一个函数 $f(s)$ 对于某一项的least square projection就是积分：

$$f_l^m = \int f(s)y_l^m(s)ds$$

对应的，重建$f(s)$就是dot product:

$$\tilde{f}(s) = \sum_{l=0}^n\sum_{m=-l}^l f_l^my_l^m(s)$$

为了方便coding，一种习惯是用index来表示球谐系数，就有

$$\tilde{f}(s) = \sum_{i=0}^{n^2}f_iy_i(s)$$

研究表明，$L_0$, $L_1$, $L_2$这三阶的球谐函数就已经能比较精准地表示低频的diffuse irradiance信号了[^7]。

### Linear SH

把$L_0$和$L_1$的SH称为linear SH，对应的basis在笛卡尔坐标下可以表示为

$$Y(x, y, z) = [\frac{1}{2\sqrt\pi}, -\sqrt{\frac{3}{4\pi}}y, \sqrt{\frac{3}{4\pi}}z, -\sqrt{\frac{3}{4\pi}}x]$$

注意，球谐basis的符号在不同的文章中会有差异，这里笔者遵循PP Sloan的定义[^9]，与wiki和刚才提到的这篇文章[^6]的定义会有出入。

这里就可以观察到，$L_0$是唯一球面积分不为0的basis。这意味着它代表了整个球面函数的“能量”，而剩下的所有basis只不过是在球面上改变这些能量的分布。特别地，对于linear SH，$\frac{\mid\vec{f_1} \mid}{f_0}$表征了信号有多么地“方向性”，为0则球面对称，越大就越集中于球面上的一个点。[^8]

一般把$L1$ band对应的方向称为Optimal Linear Direction，也就是+x, +y, +z的系数归一化得到的方向，即

$$d_{lin} = \frac{(-f_1^1, -f_1^{-1}, f_1^0)}{\lVert f_1\rVert}$$

### Radiance to Irradiance



## Precomputed Lightmap

在离线的时候，可以预计算出静态物体的irradiance，即 

$$\int_{\Omega^+} L_i\cos\theta_id\omega_i$$

，储存到texture，也就是irradiance map当中。在运行时只需要采样这张map，乘上 

$$k_d \frac{c}{\pi}$$

就可以得到 $L_{diff}$。

理论上，我们也可以在预计算的时候包含 

$$k_d \frac{c}{\pi}$$

，然而在实际情况下，color map较高频且可以在模型间复用，而irradiance map往往低频且无法复用，所以分开存储的话能减少内存占用[^2]。

这一做法的问题主要有以下几点：

- baking耗时较长，无法在editor里实时预览
- 只对静态光源、静态场景下的静态物体是准确的
- lightmap的每个texel只有在对应到平面的时候才准确，因此对于几何形状比较复杂的小物体会有self-shadowing的问题

另外，由于irradiance map的低频特性，在bake时无法使用高频的normal map（否则会出现采样频率跟不上实际频率导致的aliasing）。

## Directional Lightmap

Directional lightmap通过储存额外的光照方向信息，在采样的时候插值，从而结合normal map提供更高频的漫反射表达。

对比图[^3]：

具体的存储形式有很多，以下介绍其中几种：

### AHD

The Last of Us介绍了一种简单的实现方法[^4]：把irradiance拆分成Ambient（常数） + Highlight Direction(direction + color)。运行时的计算也很直接：

$$I = C_{amb} + C_{dir}\cdot\max((n\cdot d), 0)$$

但是，基于方向的参数化往往是非线性的，所以通常的线性插值采样在数学上是有问题的。例如在阴影的边缘，dominant light的方向往往变化剧烈，容易产生artifacts。

论文阴影边缘对比图：



一种简单的解决办法是提前spatial filter一下这个direction，相当于采样的时候取到的是和周围平均过的direction，但这样显然会损失高频信息。在插值direction的时候，也可以选择用color做weight，但这样就用不了硬件的插值了。

很自然的想法是让AHD变成线性的[^5]，这样还可以充分利用硬件的texture filtering。

思路就是先不管normal map，即考虑

$$n = z = [0, 0, 1]^T$$

的情形。这时重建的irradiance可以表示为

$$I_z = C_a + max(0, z\cdot d)C_d = C_a + d_zC_d$$

为了让$I_z$在插值的时候是正确的，我们在干脆在储存的时候$C_d$换成$I_z$，相当于做如下映射:

$$(C_a, d, C_d) \mapsto (C_a, d, I_z)$$

，然后插值的时候再映射回来:

$$C_d = \frac{I_z - C_a}{d_z}$$

，继续像一般的AHD一样做shading。

插值的图片：


可以看到传统AHD在还没用normal map的时候，线性插值就已经出问题了。

#### AHD Baking

将irradiance投影到AHD上并不直观，所以一般先投影到SH3，然后做一个least-square fit转换到AHD；或者也可以把Optimal Linear Direction当做highlight direction，然后限制z轴的irradiance相比SH3不变、限制ambient和directional color非负，去找一个最优解[^10]。[这篇知乎文章](https://zhuanlan.zhihu.com/p/701085097)详细地讲解了其中的数学过程。


#### AHD总结

总体来说，AHD的优点是：

- direction提供了dominant light的信息，例如可以借此模拟出glossy材质的specular highlight
- 非常直观，方便美术手动调效果
- 需要存的参数相对较少，只需要8个floats(direction = 2, RGB color * 2 = 6)
- 相比SH有更强的contrast（美术和灯光师可能会比较喜欢这种效果）


缺点是：

- 因为只有一个dominant light方向，所以不能很好地表示来自不同方向的光照。如果把来自多个光源的直接光照也bake到AHD lightmap，就很容易会出现这种问题；RGB三个通道的dominant light direction不同的情况下，也会有很明显的问题。


放和SH3的对比图：


可以注意到AHD的阴影更加明显，但是在靠近相机的地面上黄光的color bleeding丢失了。

### Spherical Harmonics

GDC 2018: Precomputed Global Illumination in Frostbite

SH相对比较全能，但是储存开销也很高，所以一般只能用到不是很准确的Linear SH。在Baking的时候，先monte carlo sample出radiance的SH，然后和clamped cosine function convolve一下[^7]得到irradiance。在这个过程中可以简化一些计算，具体可以参考这篇博客[^8]。

#### Deringing

参考[^12]. 我记得2020的CoD有更好的followup.

### H-basis

实际上，我们只需要normal map对应的半球信息，而SH则存了完整的球面信息，因此会有空间上的浪费。H-basis[^?]的思路是

### 

## References

[^1]: [彻底看懂 PBR/BRDF 方程](https://zhuanlan.zhihu.com/p/158025828)

[^2]: [Real Time Rendering](https://www.realtimerendering.com/)

[^3]: [Directional Lightmap, Unity Doc](https://docs.unity3d.com/6000.0/Documentation/Manual/LightmappingDirectional.html)

[^4]: [2013 Lighting Technology of The Last of Us](http://miciwan.com/SIGGRAPH2013/Lighting%20Technology%20of%20The%20Last%20Of%20Us.pdf)

[^5]: [2018 Directional Lightmap Encoding Insights](https://www.ppsloan.org/publications/lmap-sabrief18.pdf)

[^6]: [2003 Spherical Harmonic Lighting: The Gritty Details](https://3dvar.com/Green2003Spherical.pdf)

[^7]: [2001 An Efficient Representation for Irradiance Environment Maps](https://cseweb.ucsd.edu/~ravir/papers/envmap/)

[^8]: [2017 Alternative definition of Spherical Harmonics for Lighting](https://grahamhazel.com/blog/2017/12/18/alternative-definition-of-spherical-harmonics-for-lighting/)

[^9]: [2008 Stupid Spherical Harmonics (SH) Tricks](https://www.ppsloan.org/publications/StupidSH36.pdf)

[^10]: [2017 Precomputed Lighting in CoD: Infinite Warfare](https://research.activision.com/publications/archives/precomputed-lighting-in-call-of-dutyinfinite-warfare)

[^12]: [Converting SH Radiance to Irradiance](https://grahamhazel.com/blog/2017/12/22/converting-sh-radiance-to-irradiance/)

[^?]: [2010 Efficient Irradiance Normal Mapping](https://publik.tuwien.ac.at/files/PubDat_189085.pdf)
