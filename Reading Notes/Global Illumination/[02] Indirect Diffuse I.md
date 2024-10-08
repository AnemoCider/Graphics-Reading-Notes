# 漫反射全局光照（1） - Lightmap

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

另外，由于irradiance map的低频特性，在bake时无法使用高频的normal map。

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

#### Hemispherical Bases

除了AHD之外，还有一些定义在半球上的方案，例如H-Basis（或者其他的Hemispherical Harmonics, HSHs）[^14]这种类似于半球SH的，以及Half-life 2 Basis[^12] (又称Radiosity normal mapping, RNM) 这样用三个基向量表示的。



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

SH lightmap可以参考Frostbite的分享[^13]。

SH相对比较全能，但是储存开销也很高，所以一般只能用到不是很准确的Linear SH。在Baking的时候，先monte carlo sample出radiance的SH，然后和clamped cosine function convolve一下[^7]得到irradiance。在这个过程中可以简化一些计算，具体可以参考这篇博客[^8]。

#### SH encoding

在储存的时候，首先把 $L_1$的系数都除以$L_0$ (这里用的SH计算方式会保证 $0 \leq |f_1| \leq f_0$)，然后用4张RGB去存12个SH系数。$L_0$用HDR BC6H，$L_1$用LDR（因为压缩到$[0, 1]$了） BC7或者BC1。

#### Sampling

问题：linear interpolate SH的系数，是否就等价于linear interpolate radiance？

#### Approximating the Specular

之前提到过AHD可以近似出specular的效果，实际上SH也可以。

具体来说，用optimal linear direction当highlight direction，这和之前SH3转AHD是一样的；接着用 $L1$ 的长度模拟光源的集中程度，这点在介绍Linear SH的时候提过。在光源比较集中的时候想要更硬的高光，所以简单地给smoothness乘一个 $\sqrt{\lVert f_1\rVert}$，然后代入ggx specular计算。

当然，这种做法的缺点也和AHD一样，就是对来自不同方向的光源效果不好，并且这种artifact对于smooth的表面特别明显。因此，默认只对特定范围roughness的表面用lightmap specular，其他的还是用传统的反射算法。

### Lightmap Baking

Frostbite[^13]提到了一些lightmap baking的细节。

path tracing需要ray origin和direction。

对于origin，最直接的想法，就是取lightmap的每个texel中心。然而，lightmap分辨率很低，因此这么取很可能会漏掉texel内对最终光照有重要贡献的点。另外，物体的表面可能只覆盖了texel的一部分，但是没有覆盖到中心。解决办法就是用低差异序列做SSAA。

在生成direction的时候，采用的是Halton sequence。相比Hammersley的好处，Halton是progressive的，也就是不需要提前指定spp，而是基于noise判断终止条件。由于Halton sequence是deterministic的，所以每个texel还要加入随机的偏移，导致每个像素都使用完全相同的一组方向，产生band-like artifacts。

判断终止条件的时候，可以把样本当成normal distribution去计算running variance。当然实际上样本并不服从normal distribution，所以为了防止过早终止，可以每trace 数量上限10%的再做一次判断。

#### Lightmap UV

lightmap和normal map往往采用各自单独的uv，所以他们tangent space是不一样的。如果想用直接在tangent space里结合normal与directional lightmap得到光照的话，需要在baking的时候就转换到normal map的tangent space里。这相当于lightmap的uv只提供位置信息用于存储和采样，而不会用于计算tangent space[^14]。

### Lightmap Atlas

在存储lightmap的时候，一般先把物体拆成许多chunk，然后每个chunk单独参数化，从而得到texture space里相互不重叠的chart，最后把chart pack到atlas上。

color bleeding问题：除了chart之间不重叠之外，我们还希望在采样的时候不会bilinear filter到别的chart上面去，否则会产生错误的color bleeding（这也是不对lightmap做miipmap的原因之一[^2]）。一种做法是把filter会采样到的区域也标记为occupied，也就是把chart的边界向外扩张一些。

seam问题：chart之间是分开参数化的，因此在chart的接缝处容易出现不连续的情况，所以需要额外的算法来处理（几何方向的论文完全看不懂w）

pack到atlas上[^13]：思路是，先把更难找到位置的chart放上去，然后再放更容易的来填充holes。例如，1024x1的chart就比32x32的要更难放上去。在放上去的时候需要按行遍历找到第一个能放进去的位置。这里的遍历可以从上一次成功放进去的位置开始，而如果当前的chart比上一个更容易放进去，就需要从头开始遍历（因为要填之前的hole）。

这里放frostbite的对比图：



## 想填的坑

读懂这两篇[^15][^16]。

## 结语

## References

[^1]: [彻底看懂 PBR/BRDF 方程](https://zhuanlan.zhihu.com/p/158025828)

[^2]: [Real Time Rendering](https://www.realtimerendering.com/)

[^3]: [2015 Lighting in Unity](https://gdcvault.com/play/1021765/Advanced-Visual-Effects-With-DirectX)

[^4]: [2013 Lighting Technology of The Last of Us](http://miciwan.com/SIGGRAPH2013/Lighting Technology of The Last Of Us.pdf)

[^5]: [2018 Directional Lightmap Encoding Insights](https://www.ppsloan.org/publications/lmap-sabrief18.pdf)

[^6]: [2003 Spherical Harmonic Lighting: The Gritty Details](https://3dvar.com/Green2003Spherical.pdf)

[^7]: [2001 An Efficient Representation for Irradiance Environment Maps](https://cseweb.ucsd.edu/~ravir/papers/envmap/)

[^8]: [2017 Alternative definition of Spherical Harmonics for Lighting](https://grahamhazel.com/blog/2017/12/18/alternative-definition-of-spherical-harmonics-for-lighting/)

[^9]: [2008 Stupid Spherical Harmonics (SH) Tricks](https://www.ppsloan.org/publications/StupidSH36.pdf)

[^10]: [2017 Precomputed Lighting in CoD: Infinite Warfare](https://research.activision.com/publications/archives/precomputed-lighting-in-call-of-dutyinfinite-warfare)

[^12]: [2007 Efficient Self-Shadowed Radiosity Normal Mapping](https://cdn.akamai.steamstatic.com/apps/valve/2007/SIGGRAPH2007_EfficientSelfShadowedRadiosityNormalMapping.pdf)

[^13]: [2018 Precomputed Global Illumination in Frostbite](https://www.ea.com/frostbite/news/precomputed-global-illumination-in-frostbite)

[^14]: [2010 Efficient Irradiance Normal Mapping](https://publik.tuwien.ac.at/files/PubDat_189085.pdf)

[^15]: [2020 Precomputed Lighting Advances in Call of Duty: Modern Warfare](https://research.activision.com/publications/2020-09/precomputed-lighting-advances-in-call-of-duty--modern-warfare)

[^16]: [2024 Hemispherical Lighting Insights](https://www.ppsloan.org/publications/hemilightTR.pdf)

