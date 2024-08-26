# Reflection System in Thief

## Basics of Reflection

- Planar reflections
  - Very precise
  - Costly, needs to re-render the scene
  - Only works for planar surfaces
- Cube map reflections
  - Dynamically capturing cube maps is expensive as well
  - Static cube maps are cheap, however
  - Lack resolution
- Image-based reflections
- Screen space reflections
  - ray marching against depth buffer
  - need a fallback for objects not present in the screen

## Overview of the system

- Cube map (local + global)
- Image-based reflection
- SSR
