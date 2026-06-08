# Distributed Training

Training a model with 175B parameters (GPT-3) or 1.7T parameters (GPT-4) requires thousands of GPUs working together. This article covers the parallelism strategies — data, model, tensor, pipeline — that make this possible, and the communication primitives (all-reduce, all-gather, reduce-scatter) that they depend on.

## The Communication Primitives

NVIDIA's NCCL library provides optimized collective operations:

| Operation | What it does | Bandwidth utilization |
|-----------|-------------|----------------------|
| All-reduce | Sum gradients across all GPUs, result on all | ~90% of NVLink |
| All-gather | Concatenate data from all GPUs, result on all | ~90% |
| Reduce-scatter | Sum data, scatter result across GPUs | ~90% |
| Broadcast | Copy data from one GPU to all others | ~100% |

All-reduce is the most important for data parallelism. The **ring all-reduce** algorithm achieves asymptotically optimal bandwidth:

```
For N GPUs, each with D bytes of data:
  1. Scatter-reduce: each GPU sends D/N bytes N-1 times (total: D·(N-1)/N bytes sent/received per GPU)
  2. All-gather: each GPU sends its accumulated D/N bytes N-1 times
  Total: 2D·(N-1)/N bytes per GPU ≈ 2D for large N
```

```rust
// Ring all-reduce pseudocode (simplified)
fn ring_all_reduce(gradients: &mut [f32], rank: usize, world_size: usize) {
    let chunk_size = gradients.len() / world_size;
    let left = (rank + world_size - 1) % world_size;
    let right = (rank + 1) % world_size;

    // Scatter-reduce: N-1 steps
    for step in 0..world_size - 1 {
        let send_chunk = (rank - step + world_size) % world_size;
        let recv_chunk = (rank - step - 1 + world_size) % world_size;

        // Async send/recv the chunk
        send(&gradients[send_chunk * chunk_size..], right);
        recv(&mut temp_buffer, left);
        // Add received data to our local chunk
        for i in 0..chunk_size {
            gradients[recv_chunk * chunk_size + i] += temp_buffer[i];
        }
    }

    // All-gather: N-1 steps (similar pattern, but no addition)
    // ...
}
```

On an 8-GPU node with NVLink (600 GB/s), ring all-reduce of 1 GB of gradients takes ~17 ms. This is the lower bound for how frequently you can synchronize gradients.

## Data Parallelism

Each GPU holds a full copy of the model. Each GPU processes a different micro-batch. After backward, gradients are all-reduced. All GPUs then apply the same optimizer step (identical result everywhere).

```
GPU 0: forward(batch_0), backward(), all_reduce(grads), step()
GPU 1: forward(batch_1), backward(), all_reduce(grads), step()
...
```

Effective batch size = per_gpu_batch_size × num_gpus. For GPT-3 (175B params), per-gpu batch size is 2 (fits in 80 GB), so 1024 GPUs give effective batch size 2048 — matching the training recipe.

Problem: each GPU stores the full model (175B × 4 bytes = 700 GB in FP32) and optimizer states (2× for Adam m/v = 1.4 TB). Total: 2.1 TB per GPU. An A100 has 80 GB. Data parallelism alone can't train models > ~10B parameters.

## ZeRO (Zero Redundancy Optimizer)

DeepSpeed's ZeRO partitions the optimizer state across GPUs:

- **ZeRO-1**: Partition optimizer states (m, v in Adam) across GPUs. Each GPU holds 1/N of the optimizer states. Memory: 4× params / N.
- **ZeRO-2**: Also partition gradients. Memory: 2× params / N.
- **ZeRO-3**: Also partition parameters. Each GPU holds only 1/N of the model at any time; all-gathers parameters before each layer's forward, discards after backward.

```
ZeRO-3 forward pass for layer L:
  all_gather(parameters_L)  # collect parameters from all GPUs
  compute forward for layer L
  discard parameters_L      # free memory for next layer
```

ZeRO-3 reduces per-GPU memory from O(params) to O(params/N), enabling training of models with 100B+ parameters on 8 GPUs. The cost: extra all-gather communication per layer (asynchronous, can be overlapped with computation).

## Tensor Parallelism

Split individual layers across GPUs. For a linear layer Wx:

- **Row-parallel**: Split W by rows. GPU i computes (W_i @ x) — partial result. All-reduce to get full result.
- **Column-parallel**: Split W by columns, x by corresponding segments. GPU i computes W_i @ x_i. No communication needed for the forward pass.

Megatron-LM applies tensor parallelism to transformer layers:
```
Attention: column-parallel for QKV projection, row-parallel for output projection
MLP: column-parallel for first linear, row-parallel for second
```

Each transformer layer does 2 all-reduces (forward) + 2 all-reduces (backward) = 4 all-reduces. For a 96-layer model, that's 384 all-reduces per iteration — latency-bound, not bandwidth-bound. Tensor parallelism is limited to ~8 GPUs within a single node (NVLink has low latency; InfiniBand does not).

## Pipeline Parallelism

Split layers across GPUs. GPU 0 does layers 0–23, GPU 1 does 24–47, etc. The batch is split into micro-batches that flow through the pipeline:

```
Time →
GPU 0: [F0][F1][F2][B0][B1][B2]
GPU 1:    [F0][F1][F2][B0][B1][B2]
GPU 2:       [F0][F1][F2][B0][B1][B2]
```

The "bubble" (idle time at the start/end of the pipeline) is the main inefficiency. With M micro-batches and P pipeline stages, the bubble fraction is (P-1)/(P+M-1). For P=8, M=64: bubble = 7/71 ≈ 10%. Acceptable.

GPipe (Google) uses synchronous pipeline parallelism. PipeDream (Microsoft) uses asynchronous (1F1B scheduling — one forward, one backward, interleaved) for better overlap but stale gradients.

## 3D Parallelism (Data + Tensor + Pipeline)

Production training combines all three:

```
World size = D (data) × T (tensor) × P (pipeline)

Example: GPT-3 training on 1024 A100 GPUs:
  D = 64 (data parallel groups of 16 GPUs each)
  T = 8 (tensor parallel within each node)
  P = 8 (pipeline parallel across nodes)
  Total: 64 × 8 × 8 = 4096 GPUs (actual: 1024 with some groups smaller)
```

Communication hierarchy:
1. Tensor parallel all-reduces: within NVLink domain (fastest).
2. Pipeline parallel sends: within InfiniBand island.
3. Data parallel all-reduces: across all GPUs (slowest, but lowest frequency).

## Gradient Compression

For bandwidth-limited clusters, compress gradients before all-reduce:

- **Top-k sparsification**: Send only the k largest gradient components. Error feedback compensates for the dropped components.
- **QSGD (Quantized SGD)**: Quantize gradients to 8-bit or 4-bit. SignSGD: only send the sign of each gradient (+1/-1).
- **PowerSGD**: Low-rank approximation of the gradient matrix.

QSGD reduces communication by 4× (32-bit → 8-bit) with <0.5% accuracy loss on ImageNet. For GPT-scale models, gradient compression is critical — all-reducing 175B gradients in FP32 is 700 GB, taking ~1.2 seconds on InfiniBand. With 8-bit quantization: 175 GB, ~0.3 seconds. Over millions of iterations, this saves days of training time.

## When to Use What

| Model size | GPUs | Strategy |
|-----------|------|----------|
| < 1B params | 1–8 | Data parallelism (DDP) |
| 1B–10B | 8–64 | DDP + gradient checkpointing |
| 10B–100B | 64–512 | ZeRO-3 + tensor parallel (within node) |
| 100B–1T | 512–4096 | 3D (data + tensor + pipeline) + ZeRO |
| > 1T | 4096+ | 3D + ZeRO + gradient compression + expert parallelism (MoE) |

The trend: as models grow, parallelism strategy shifts from pure data parallelism to hybrid schemes. The communication topology determines the scaling limit — NVLink bandwidth (within node) is 10× InfiniBand bandwidth (across nodes), so tensor parallelism is best within nodes and pipeline/data parallelism across nodes.
