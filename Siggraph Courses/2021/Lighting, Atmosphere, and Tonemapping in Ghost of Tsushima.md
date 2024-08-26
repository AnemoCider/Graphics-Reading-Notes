# Lighting, Atmosphere, and Tonemapping in Ghost of Tsushima

## Indirect Diffuse

### Placing

- Relatively coarse grid
- Use additional tetrahedral meshes when needed, and blend

### Baking and update

Need to support dynamic TOD, so baking irradiance is not feasible.

- Offline capture sky visibility, encoded into 2nd SH
- Runtime, project sky to SH, multiplied by visibility to update the probe, and finally convolve with lambertian cos probe
- What about bounced sky light?
  - transfer matrix is too much memory, and too long to capture
  - approx: skylight is constant over the hemisphere
  - light the world with a uniform white sky, then capture "bounced sky visibility" and project to SH
  - at runtime, multiply the bounced visibility by averaged skylight, adding it to the direct skylight
- What about bounced sun/moon light?
  - Reuse bounced sky visibility somehow
  - Assume diffuse sunlight comes from fully reflective floors and walls. So use imaginary planes to generate these directions and project to SH. Then scale it by light color, and multiply it by bounced visibility.
  - Essentially, approximating the sunlight shadowing by cos-weighted sky visibility

### Directionality boost

Use 2nd SH, but this is too flat, so lerp towards delta in direction of linear SH maximum.

### Light Leaking

Most straightforward solution: Use fewer interiors, and use thick walls.

However we have thin walls and lots of interiors...

Solution (applied to tet mesh probes):

- classify probes as interior or exterior
- Assign all surfaces an interior mask value w, by default 0.5
- To perform lighting, calculate tet barycentric coordinates as usual, but then multiply each one by: probe is interior? w : 1-w, and uses weighted radiance blend
- Intuitively, if the surface is interior, assign largeer weights to interior probes.

## Indirect Specular

Reflection probes.

- Offline, captures 256x256x6 cubemaps (mini G-buffer)
  - Albedo
  - Normal + depth
- Relight (use the mini G-buffer to get specular lighting)
  - Update one probe per frame
  - Use shadowmap to estimate occlusion

**...and Other Details**

