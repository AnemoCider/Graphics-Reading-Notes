# GPU Driven Rendering Pipeline

## Mesh cluster

At that time there may not exist multi-draw indirect support (seems to came out in 2019?), so to render different meshes with gpu instancing in one DC, the only way is to enforce a shared topology - cluster. (Also, a multiple of 64 may also be friendly to the GPU)

We split meshes into clusters of 64 vertex strips (62 triangles per cluster). Add degenerate vertices for padding.

- One draw call for arbitrary number of meshes
- Fine-grained GPU culling

## Rendering Pipeline

### CPU

- Coarse frustum culling
  - instance level, using quad tree
- Update instance data and build batch hash
  - data includes transform, LOD factor, etc.
  - For those non-GPU-instanceable data, build a hash and batch instances based on it
  - this gives a stream of instances: a list of offsets into a GPU buffer, one per instance

### GPU

- GPU instance culling: frustum and occlusion
  - generates a list of cluster chunks, each containing at most 64 clusters
- Cluster chunk expansion
  - why not directly expand each instance to clusters: number of clusters of each instance varies drastically, causing unbalanced workload between threads within a wavefront
- Cluster culling: frustum + occlusion + backface
  - Clusters that pass export a mask that indicates triangles to send the next phase
- Index compaction
  - One wavefront handles one cluster, and each thread handles one triangle, writing data into a dynamic index buffer

### CPU again

- MultiDrawIndexInstancedIndirect per batch

## Bake static triangle backface

At each cluster's center generate a cube, each face exporting triangles that are visible from that side.

## Depth culling

- Depth prepass with 300 best occluders, full resolution
- Downsample to 512x256
- Combine with 1/16 resolution depth buffer from previous frame (reprojected)
  - Holes exist, but this is conservative so it's fine
  - False occlusion exists, cause there are moving objects in last frame's depth, especially large occluders can cause problem. So reject occluders near the camera during reprojection.
- Generate depth hierarchy

## Camera Depth Reprojection

Aims at removing shadow casters that cannot have any receivers, e.g., those objects already invible from the camera.

- From the camera's view, for each 16x16 tile, render a frustum, reproject to light space, and calculate it's maximum depth there
- Use this to depth test when generating the shadow map

## Virtual Texturing

Keep only the visible set of texture data in memory.

- 256k^2 virtual address space, splitted into 128^2 texel pages; 8k^2 physical address
- One single texture binding is sufficient! No longer need to batch by texture.
- Utilizes the bindless feature
- Can cache complex results (like material blending and decal rendering) in VT as well

Deferred VT: 

- Traditional: in G-Buffer, store material ID (16) + UV(2 * 16) + LOD factor (16) or (possibly) UV derivative (4 * 16), to support anisotropic filtering
  - G-Buffer would be too large
- Modifications
  - UV into the VT
  - Calculate UV derivatives in screen space, i.e., the UV Gbuffer. Needs to address material discontinuities between neighboring pixels.
  - MSAA trick: UV can be interpolated, so render a low-res G-Buffer and interpolate (MSAA) the missing data

## Culling

Two-phase:

- Use last frame's depth pyramid to cull objects and clusters
- Render visible object's depth, test against culled objects and clusters to avoid false occlusion
  - all happen on GPU, otherwise high latency

## Virtual Shadow Mapping

- Split shadow map into pages
- For every pixel in the depth buffer, reconstruct world position and project to light space
- Mark the corresponding shadow map pages as used
- Render needed pages only

