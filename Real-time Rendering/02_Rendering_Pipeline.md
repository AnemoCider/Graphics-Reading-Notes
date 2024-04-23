# 02 The Graphics Rendering Pipeline

## Application Stage

The CPU part. It passes geometry, i.e., `rendering primitives` like points, lines, and triangles, to `geometry processing stage`.

Typical tasks include:

- collision detection
- acceleration algorithms, e.g., specific culling algorithms

## Geometry Processing Stage

It has several substages:

- vertex shading
- projection
- clipping
- screen mapping

### Vertex Shading

Tasks:

- compute the position of each vertex (`gl_Position`)
- evaluate vertex output data, e.g., color, texCoord (`out` variables)

With the advent of GPU, we now do most shading per pixel. Therefore, vertex shader's main purpose is to setup per vertex data.

## Rasterization

Substages:

- triangle setup (primitive assembly)
- triangle traversal
  - generating the `fragment`: the part of a pixel that overlaps the triangle
  - *perspective-correctly* interpolating properties from vertices

## Pixel Processing