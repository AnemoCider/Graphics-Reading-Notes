# Pipeline Optimization

## GPU Performance Measurement

- Note that simple benchmarks may be misleading
  - instructions per second?
  - floating point operations per second?
  - gigahertz?
- In practice:
  - measure the wall clock times (i.e., real-world times) for various real programs, and compare
  - most straightforward: FPS
  - details:
    - turn off double buffering and vsync, because otherwise the FPS will be synced with the monitor refresh rate

## Optimization

### Application Stage: CPU

Just code optimization...

- Memory access patterns are crucial
  - Store sequentially accessed data in contiguous memory locations, e.g., ECS
- Avoid pointer indirection, jumps, and function calls
  - pointer invalidation: prevents data prefetching, because the CPU can hardly predict the next memory access
  - jump: add branches, not easy to predict, may lead to useless instruction prefetching and stall the pipeline
  - function calls: like jump. the instructions may not be in the cache
- Aligning frequently used data structures to multiples of the cache line size
  - avoid false sharing

### API calls

#### State Changes

- State changes are expensive
  - Idea: grouping objects with a similar state
- Costs of state changes (expensive to cheap):
  - Rendering mode: compute or rendering
  - Framebuffer
  - Shader Program
  - Blend Mode
  - Texture bindings
  - Vertex format
  - UBO
  - Vertex bindings
  - Uniform updates
- Batching: grouping (sorting) objects with the same framebuffer, then shader, then blend mode, etc.
- Restructure data layout
  - texture array
  - use if statement in one shader to handle different materials

#### Consolidating and Instancing

Issue: preparing and sending each draw call is expensive, on CPU. Too many draw calls cause GPU to starve.

- Consolidation
  - Consolidate several objects into a single mesh
  - Useful for (relatively) static objects that use the same state
  - Potential issues: large batches can make other algorithms, e.g., culling, less effective
  - In practice: use, e.g., BVH, to group nearby objects
- Instancing
  - Draw one object several times in a single draw call
    - Specify unique attributes and add noise to each instance
    - In practice: use instance draw API (recommended), or use geometry shader, which can duplicate the data of an incoming mesh (may be slower)

### Geometry Stage

- efficient culling
  - utilize compute shaders
- reducing per-vertex lighting time
  - baking
  - trim down local lighting to reduce computation

### Rasterization Stage

- Backface culling

### Fragment Stage

- Texture level optimization
  - Issue: sampling a texture is expensive
  - load only the mip levels needed, and apply texture compression
    - these use less memory, leading to lower transfer and access times, and may improve cache performance
  - Level of detail
  - early-z, or even z-prepass
    - utilize early-z by (roughly) sorting and drawing near to far
    - **note that** sorting by depth conflicts with sorting by state, so we need to balance them
      - e.g., create a sorting key that encodes both depth and state

### Framebuffer

Issue: Rendering incurs many accesses to the framebuffer, adding pressure on the cache

- Reduce the size of each fragment
  - `chroma subsampling`: human vision is less sensitive to color than to brightness
    - decompose the image into luminance and chroma
    - luminance stored at full resolution, chroma at lower
    - Mentioned in: [Frostbite: More performance](https://www.ea.com/frostbite/news/more-performance-five-rendering-ideas-from-battlefield-3-and-need-for-speed-the-run)

### Merging state

