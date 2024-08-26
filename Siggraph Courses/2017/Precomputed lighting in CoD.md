# Precomputed lighting in Call of Duty: Infinite Warfare

## Lightmaps

- For large, structural geometries
- AHD, ambient + highlight direction + highlight
  - Interpolation may be a problem, when light comes from different directions, color bleeding can be inaccurate.
- H-basis: ring. De-ringing causes lost of directional contrast.


## Approximating the Rendering Equa.

- Previous: for each object, precompute lighting in only one location
  - 3rd SH
  - convolve with cosine kernel on the fly
  - flat and uninteresting
- Idea
  - Lighting term changes smoothly, while visibility changes rapidly across an object.
  - Decouple them and treat separately. Lighting is unique to every object, but can be stored in a lower resolution (because of its low frequency); visibility (self visibility) can be shared among instances.
- Visibility
  - Per-vertex
  - As a cone. Axised along the main unoccluded direction, an angle for unoccluded portion, and a scale for more flexibility, e.g., when the vertex is occluded from multiple directions.
  - Baked only once for one type of mesh, shared among instances.
  - Encoded by 4th order SH. Optimal Linear Direction for axis, and fit others.
- Lighting
  - Attempt: still one probe/object, use irradiance gradients. Not satisfactory.
  - Scatter sampling locations across the object and interpolate later
  - Store indices of the probes to use, plus the weights for interpolation
  - Static objects have lighting baked, while dynamic ones sample volumetric structure
- Combine together
  - Rotate lighting SH to visibility's frame, where the later is just a ZH. Dot product then rotate back.
  - Could be optimized using maths.

## Light Grid

- Placed using a very complex tetrahedralization algorithm
- To prevent leaking, store visibility info
  - Specifically, a depth map that stores the barycentric coordinates of the first intersection on the ray to the opposite triangle of the tetrahedron. 15 values, 8 bits each.

## PP Sloan Part

- Some insights with playing around with ZH, for, e.g., deringing
- Baking: first visibility, and then solve for the optimal lighting coefficients that minimize the error when multiplied with the visibility and interpolated
  - Very Sloan style...