# A Deep Dive into Nanite Virtualized Geometry

## Occlusion Culling

- HZB
- Find the mip level on which the bound on the screen is $<= 4\times4$
- Reprojecting the z-buffer from previous frame causes both holes and false occlusion
  - Recall that 2015 talk uses 300 best occluders from the current frame to fill the holes, and reject occluders near the camera during reprojection to mitigate false occlusion
- Idea: visible objects last frame are likely still visible this frame
- Two-pass:
  - Draw what was visible in the previous frame, build HZB (instead of reprojecting the depth buffer, reproject the geometry!)
  - Depth test the rest

## Visibility Buffer and Deferred Material

We can render more than just depth.

Traditional deferred rendering: visibility is bound with material evaluation, as you output both depth and material properties, causing overdraw of material evaluation. Either you use a pure depth prepass to avoid that, or you have to accept the overdraw, which causes switching of shaders. Also, there is pixel quad inefficiencies when drawing dense meshes (small triangles).

- Visibility Buffer: depth + cluster ID + triangle ID
- Material Shader per pixel: load VisBuffer, instance transform, vertex indices, positions, transform positions; interpolate to get per pixel barycentric coordinates, and finally load and lerp material properties.
- A bit about the terminology: if store UVs or normals in the buffer, it's likely called deferred texturing. Here, even the properties are deferred.
- The output is to G-Buffer. In a forward pipeline, can do shading together with the material shader.

Analysis:

- Materials are one draw per shader, but not too much cost
- Rendering geometries only once still

## Cluster LOD

Idea: Why should we draw more triangles than pixels?

Therefore we need constant time in terms of scene complexity, which implies using LOD.

- Build a tree of clusters, each parent being a simplified versions of its children.
- During runtime, find a cut of the tree that matches the desired LOD. Its view dependent - a parent will draw instead of its children if you can't tell the difference from this point of view.
- Every frame only the necessary cut of the tree in RAM, like virtual texture

Problem: what if neighboring zones select different LODs? Boundaries no longer match.

- Naive solution: lock boundary between clusters during simplification.\
  - Same boundary across different levels, causing dense triangles there.
- Solution: group clusters and force them to make the same LOD selection
  - key idea: alternate group boundaries from level to level, by grouping different clusters
  - A boundary locked in level 0 is likely to be unlocked in level 1, so no dense triangles clustering. Key step: merge and then split.
  - Finally the hierarchy is a DAG, not a tree.

## Tiny Triangles

Software rasterizer is 3x faster than HW when dealing with tiny triangles. HW was designed for triangles larger than one pixel.

**To be reviewed later...**

## Material Shading

- Don't want one material for all pixels.
- Derive material ID through visBuffer info. 
- Draw a full screen quad per unique material, skipping those whose material IDs don't match.
- Material culling: Material ID -> depth value, utilizing depth test (equal)!
- Finer culling: most materials only cover a small portion of the screen. Use tile-based approach: a bitmask to indicate what materials appear in this tile, constructed during material depth write.

## Shadows

**Reviewed later...**
