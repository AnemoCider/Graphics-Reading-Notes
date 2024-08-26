# Dynamic Occlusion with Signed Distance Fields

## SDF

- Storage
  - 3D texture, float16.
  - For each mesh, usualy 50^3
  - 50^3 * 16 / 1024 / 8 = 240KB
- Scene representation
  - Partial occluders (e.g., foliage): treated as two-sided surface, so no full intersection
  - Atlas distance field textures for global scene. 512^3 for a level
  - Scene is on GPU

## Application

Basically do cone tracing over SDF

- DFSS
- DFAO
- Sky occlusion (apply to sky SH)

## Global SDF

- 4 clipmaps, 128^3 each
- sample object SDFs near start of cone, global when distant