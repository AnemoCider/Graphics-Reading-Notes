# Global Illumination Based on Surfels

GIBS

## Surfel

Surfel: position + radius + normal

### Surfelization

Spawn surfels from the G-Buffer:

- In general, filling gaps in screen space
- Split the screen into 16x16 texel tiles. Each tile finds the least covered texel and spawns a surfel there, until coverage threshold is reached.
- Surfels are persistent (even if go out of view), and positions are updated every frame.
  - Achieved by using a global transform buffer for transforms of any geometry, and writing IDs to the G-Buffer.
  - Surfels get ID once spawned, and store local(relative) position to the transform of the geometry
  - ID->transform, transform + local -> world position

### Deal with Surfel Scaling

- Purpose: maintain a constant level of quality across distances. Surfels shrink/grow with geometry in the screen space.
- When surfels shrink, generate more; when grow, remove. Maintaining a constant coverage.

### Memory Management

- All resources allocated up front, so there are limits on max number.
- Each surfel is a fixed size buffer. 
- Stack-based (on GPU) approach: use a stack that contains indirection to the whole surfel space.
- Decrement the stack and get an ID during spawning, increment and push ID back during removal.
- Recycling heuristic, based on:
  - how many surfels are alive
  - the time since the surfel last contributed to lighting
  - how far away the surfel is

### Accel Structure

Purpose: needs to find which surfels are in the vicinity of a given position.

- Use a uniform grid
- Insert a surfel into cells that it overlaps, and immediate neighbors, to facilitate later access and avoid discontinuities.
  - Restriction: a surfel radius should always be smaller than the side length of a cell
  - Distant surfels have 