# Reflection

## 前言

环境光在光滑表面的反射(indirect specular)是场景光照的重要组成部分。但由于这部分反射是高频信息，因此往往采取和环境光漫反射不同的策略来实现。

## 概述

Eidos在2014年的talk中[^1]分享了他们的反射方案，用的是cubemap, image-based reflection(IBR)和SSR的组合。这里的IBR特指UE3的一套渲染技术，也被称为Billboard Reflection[^2], 流程是由美术在场景中需要被反射出来的地方（例如水坑旁的建筑）放置平面，提前把光照烘焙上去

UE4和Unity也都提供了静态或者实时的Reflection Probe/Capture, 实时的Planar reflection以及SSR。例如，Unity会优先使用

一个问题：对于glossy surface有contact hardening effect，反射面越靠近物体，反射越清晰。SSR可以很好地模拟这个效果，那cubemap可以吗？cubemap是如何处理glossy surface的？anisotropic的glossy呢？

## Reflection Probe

reflection probe主要有立方体和球面两种形式。理论来说，probe的形状和被反射的物体贴合得越好，误差就越小。球面probe会有较为均匀的误差，因此适用于大部分情况；而立方体往往只适用于室内场景，因为如果环境的形状不能贴合立方体的话，在立方体的边缘上会有很大的误差。

### Parallax Correction

面对非无穷远处的环境，采样probe时会有视差(parallax)，并且离probe越远，视差越大。我们可以用一个Volume来标记实际被probe采样的几何体（这在Unity中被称为Reflection Proxy Volume[^3]），然后运行时找出与几何体的交点，再去cubemap上正确地采样。

### 物体自遮挡问题

### Glossy表面

## Planar Reflection

### Image-based Reflection

## Screen Space Reflection

### 为什么需要SSR

[^1]: [2014 Siggraph Course: Reflection System in Thief](https://advances.realtimerendering.com/s2014/index.html#_REFLECTION_SYSTEM_IN)

[^2]: [2011 GDC The Technology Behind the DirectX 11 Unreal Engine "Samaritan" Demo](https://www.nvidia.com/content/pdf/gdc2011/gdc2011epicnvidiacomposite.pdf)

[^3]: [Unity HDRP: Accurate Reflection](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@17.0/manual/Reflection-Probe.html)

[游戏中的全局光照(五) Reflection Probe、SSR和PPR] https://zhuanlan.zhihu.com/p/313845354