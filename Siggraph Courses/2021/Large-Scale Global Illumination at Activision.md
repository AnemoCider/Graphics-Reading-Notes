# Large-Scale Global Illumination at Activision

## Representing Lighting - Warped Irradiance Volumes

- Memory limitation is an issue
- idea: Warp the irradiance volume domain to encompass the sparse detail areas around the interesting parts of the scene

Steps:

- Shoot vertical rays from both the above and below to obtain depth bounds
- Obtain the shaded detail region
- Warp(remap) the region to a regular 3d rectangular grid
  - Use adaptive height resolution

## Handling light leaks - Sampling Volumetric Lighting

- Possible solutions: split indoor and outdoor volumes; use geometry-aware reconstruction filter, e.g., take the normal into account
- Visibility based sample validation
  - Observation: The major problem is probe indoor leaks to outdoor, not the inverse. We note that, camera is in most time outdoor, and is far away from the probe.
  - Sample radiance within the voxel, with many sample points
  - For each sample point, shoot several visibility validation rays
  - If the intersection is quite near, the sample becomes invalid.

## Handling constraints of SH - Constrained SH

- Unconstrained least squares projection to SH easily causes ringing artifacts and color shifts
  - By projection we mean a least square fit to a lower-order representation, for SH it's simply dropping the higher orders
- Possible solutions
  - Windowing: shrink higher-order SH towards 0 until reconstruction becomes non-negative. However, this may not be enough to prevent color distortion.
  - Modify reconstruction, see 2020 Precomputed Lighting Advances in CoD and Hazel's work in 2017
- More general constraints?

**Reproducing kernels, to be read later**

## Compression - MBD

- source data is 1.5GB