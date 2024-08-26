# Volumetric Global Illumination At Treyarch

A good article on irradiance volume: https://zhuanlan.zhihu.com/p/23410011

## Lightmaps

Cons:

- Have issues with complex or intersecting geometry
- Don't work on dynamics
- Software ray tracing on CPU is slooooooow
- Baking results not immediately visible in the editor - needs importing

## Where we start

- Deferred-lighting pipeline
- Reflections already in place, using reflection probe

An idea: store diffuse BRDF in the reflection probe as well?

Fundemental problem: reflection probe is too sparse, and what it doesn't capture becomes shadowed, which is incorrect.

So we want some form of volume texture.

## Baking irradiance volume 

What if one reflection probe per voxel:

Suppose we have 138 volumes, 40^3 texels each. Needs 10 days to render... Thereforem, simply ray tracing each voxel is too expensive.

Idea: Use data from the already baked reflection probes!

For each voxel, fire 4096 rays to intersect with the scene. If the intersection is visible from some reflection probe, just use data there. Else, place a temporary reflection probe, which is only for volumetric diffuse GI and will be discarded later.

## Storage

- AHD
  - Lack of details
  - The entire back hemisphere of normal would be flat
  - When highlight direction changes drastically, interpolating needs additional care
- 2-order SH
  - Easy to blend, which is good
  - Ringing artifacts
- Ambient cube
  - Each face stores a float3 for RGB
  - Higher precision than SH2
  - Storage cost between SH2 and SH3, but at runtime we only need to sample 3 directions (not all 6), thus cheaper than SH2
  - Make use of the hardware trilinear filtering

## Light leaking

LET THE LIGHT ARTISTS FIX IT...

Place volume edges precisely to fit the space. Add a fast attenuation near the boundary of the box.

## Side notes

Ambient Cube is more intuitive for artists...

Artists using SH be like: how to adjust lighting from a certain axis?