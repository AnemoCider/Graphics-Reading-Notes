# Improved Culling for Tiled and Clustered Rendering

## GPU divergence

One thing to note: it's bad for a wavefront to cross multiple units (cluster, tile, voxel, etc.), as this causes threads to read from multiple data sources.

Divergence is inevitable.

- Tile
  - in forward, triangles cross tiles
  - deferred, wavefront can match with tile size, which is good
- Cluster
  - forward, triangle cross clusters
  - deferred, wavefront can cross multiple z-slices

Using scalarization to mitigate this.

**To be read later**: scalarization, z-binning, and all the stuff below