# Multi-Scale Global Illumination in Quantum Break

Needs quasi-dynamic (destruction), so mesh-based baking, like PRT and lightmap, are not feasible. Also needs TOD.

## Large Scale Lighting

### Indirect Diffuse

Irradiance volume:

- irradiance from local light sources
- indirect sunlight transport
- sky light transport

### Indirect Specular

PRT not possible, a decent angular resolution (think about the matrix...) would require too much data.

A very classical solution: SSR + reflection probe.

Probe:

- Addressing light leaking: store probe visibility in the volume. For each texel, trace rays and find scene intersections, then trace visibility rays to the probe from the intersections.
  - Side note: why not just set an effective radius for each probe...

## Screen Space Lighting

Provides fully dynamic, finer scale details.

- SSAO
- Screen-space diffuse lighting