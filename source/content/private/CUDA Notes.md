### Terms

- **Memory Coalescing** - Memory coalescing refers to **efficient global memory access** where consecutive threads **access consecutive memory locations**, allowing the GPU to combine these accesses into a **single transaction**.
- **Warp** - A warp is **a set of 32 threads within a thread block such that all the threads in a warp execute the same instruction**. These threads are selected serially by the SM. A warp comprises 32 _lanes_, with each thread occupying one lane.
- **Warp divergence** - Warp divergence refers to a situation in which threads within a warp of a GPU execute different instructions, causing a slowdown in computation. Can be solved by using **sequential threads**.
- **Warp shuffles** - A mechanism for moving data between threads within a warp. Is faster than using shared memory. 