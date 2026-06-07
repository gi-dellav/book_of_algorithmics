# CUDA

CUDA (Compute Unified Device Architecture) is NVIDIA's platform for GPU programming. It extends C++ with GPU-specific constructs: kernel functions (`__global__`), device memory management (`cudaMalloc`, `cudaMemcpy`), and synchronization primitives (`__syncthreads`, `cudaStream`). Understanding CUDA means understanding the hardware it abstracts.

## The CUDA Programming Model

A CUDA program has two components:
1. **Host code** (CPU): manages memory, launches kernels, handles I/O.
2. **Device code** (GPU): kernels that execute in parallel across thousands of threads.

```cuda
#include <cuda_runtime.h>
#include <stdio.h>

__global__ void hello() {
    printf("Hello from thread %d of block %d\n", threadIdx.x, blockIdx.x);
}

int main() {
    hello<<<4, 256>>>();  // 4 blocks × 256 threads = 1024 threads
    cudaDeviceSynchronize();
    return 0;
}
```

The `<<<blocks, threads>>>` syntax launches a grid of threads. Each block has up to 1024 threads. The grid can have up to 2³¹-1 blocks in the x dimension. Threads within a block can cooperate; blocks are independent.

## Memory Management

CUDA memory is explicitly managed. The programmer allocates, transfers, and frees memory:

```cuda
int n = 1000000;
size_t size = n * sizeof(float);

float *h_a = (float*)malloc(size);     // Host memory
float *h_b = (float*)malloc(size);
float *h_c = (float*)malloc(size);

// Initialize host arrays
for (int i = 0; i < n; i++) {
    h_a[i] = i; h_b[i] = 2 * i;
}

float *d_a, *d_b, *d_c;
cudaMalloc(&d_a, size);  // Device memory
cudaMalloc(&d_b, size);
cudaMalloc(&d_c, size);

cudaMemcpy(d_a, h_a, size, cudaMemcpyHostToDevice);
cudaMemcpy(d_b, h_b, size, cudaMemcpyHostToDevice);

// Launch kernel
vec_add<<<(n + 255) / 256, 256>>>(d_a, d_b, d_c, n);

cudaMemcpy(h_c, d_c, size, cudaMemcpyDeviceToHost);

cudaFree(d_a); cudaFree(d_b); cudaFree(d_c);
free(h_a); free(h_b); free(h_c);
```

PCIe bandwidth (host ↔ device): ~16 GB/s (PCIe 4.0 ×16) to ~50 GB/s (PCIe 5.0). This is the primary bottleneck for GPU computing — data must be on the device for computation. Minimize host-device transfers.

## Unified Memory

CUDA 6+ supports **unified memory**: a single pointer accessible from both host and device. The driver migrates pages automatically:

```cuda
float *a;
cudaMallocManaged(&a, size);  // Accessible from both host and device

// Initialize on host
for (int i = 0; i < n; i++) a[i] = i;

// Launch kernel — pages fault to device on access
kernel<<<blocks, threads>>>(a, n);
cudaDeviceSynchronize();

// Access result on host — pages fault back to host
printf("Result: %f\n", a[0]);

cudaFree(a);
```

Simpler than explicit `cudaMemcpy`, but page fault overhead (several µs per fault) can be significant. For large arrays, use `cudaMemPrefetchAsync` to hint migration before the kernel.

## Streams and Overlap

CUDA **streams** enable concurrent execution: kernel on stream A overlaps with data transfer on stream B:

```cuda
cudaStream_t stream1, stream2;
cudaStreamCreate(&stream1);
cudaStreamCreate(&stream2);

// Transfer to device on stream 1
cudaMemcpyAsync(d_a, h_a, size, cudaMemcpyHostToDevice, stream1);

// Kernel executes on stream 2 (needs d_b which is already on device)
kernel<<<blocks, threads, 0, stream2>>>(d_b, d_c, n);

// Overlap: transfer on stream1 happens concurrently with kernel on stream2

cudaStreamSynchronize(stream1);
cudaStreamSynchronize(stream2);
```

Streams are the key to hiding PCIe latency. A typical pattern: double-buffer inputs. While kernel K processes buffer N, transfer buffer N+1 to the device. When K finishes, immediately launch K+1 on buffer N+1 while copying results from buffer N back to the host.

## Warp-Level Primitives

CUDA provides warp-level operations that avoid shared memory and `__syncthreads`:

```cuda
// Warp vote: do all threads in the warp agree?
int result = __all_sync(0xFFFFFFFF, condition);  // Returns 1 if all threads' condition is true

// Warp shuffle: exchange registers between warp lanes
float val = __shfl_down_sync(0xFFFFFFFF, my_val, 1);  // Get my_val from lane tid+1

// Warp reduce using shuffle
float warp_reduce(float val) {
    for (int offset = 16; offset > 0; offset /= 2)
        val += __shfl_down_sync(0xFFFFFFFF, val, offset);
    return val;  // All lanes now have the sum
}
```

Warp shuffle is 5× faster than shared memory for reduction (register-to-register communication, ~5 cycles per shuffle vs. ~20 cycles for shared memory load). It's the key technique for high-performance GPU reductions and scans.

## Tensor Cores

Tensor Cores (Volta+, 2017) perform 4×4 matrix multiply-accumulate in one instruction:

```cuda
// Input: A (16×16 FP16), B (16×16 FP16), C (16×16 FP32 accumulator)
// Output: C += A × B  (one warp instruction)

// Using wmma (Warp Matrix Multiply-Accumulate) API
#include <cuda_fp16.h>
#include <mma.h>

using namespace nvcuda;

wmma::fragment<wmma::matrix_a, 16, 16, 16, half, wmma::row_major> a_frag;
wmma::fragment<wmma::matrix_b, 16, 16, 16, half, wmma::row_major> b_frag;
wmma::fragment<wmma::accumulator, 16, 16, 16, float> c_frag;

// Load fragments from shared memory
wmma::load_matrix_sync(a_frag, smem_a, 16);  // 16 = leading dimension
wmma::load_matrix_sync(b_frag, smem_b, 16);
wmma::fill_fragment(c_frag, 0.0f);

// Single instruction: C += A × B for 16×16 matrices
wmma::mma_sync(c_frag, a_frag, b_frag, c_frag);

// Accumulate result
wmma::store_matrix_sync(smem_c, c_frag, 16, wmma::mem_row_major);
```

On H100, a Tensor Core does 1024 FP16 FMA operations per clock (one warp instruction). At 1.8 GHz: ~1.8 × 1024 × 128 (SMs) ≈ 235 TFLOPS. This is the source of the 60× gap vs. CPU — specialized matrix hardware.

For matrix multiplication, use `cublasGemmEx` (cuBLAS library) which uses Tensor Cores automatically. For custom kernels, the `wmma` API or the newer `mma` (PTX-level) gives direct access.

## Performance Optimization Checklist

1. **Maximize occupancy** (within register/shared memory limits). Use CUDA Occupancy Calculator.
2. **Coalesce memory access**: consecutive threads access consecutive addresses.
3. **Use shared memory** for data reuse and inter-thread communication.
4. **Avoid warp divergence**: minimize `if (threadIdx.x % 2)` branches within warps.
5. **Overlap kernel execution with data transfer** using streams.
6. **Use Tensor Cores** for matrix operations (automatic with cuBLAS).
7. **Minimize PCIe transfers**: keep data on device between kernel launches.
8. **Profile with `nvprof` and Nsight Compute**: GPU profiling is mature and precise.

## Key Lessons

1. **CUDA is low-level by design.** It exposes the hardware — threads, warps, shared memory, streams — because GPU performance depends on these details. Higher-level libraries (cuBLAS, cuDNN, Thrust) are built on CUDA.
2. **Data movement dominates runtime.** PCIe is 10-30× slower than GPU memory bandwidth. Overlap transfers, use unified memory carefully, and keep data on device.
3. **Streams are the key to GPU utilization.** Without them, the GPU is idle during host-device transfers. With double buffering and streams, you can approach 100% utilization.
4. **Tensor Cores are the GPU's real advantage.** FP32 FLOPS: CPU 2 TFLOPS, GPU 60 TFLOPS. Tensor Core: GPU 235 TFLOPS (FP16). For matrix operations, the gap is 100×, not 30×.
5. **GPU code is portable within NVIDIA hardware.** The same CUDA code runs on a laptop RTX 4060 and a data-center H100 (with different occupancy and performance, but correct). This is the primary reason CUDA dominates GPU computing.
