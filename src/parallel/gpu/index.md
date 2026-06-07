# GPU Programming

GPUs are the most powerful parallel processors available: an NVIDIA H100 has 16,896 CUDA cores delivering 60 TFLOPS of FP32 and 3 TB/s of memory bandwidth. A high-end CPU delivers 1-2 TFLOPS and 100 GB/s. The 30-60× gap is real, but only for workloads that match the GPU's execution model: thousands of threads executing the same instruction on different data (SIMT: Single Instruction, Multiple Threads).

## Why GPUs Are Different

A CPU optimizes for low latency: branch prediction, out-of-order execution, deep cache hierarchies. It spends transistors to make a single thread run fast.

A GPU optimizes for throughput: it runs tens of thousands of threads, hiding memory latency by switching between warps (groups of 32 threads). When one warp stalls on a memory load, the GPU immediately switches to another warp that's ready to execute. This is zero-cost context switching — the warp state is already in registers.

The cost: threads within a warp execute in lockstep. If threads diverge (take different branches), the warp executes both paths sequentially, with threads masked off. A `if (threadIdx.x % 2 == 0)` branch cuts warp throughput in half.

## The SIMT Execution Model

Key concepts:
- **Thread**: the smallest unit of execution. Each thread has its own registers and can access a unique `threadIdx`.
- **Warp**: 32 threads that execute together on a single streaming multiprocessor (SM). All threads in a warp execute the same instruction at the same time.
- **Block**: a group of threads (up to 1024) that cooperate via shared memory and synchronization (`__syncthreads()`). Threads in the same block run on the same SM.
- **Grid**: the collection of all blocks for a kernel launch.

```cuda
// CUDA kernel: each thread computes one element of vector addition
__global__ void vec_add(float *a, float *b, float *c, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < n)
        c[idx] = a[idx] + b[idx];
}

// Launch: 1024 threads per block, enough blocks to cover n elements
int blocks = (n + 1023) / 1024;
vec_add<<<blocks, 1024>>>(d_a, d_b, d_c, n);
```

Each thread computes its global index from `blockIdx`, `blockDim`, and `threadIdx`. The hardware schedules warps within each block; all warps in all blocks execute the same `vec_add` kernel.

## Memory Hierarchy

GPU memory is explicitly managed (unlike CPU, where the cache hierarchy is transparent):

| Memory Type | Size (H100) | Latency | Scope | Speed (vs. global) |
|------------|-------------|---------|-------|---------------------|
| **Registers** | 256 KB/SM (64K × 32-bit) | ~0 cycles | Per-thread | ~200× |
| **Shared memory** | 228 KB/SM | ~20 cycles | Per-block | ~100× |
| **L1 cache** | 256 KB/SM | ~30 cycles | Per-SM | ~80× |
| **L2 cache** | 50 MB | ~200 cycles | All SMs | ~10× |
| **Global memory (HBM)** | 80 GB | ~300-800 cycles | All SMs | 1× (baseline) |

The programmer explicitly manages data movement: load from global → shared memory → registers, compute, write back. This is tedious but gives precise control over bandwidth utilization.

### Coalesced Memory Access

Threads in a warp should access contiguous memory addresses. If warp lane `i` accesses address `base + i * 4`, the hardware combines 32 requests into one 128-byte transaction. If threads access scattered addresses, each request becomes a separate transaction — 32× the bandwidth.

```cuda
// Coalesced: thread i accesses a[i] (contiguous)
float val = a[threadIdx.x];  // 1 transaction

// Strided: thread i accesses a[i * 1024] (strided by 1024)
float val = a[threadIdx.x * 1024];  // 32 transactions (worst case)
```

### Shared Memory Bank Conflicts

Shared memory is divided into 32 banks (one per warp lane). If two threads in the same warp access different addresses in the same bank, the accesses are serialized. The rule: `bank = (address / 4) % 32`. Consecutive 4-byte words map to consecutive banks — ideal for coalesced access. Strided access by 32 creates bank conflicts.

```cuda
__shared__ float smem[1024];

// No bank conflicts: consecutive threads access consecutive words
float val = smem[threadIdx.x];  // Perfect

// 32-way bank conflict: all threads access bank 0
float val = smem[threadIdx.x * 32];  // 32× serialization
```

## Occupancy and Latency Hiding

**Occupancy** = active warps per SM / maximum warps per SM. Higher occupancy means more warps to switch between when one stalls. The H100 supports 64 warps (2048 threads) per SM.

For memory-bound kernels: low occupancy means the SM has fewer warps to hide latency → the SM stalls waiting for memory. Target: 50%+ occupancy.

For compute-bound kernels: occupancy matters less — the SM is busy executing, not waiting.

The occupancy limiters:
- **Registers per thread**: if each thread uses 128 registers, the SM can fit 256 KB / (128 × 4 bytes) = 512 threads = 16 warps → 25% occupancy.
- **Shared memory per block**: if each block uses 100 KB of shared memory, only 2 blocks fit per SM (228 KB/SM). With 256 threads/block → 16 warps → 25% occupancy.

## Reduction on GPU

Summing an array uses shared memory for the reduction tree:

```cuda
__global__ void reduce(float *input, float *output, int n) {
    __shared__ float sdata[256];
    
    int tid = threadIdx.x;
    int idx = blockIdx.x * blockDim.x * 2 + tid;
    
    // Coalesced load into shared memory
    sdata[tid] = (idx < n ? input[idx] : 0) + 
                 (idx + blockDim.x < n ? input[idx + blockDim.x] : 0);
    __syncthreads();
    
    // Reduction in shared memory
    for (int stride = blockDim.x / 2; stride > 0; stride /= 2) {
        if (tid < stride)
            sdata[tid] += sdata[tid + stride];
        __syncthreads();
    }
    
    // Write result
    if (tid == 0)
        output[blockIdx.x] = sdata[0];
}
```

This kernel reduces 2 × 256 = 512 elements per block to 1 element, using shared memory for the reduction tree. A second kernel launch reduces the block results further. For n = 10⁶: first pass produces ~2000 partial sums, second pass reduces to 1.

Performance on H100: ~3 TB/s memory bandwidth. Reduction of 1B floats: ~10 ms (compare with ~100 ms on a 32-core CPU).

## Matrix Multiplication on GPU

The canonical GPU kernel. Each block computes a tile of the output matrix:

```cuda
__global__ void matmul(float *A, float *B, float *C, int N) {
    __shared__ float As[TILE][TILE];
    __shared__ float Bs[TILE][TILE];
    
    int row = blockIdx.y * TILE + threadIdx.y;
    int col = blockIdx.x * TILE + threadIdx.x;
    float sum = 0;
    
    for (int t = 0; t < N / TILE; t++) {
        // Coalesced load into shared memory
        As[threadIdx.y][threadIdx.x] = A[row * N + t * TILE + threadIdx.x];
        Bs[threadIdx.y][threadIdx.x] = B[(t * TILE + threadIdx.y) * N + col];
        __syncthreads();
        
        // Compute partial dot product from shared memory
        for (int k = 0; k < TILE; k++)
            sum += As[threadIdx.y][k] * Bs[k][threadIdx.x];
        __syncthreads();
    }
    
    C[row * N + col] = sum;
}
```

TILE = 16: shared memory = 2 × 16 × 16 × 4 bytes = 2 KB per block. Registers: ~64 per thread. Occupancy: high. Performance: ~30 TFLOPS on H100 (50% of peak) — limited by memory bandwidth, not compute.

Optimizations: double buffering (load next tile while computing current), warp-level matrix multiply (using `mma.sync` tensor cores), and register tiling (each thread computes a 4×4 output sub-matrix for better register reuse).

## Key Lessons

1. **GPUs are throughput machines, CPUs are latency machines.** A GPU hides memory latency with massive parallelism; a CPU avoids latency with caches and prefetching. The programming models reflect this.
2. **Coalesced memory access is non-negotiable.** Uncoalesced access is 32× slower. Always arrange data so that threads in a warp access contiguous addresses.
3. **Shared memory is manually managed L1.** Use it for inter-thread communication (reductions), data reuse (matrix multiply tiles), and to enforce coalescing (load scattered data into shared memory, then access coalesced from shared memory).
4. **Occupancy determines how well you hide latency.** More active warps = more ability to switch when one stalls. But max occupancy isn't always optimal — sometimes fewer threads with more registers each are faster (compute-bound kernels).
5. **GPU programming is explicit parallelism.** Every thread index, every shared memory byte, every synchronization point is in the code. This is harder than OpenMP but gives full control over the hardware.
