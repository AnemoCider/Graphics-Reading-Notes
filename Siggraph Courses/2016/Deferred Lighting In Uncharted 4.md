# Deferred Lighting In Uncharted 4

More discussion see https://zhuanlan.zhihu.com/p/77542272

## Motivation for Deferred Lighting

Screen-space effects require to store material data, e.g., SSR, decal, etc.

An option: thin G-Buffer + forward geometry pass.

Sidenote: this is actually light pre-pass: 1st geometry pass for lighting attributes, producing a thin G-Buffer; 2nd stage calculate lighting properties (without material); 3rd stage (2nd geometry pass) apply lighting to materials.

Problem: geometry pass is not cheap. Also we have dense objects. Also complex lighting in the forward shader adds register pressure.

Solution: fully deferred.

## Deferred Rendering

GBuffer needs to support all material.

- Compress, compress, and compress
- Problem: Deferred shader gets complex very quickly, to support different materials
  - Different lighting types
  - Different materials: Skin, fabric, plants, metal, etc.
- Solution: manually categorize shaders. 
  - Use a bitmask for used shader features for each pixel. 
  - For each tile generate a final mask for all shader features, later index into shader table.
  - Generally, needs branch in the shader (because pixels within the tile have different materials). However, this may not always be the case - all pixels within the tile may share the same material. In this case, fetch from a branchless shader table.