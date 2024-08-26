# Reflection

## 前言

学习笔记，仅供参考；个人水平有限，欢迎讨论以及指正。

## 概述

环境光在光滑表面的反射(indirect specular)是场景光照的重要组成部分。但由于这部分反射是高频信息，因此往往采取和环境光漫反射不同的策略来实现。

UE4和Unity都提供了静态或者实时的Reflection Probe/Capture, 实时的Planar reflection以及SSR。在性能允许的情况下，一般会优先使用SSR，然后在采样不到（屏幕外）的时候再用probe。

## Reflection Probe

在烘焙的时候，把环境的radiance值存储到cubemap中，并根据IBL算法生成mipmap。这种方法也被称为localized environment map[^4].

reflection probe主要有立方体和球面两种形式。考虑到视差修正，probe的形状和被反射的物体贴合得越好，误差就越小。球面probe会有较为均匀的误差，因此适用于大部分情况；而立方体往往只适用于室内场景，因为如果环境的形状不能贴合立方体的话，在立方体的边缘上会有很大的误差。

### Image-based Lighting

大多数cubemap采取的是IBL算法，也就是会用到ue4的split-sum approximation[^10]. [知乎上有文章](https://zhuanlan.zhihu.com/p/66518450)对此进行了分析，而frostbite[^9]的解释是这其中并没有严格的数学原理，只是在保证对于constant环境光准确的情况下采取了近似。他们还尝试过使用数学方法，但是即使理论上更正确，实际视觉效果也比不上split-sum（这就是实时渲染吧，笑）。总之，大致的思路就是根据BRDF里的D项做重要性采样，然后使用近似把光照拆出来，得到DFG和LD两项积分，然后分别做预计算。

### Parallax Correction

面对非无穷远处的环境，采样probe时会有视差(parallax)，也就是着色点和probe向相同方向会采样到不同的environment。离probe越远，视差越大。我们可以用一个Volume来标记实际被probe采样的几何体（这在Unity中被称为Reflection Proxy Volume[^3]），然后运行时找出与几何体的交点，再去cubemap上对应的方向采样。

![Parallax](./Images/Reflection/Parallax.PNG)

图中p是着色点，红框是proxy volume，r'是修正之后的采样方向。

### 遮挡问题

因为probe和着色点不在同一位置，所以probe存的radiance可能会被物体的局部几何遮挡，进而采样时产生漏光。

RTR4中提到可以用specular occlusion来算遮挡。Frostbite[^9]是根据视线方向、ambient occlusion和GGX roughness拟合的：对于完全rough的表面，SO = AO; 对于光滑表面，越接近grazing angle，AO的影响就越大。

另一种方法是对比probe和着色点的漫反射信息，来估测遮挡的比例（一般称作normalization）。CoD里的做法[^5][^6]是对每个probe，在environment map的基础上计算出半球的irradiance，编码到3阶SH，在运行的时候用反射方向采样，得到env irradiance；然后着色点在计算diffuse GI的时候，会得到实际的pixel irradiance；最终，把采样environment map得到的radiance乘上pixel/env，作为遮挡修正。

### 采样修正

对于glossy表面，着色点离最终的环境越远，反射lobe对应的区域也就越大。或者说，被反射的物体与计算反射的表面接触的地方，反射需要采样的区域很小，即使是glossy的表面看到的反射也会很清晰（这被称为contact hardening）。Frostbite的做法是基于距离修正采样的roughness (distance based roughness). 具体来说，分别对于probe和着色点估计一个cone来拟合lobe，并计算cone和proxy volume的交面的半径R，最后用R_shading/R_probe来修正roughness。

另外，GGX的lobe并不是以反射方向为中心的。roughness越大、越靠近grazing angle，lobe的中心就会越靠近Normal，这被frostbite称为off-specular peak，并且在积分和采样的时候都引入了相应的修正。

![Off-specular peak](./Images/Reflection/Off-specular.PNG)

### 动态更新与Relight

当场景或光源发生动态变化的时候，需要动态地更新probe。

对于静态场景动态光源(例如TOD)，一种常见的方法是bake的时候存下场景的信息(cubemap gbuffer)，然后做根据光照信息relight。

CoD[^6]使用的是albedo + specular, depth, normal以及emissive/ambient lighting；对马岛之魂[^8]则是albedo和normal + depth。

relight基本上就是基于cubemap gbuffer的渲染。为了获取全局光照信息，对马岛之魂用的是采样单个diffuse探针（性能考虑）来计算indirect diffuse。对于indirect specular应该是直接开摆了，毕竟用reflection probe来算就套娃了，用SSR的话性能开销就太高了。对于只有specular的表面（例如金属），或许可以像Frostbite那样把fresnel f0当albedo，按照diffuse来算。

对马岛之魂在relight遇到的另一个问题是，室内的probe算阴影的时候拿到的far shadow map分辨率太低了，而且建筑的LOD也可能不匹配。解决办法是不管shadow map，找到directional light和proxy volume的交点，然后采样cubemap在那一个点的depth，足够大的话就认为是天空，也就是没被遮挡。

![Cubemap Shadow](./Images/Reflection/Cubemap%20shadow.jpg)

在生成mip chain的时候，prefilter cubemap需要大量采样，所以运行时需要做一些加速，例如重要性采样，或者像CoD一样用[这套快速的近似方案](https://www.ppsloan.org/publications/ggx_filtering.pdf)。

## Planar Reflection

平面反射能提供更高精度，但是开销更大的反射效果。一般的实现方法是手动在场景中放置一个planar reflection actor，在更新的时候复制一份main view，给view matrix乘上mirror matrix来把场景反射到mirrored space中，然后渲染场景；在计算indirect specular的时候去采样这个texture。

一般而言，用正常的视线方向采样就行，但是ue为了支持normal distortion的效果（比如计算海面反射的时候，波浪可以扭曲平面反射），具体实现是把normal和视线方向都反射到mirrored space中，再计算反射方向，最后结合normal distortion的参数来估计物体反射后的位置。

处于镜子后方的物体不应该被渲染到actor上。常见的解决方式是把actor平面设为clipping plane，例如openGL的glClipPlane，或者unity的Camera.CalculateObliqueMatrix. 

## Screen Space Reflection

屏幕空间反射的大体思路是从G-Buffer拿到深度和法线，沿着反射方向在trace光线，使用depth buffer来判断交点，然后采样color。

Frostbite有一篇非常详细的分享[^1]，不妨就跟着梳理一遍。

首先，比较早的是CryEngine在2011年的分享[^11]。大体思路是在世界空间ray marching，步长引入jitter来减少artifact；采样的时候reproject intersection去拿上一帧的结果。随后，这篇[^12]提到了上述做法的一个问题，就是世界空间均匀步长的ray marching在屏幕空间是不均匀的，因此交点判断不准确。因此，这篇文章的做法是在屏幕空间用DDA算法去遍历对应的所有像素（屏幕空间的插值要手动算一个perspective correction）。

以上的做法只支持完全的镜面反射，对于glossy表面大约只能trace多条光线来算。Killzone的这篇[^13]专门处理了glossy的情形，思路是filter然后基于roughness采样。第一步是对每个pixel计算完全反射。在trace ray的时候，表面越光滑步长就越短，找到交点之后同样reproject回上一帧，然后插值最后两个采样点(hit前和hit后)。接下来做了一个非常快速的blur（生成mipmap chain，然后采样得到最终的反射值。为了进一步提高性能，计算的时候用half resolution，然后类似于taa的做法，加jitter然后和上一帧blend。

Frostbite的目标是有以下的效果：

- 支持sharp和blurry的反射
- contact hardening
- specular elongation，也就是对于anisotropic的表面，反射会被拉长
- per-pixel的roughness和normal

Elongation的效果[^2]:

![Elongation](./Images/Reflection/Lengthy.PNG)

Killzone问题主要是在blur那一步。首先，reflection probe是对cubemap做blur，然后用normal采样；而Killzone直接blur了ray tracing之后的周围像素反射结果，所以肯定会有问题。另外，cubemap的filter之所以有contact hardening，是因为用户会提前创建好proxy volume，然后计算distance-based roughness。但SSR没有这部分信息（或许可以在trace的时候把距离存下来？），所以只能用原本的roughness做blur以及采样。

Frostbite的做法是，首先一个pass，用重要性采样记录下所有intersection；接着一个resolve pass，相邻的点复用intersection，并且基于distance-based roughness做了一个cone fit来blur。以下几个细节值得一提：

- 在最开始的时候，把屏幕分tile，每个tile trace预先几条光线来估计需不需要SSR，如果需要的话大概要实际trace多少。
- 光滑表面基于Hi-Z做trace，其余的直接coarse ray marching，反正会被狠狠地filter。
- 具体复用的是邻近像素采样到的值，以及采样光线的pdf，但brdf和cosine用的是实际像素的。然而实际发现，邻近像素之间的采样分布总体上variance比较大，导致噪点非常明显。例如，当前像素一个很大的brdf可能会对上相邻像素一个很小的pdf。解决思路是，既然这一项的值容易出问题，我们就用它来做normalization。这里不得不放个公式：

$$ L_o = \frac{\int L_i(l)f_s(l\rightarrow v)\cos\theta_l dl}{\int f_s(l\rightarrow v)\cos\theta_l dl} \int f_s(l\rightarrow v)\cos\theta_l dl \approx \frac{\sum \frac{L_i(l_k)f_s(l_k\rightarrow v)\cos\theta_{l_k}}{p_k}}{\sum \frac{f_s(l_k\rightarrow v)\cos\theta_{l_k}}{p_k}} \int f_s(l\rightarrow v)\cos\theta_l dl$$

- 左边的分式对于分子分母分别重要性采样，还是一样BRDF用自己的，pdf用相邻样本的。右边的积分直接预计算。不难发现这和split-sum非常相似，实际上这个预积分的结果和IBL那里的一模一样，所以可以直接拿过来用。极端情况下，当遇到某个非常大的 $\frac{f_s(l_m\rightarrow v)\cos\theta_{l_m}}{p_m}$ ,  这个式子会退化为 $L_o = L_i(l_m) \int f_s(l\rightarrow v)\cos\theta_l dl$ , 所以确实能有效减少噪点（代价就是模糊）。
- Sparse ray-tracing: 上述weight normalization非常好用，以至于可以随意白嫖周围像素的结果，而自己甚至不用做任何trace。所以，我们只需要half-res ray tracing，然后复用周围样本做full-res resolve。最后加上full-res的temporal filter，效果还是很好的。
- Temporal filter for reflection: 在视角移动的情况下，反射面上同一个像素在两帧之间可能会反射不同的物体。因此，简单地reproject像素位置会导致smearing。具体做法猜测是（原文这里没有写得很详细）在trace ray的时候对每个点记录下object的depth，然后reproject反射平面上的像素，在周围找到object depth相近的点去做blend。
- Filtered importance sampling: 相当于拟合的cone tracing采样，对用于采样的color texture做了blur。采样的时候基于roughness + distance。由于cone并不能很好地fit grazing angle时候的anisotropic lobe，所以这里做了一个拟合去减小cone的大小，防止过度模糊。

## 结语

Cubemap, Planar Reflection和SSR都是实时渲染中非常经典的反射算法。除此之外，比较主流的还有性能较为友好、适用移动端的PPR，以及基于光线追踪的Ray-traced Reflection和Lumen等算法。可以看到，即使是可选方案相对较少的反射，也有诸多变体和各种细节；可以预见，接下来想整理的漫反射又是一个大坑。

那么，前面的坑，就以后再来填吧。

\_( -ω-` )⌒)\_

[^1]: [2015 Siggraph Course: Stochastic SSR](https://advances.realtimerendering.com/s2015/Stochastic%20Screen-Space%20Reflections.pptx)

[^2]: [2011 GDC The Technology Behind the DirectX 11 Unreal Engine "Samaritan" Demo](https://www.nvidia.com/content/pdf/gdc2011/gdc2011epicnvidiacomposite.pdf)

[^3]: [Unity HDRP: Accurate Reflection](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@17.0/manual/Reflection-Probe.html)

[^4]: [Real Time Rendering](https://www.realtimerendering.com/)

[^5]: [2013 Getting More Physical in CoD: Black Ops II](https://blog.selfshadow.com/publications/s2013-shading-course/lazarov/s2013_pbs_black_ops_2_slides_v2.pdf)

[^6]: [2017 Rendering of CoD: IW](https://research.activision.com/publications/archives/rendering-of-call-of-dutyinfinite-warfare)

[^8]: [2021 Lighting, Atmosphere, and Tonemapping in Ghost of Tsushima](https://advances.realtimerendering.com/s2021/jpatry_advances2021.pdf)

[^9]: [2014 Moving Frostbite to PBR](https://media.contentapi.ea.com/content/dam/eacom/frostbite/files/course-notes-moving-frostbite-to-pbr-v2.pdf)

[^10]: [2013 Real Shading in Unreal Engine 4](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)

[^11]: [2011 Siggraph Course](https://advances.realtimerendering.com/s2011/index.html)

[^12]: [2014 Efficient GPU SS Ray Tracing](https://jcgt.org/published/0003/04/04/)

[^13]: [2014 Reflections and Volumetrics of Killzone Shadow Fall](https://www.guerrilla-games.com/media/News/Files/Siggraph2014_ProductionAndVisualFX.pdf)

[游戏中的全局光照(五) Reflection Probe、SSR和PPR](https://zhuanlan.zhihu.com/p/313845354)