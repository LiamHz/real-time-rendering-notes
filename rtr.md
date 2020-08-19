Real-time Rendering - Notes
===========================

## 2 - The Graphics Pipeline
At a high level, there's 4 stages in the graphics pipeline, which can be (and usually are) subdivided into smaller stages.

Pipeline stages execute in parallel, with each stage dependent on the result of the previous stage. This means that the pipeline is as fast as its slowest stage.

### Application Stage
- Developer has full control since this stage is usually executed on the CPU
- Compute shader allows parts of the application stage to execute on the GPU
- Handles tasks like collision detection, user input, and acceleration algorithms such as culling.
- Rendering primitives are created in this stage and passed to the geometry processing stage.

### Geometry Processing
Responsible for most per-triangle and per-vertex operations.

#### Vertex Shader
- Sets per-vertex data such as position and shading info, which are interpolated between vertices when sent to the next stage.
- Vertices are transformed into several different coordinate systems: model space (mesh relative), world space (consistent between meshes), and view space (world oriented so that camera is at origin and is axis aligned - which makes projection and clipping simpler)

#### Projection
- Perspective (or orthographic) projection is applied via a matrix.
- In perspective project the view volume, a frustum, is a truncated pyramid with a rectangular base
- The frustum is transformed into a unit cube, and is now in clip space

#### Clipping
- Primitives entirely inside of the view volume are passed to the next stage of the pipeline as is
- Primitives entirely outside of the view volume are not passed down further down the pipeline
- Primitives that are partially inside the view volume are clipped against the unit cube created by the projection matrix
- Clipping is performed using the homogeneous coordinates
- Perspective division is performed which places the resulting vertices into normalized device coordinates (i.e. from -1 to 1)

#### Screen Mapping
Vertices in NDC are mapped to screen coordinates

### Rasterization
Goal is to find pixels that are inside primitives (triangles). This is called triangle traversal, and can be done with the center of a pixel, or with multiple samples per pixel for MSAA

### Pixel Processing
#### Fragment Shader
Texturing, z-buffer (aka depth buffer), color buffer, and stencil buffer are created / applied here. A swapchain of buffers is used to display the color buffer to the user.

## 3 - The Graphics Processing Unit (GPU)
A GPU has many shader cores, and is optimized for throughput rather than memory latency. This is reflected in the relatively small amount of chip area dedicated to cache memory.

GPUs use "single instruction, multiple data" (SIMD) to separate execution logic from data, allowing the GPU to spend less power processing data and switching.

### History
The first consumer graphics chip to include hardware vertex processing was Nvidia's GeForce256, shipped in 1999. Since then, GPUs have moved from fixed-function pipelines to more customizable systems, primarily through programmable shaders.
### Threads, Warps, and Warp-swapping
Each invocation of the shader program for a specific fragment is called a **thread**. A thread consists of some memory for the input values to the shader, and any register space that is needed for the shader's execution.

A group of threads that use the same shader program is known as a **warp**. Warps on Nvidia GPUs contain 32 threads. All threads in a warp are executed simultaneously.

When a memory fetch is encountered, all threads encounter it at the same time. Instead of stalling for the fetch, the shader core executing it swaps out that warp for a different warp. This **warp-swapping** is the primary latency hiding mechanism used by GPUs.

### Dynamic Branching
Since all threads in a warp execute the same shader program dynamic branching, caused by "if" statements and loops, can affect performance.

If all the threads in a warp take the same branch in a conditional, the warp can continue without care for the other branch. But if just one thread takes the other branch, then the warp must execute both branches, throwing away the results not needed by each particular thread. This is known as **thread divergence**.

### The GPU Pipeline
The **logical model** of the GPU is what you can control via an API, but the **physical model** of the GPU is what actually implements the pipeline and is up to the hardware vendor.

The GPU pipeline implements geometry processing, rasterization, and pixel processing into fixed-function and programmable stages.

A simplified overview of the graphics pipeline consists of 7 stages:

1. **Input Assembler:** Collects the raw vertex data from specified buffers. Optionally, an index buffer can be used to repeat certain elements without duplicating vertex data.

2. **Vertex Shader:** Runs on every vertex and passes per-vertex data down the graphics pipeline. Usually applies transformations to vertices, and converts from model space to screen space.

3. **Tessellation:** Optional. Runs on arrays of vertices ("patches") and subdivides them into smaller primitives.

4. **Geometry Shader:** Optional. Runs on every primitive (triangle, line, point) and can discard or output more primitives. This stage is often not used because its performance isn't great on most graphics cards.

5. **Rasterization:** Discretizes the primitives into fragments (the data necessary to generate a pixel). Fragments that fall outside the screen and fragments that are behind other primitives are discarded. 

6. **Fragment Shader:** Runs on every fragment and determines its color, depth value, and which framebuffer the fragment is written to. Often uses interpolated data from the vertex shader such as surface normals to apply lighting.

7. **Merging:** Applies operations to mix different fragments that map to the same pixel in the framebuffer. Fragments can overwrite each other, or be mixed based on transparency. This is where stencil-buffer and z-buffer operations occur.

## The Central Processing Unit (CPU)
I was encountering frequent references to CPU concepts that affect performance such as caches, cache misses, and SIMD.Below are some notes on these concepts.

The primary source for this info is section 4.5 of [Structured Computer Organization by Tanenbaum][003].

### CPU Cache
#### The Basics
A CPU cache is a smaller and faster memory which stores copies of data from the most frequently used memory locations. The latency of memory accesses from the CPU cache is much lower (~100x) than accesses from main memory.

The CPU cache is divided into fixed-size blocks (typically 4 to 64 bytes) called cache lines, which consists of an sequential index, a tag that points to the index location of the data in main memory, and the data itself.

When the CPU wants to read from or write to a location in main memory, it first checks if that memory location is in the cache by checking the tags of possible corresponding cache lines. If the memory location is in the cache, a _cache hit_ has occurred, otherwise it is a _cache miss_.

Multiple caches are often used, in the form of multi-level caches (covered later) and split caches, where separate caches are used for instructions and data.

#### Replacement Policies
On a cache miss, room is made for a new entry by evicting an existing entry. The evicted entry is chosen using a replacement policy, a common one being Least Recently Used (LRU).

#### Write Policies
Eventually memory written to the CPU cache has to be written to the main memory. The write policy determine when this happens.

Write-through cache: every write to the cache causes a write to main memory.

Write-back cache: cache marks which locations have been written over as dirty. Data in dirty locations is written when it is evicted from the cache.

A cache miss in a write-back cache will require two memory accesses, one to read the new location from memory and the other to write the dirty location to memory.

#### Address Locality
**Spatial locality** is the observation that memory locations with indexes close to a recently accessed memory location are more likely to be accessed in the near future. Prefetching of instructions and data occurs on memory with high spatial locality.

**Temporal locality** occurs when recently accessed memory locations are accessed again. The LRU replacement policy performs better the greater the temporal locality is.

#### Associativity
Associativity refers to the restrictions that the replacement policy has on where it can place main memory in the cache.

**Direct Mapped:** Each entry in main memory can go in just one location in the cache. This allows for **speculative execution** (TODO Define this and tag hints)

**N-way Set Associative:** Each entry in main memory can go in a fixed number of locations in the cache. A 2-way set associative cache can store any particular location in main memory in one of two locations in the cache.

Greater associativity requires more power and potentially time, since more locations have to be searched when checking for a cache hit. However, it also reduces the frequency of cache misses (specifically conflict misses). This is because in a direct mapped cache, if two main memory addresses that are stored in the same cache entry are frequently used, the data in that cache entry will have to be evicted repeatedly causing many conflict misses. Whereas a 2-way associative cache in this scenario would be able to contain both memory addresses.

Typically, only direct mapped, 2-way, or 4-way set associative caches are used.

#### Cache Misses
There are three types of cache misses:

1. A cache read miss from an instruction cache usually causes the most delay, because the execution thread has to wait until the instruction is fetched from main memory.
2. A cache read miss from a data cache usually causes less delay because instructions not dependent on the cache read can still be issued.
3. A cache write miss usually causes the least delay because the write can be queued for later.

There are three categories of cache misses

**Compulsory misses** are caused by the first reference to a piece of data. Cache size and associativity don't affect the frequency of these misses, but it can be mitigated by prefetching, where instructions and data with high spatial locality are fetched from main memory and copied to the cache before they're needed.

**Capacity misses** occur because the cache size is finite. EX: Iterating through a array with more data entries then there are cache lines.

**Conflict misses** occur because the cache evicted a specific entry earlier. These misses can be broken down into _mapping misses_, they're unavoidable given a certain amount of associativity, and _replacement misses_, which are due to the particular data entry being evicted previously.

#### Multi-level Caches
Larger caches have better hit rates but higher latency. This is often addressed by having multiple levels of caches, and checking the smaller and faster level 1 (L1) cache first, and on a miss checking checking the larger and slower L2 cache (and so on) before accessing main memory.

Having two levels of caches is common, and occasionally L3 caches - Intel's core i7 being one of the more common CPUs that use L3 caches.

Most modern L1 caches have hit rates greater than 95%.

### Speculative Execution
### Out-of-Order Execution
### Branch Prediction
## <!-- Links -->
[002]: https://en.wikipedia.org/wiki/CPU_cache
[003]: https://www.goodreads.com/book/show/457107.Structured_Computer_Organization
