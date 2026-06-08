# Hardware-Aware Training

The difference between 30 TFLOPS (naive PyTorch) and 312 TFLOPS (well-tuned) on an NVIDIA A100 is not the hardware — it's the software stack. This article covers the techniques that squeeze maximum performance from training hardware: mixed precision, kernel fusion, memory-efficient attention, and overlapping communication with computation.

## The Roofline Model for Training

The A100 has 312 TFLOPS (FP16 tensor core) and 2 TB/s HBM bandwidth. The arithmetic intensity (FLOPs per byte loaded) determines whether you're compute-bound or memory-bound:

| Operation | Arithmetic intensity | Bottleneck |
|-----------|---------------------|------------|
| Elementwise (ReLU, dropout) | ~1 FLOP/2 bytes = 0.5 | Memory-bound |
| Linear (matmul, n≥1024) | O(n):1 | Compute-bound |
| Attention (n=512) | ~128 FLOP/byte | Compute-bound |
| Layer norm | ~5 FLOP/byte | Memory-bound |
| Softmax | ~3 FLOP/byte | Memory-bound |

The compute-bound operations (matmul, attention) achieve 70–90% of peak TFLOPS. The memory-bound operations achieve 5–20%. The goal of hardware-aware training is to fuse memory-bound operations into compute-bound ones, or eliminate them entirely.

## Mixed Precision Training (FP16/BF16)

Train in half-precision (FP16 or BF16) but keep a master copy of weights in FP32. This doubles throughput (FP16 tensor cores are 2× faster than FP32) and halves memory:

```rust
// Pseudocode for mixed-precision training loop
fn training_step(model: &Model, optimizer: &mut Adam,
                 batch: &Batch, scaler: &mut GradScaler) {
    // Forward in FP16
    let (loss, activations) = model.forward_fp16(batch);

    // Scale loss to prevent underflow
    let scaled_loss = loss * scaler.scale_factor;

    // Backward in FP16 (gradients also in FP16)
    scaled_loss.backward();

    // Unscale gradients, check for inf/nan
    scaler.unscale_gradients(model);

    // Update FP32 master weights using FP16 gradients
    optimizer.step_fp16(model);

    // Update scaler factor
    scaler.update();
}
```

The `GradScaler` multiplies the loss by a large factor (e.g., 2¹⁶) before backward to prevent small gradients from flushing to zero in FP16 (minimum positive FP16 is ~6×10⁻⁸). After backward, it unscales. If any gradient is inf/nan, the step is skipped and the scale factor is halved.

BF16 (bfloat16: 8 exponent bits, 7 mantissa bits) has the same dynamic range as FP32 — no scaling needed. It's supported on A100 and later, and is the default for Google TPUs.

## Flash Attention

Standard attention stores the n×n attention matrix Q·Kᵀ — O(n²) memory. Flash Attention tiles the computation: compute softmax incrementally within SRAM, never materializing the full attention matrix.

```
Standard:
  S = Q @ Kᵀ              # n×n matrix in HBM
  P = softmax(S)          # n×n matrix in HBM
  O = P @ V               # n×d matrix

Flash Attention:
  For each block of Q (size B_r × d):
    For each block of K, V (size B_c × d):
      S_block = Q_block @ K_blockᵀ       # in SRAM, B_r × B_c
      P_block = online_softmax(S_block)  # in SRAM
      O_block += P_block @ V_block       # accumulate in SRAM
```

Online softmax: maintain running max and sum, rescale previous results when a new max is found. This is numerically identical to standard softmax (not approximate).

Flash Attention achieves:
- O(n) memory instead of O(n²).
- 2–4× wall-clock speedup (fewer HBM reads/writes, despite more total FLOPs).
- Enables training with sequence length 8192+ (GPT-4 uses 8K–32K context).

## Kernel Fusion

PyTorch's eager execution launches a separate GPU kernel for each operation. Kernel fusion combines multiple operations into a single kernel:

```
// Instead of:
//   kernel_1: y = x + residual    (read x, residual → write y)
//   kernel_2: z = layer_norm(y)   (read y → compute mean/var → write z)
//
// Fused:
//   kernel_fused: z = layer_norm(x + residual)
//   (read x, residual once → all compute in registers/SRAM → write z once)
```

Fusion eliminates HBM round-trips for intermediate results. For memory-bound operations (ReLU, dropout, residual adds), fusion gives 20–50% speedup. For compute-bound operations (matmul), fusion has marginal benefit because HBM traffic is already minimal.

`torch.compile` (PyTorch 2.0+) does automatic kernel fusion using the Inductor compiler. Also helpful: `Apex` (NVIDIA), `xFormers` (Meta), and `FlashAttention` (Dao et al.).

## Overlapping Communication and Computation

In distributed training (next article), gradient all-reduce happens after backward. While GPU 0 reduces gradients, it could be computing the next forward pass. Overlapping is achieved with **gradient bucketing**: accumulate gradients from several layers into a buffer, then launch the all-reduce while computing gradients for the next layers:

```python
# PyTorch Distributed with gradient bucketing
model = torch.nn.parallel.DistributedDataParallel(
    model,
    bucket_cap_mb=25,  # all-reduce buckets of 25 MB
    gradient_as_bucket_view=True,  # avoid copies
)
```

The DDP wrapper registers backward hooks: after each parameter's gradient is computed, it's added to a bucket. When the bucket is full (or at the end of backward), the all-reduce is launched asynchronously. The next forward pass overlaps with the in-progress all-reduce. Effective overlap: 50–80%, meaning 50–80% of communication time is hidden behind computation.

## Gradient Accumulation

When the batch size doesn't fit in GPU memory, accumulate gradients over multiple micro-batches before updating weights:

```rust
fn train_with_accumulation(model: &mut Model, data: &DataLoader,
                           micro_batch_size: usize, accum_steps: usize,
                           optimizer: &mut Adam) {
    optimizer.zero_grad();
    let mut step = 0;

    for batch in data.iter_batches(micro_batch_size) {
        let loss = model.forward(&batch);
        (loss / accum_steps as f32).backward(); // scale loss

        step += 1;
        if step % accum_steps == 0 {
            optimizer.step();
            optimizer.zero_grad();
        }
    }
}
```

This gives the same mathematical result as a larger batch (for batch norm statistics, use `SyncBatchNorm` across micro-batches). The cost: `accum_steps` forward-backward passes instead of one. The benefit: training with effective batch size 4096 on a single GPU that only fits 64 samples.

## Putting It All Together: A100 Training Throughput

Training a 7B-parameter Llama-style model with sequence length 2048:

| Optimizations | TFLOPS (effective) | % of peak | Tokens/second |
|--------------|-------------------|-----------|---------------|
| Naive FP32 PyTorch, no fusion | 28 | 9% | 1,200 |
| + FP16 mixed precision | 56 | 18% | 2,400 |
| + Flash Attention | 72 | 23% | 3,100 |
| + Kernel fusion (torch.compile) | 85 | 27% | 3,700 |
| + Gradient accumulation (bs=4M) | 90 | 29% | 3,900 |
| + ZeRO-3 (model sharding) | 2800 (8 GPUs) | 28% | 32,000 |
| + 8× A100 (tensor parallel + pipeline) | 11,200 (32 GPUs) | 28% | 127,000 |

The jump from 9% to 29% of peak is entirely software optimization — no hardware changes. The jump to multi-GPU adds communication overhead (gradient all-reduce, parameter broadcast) that limits further scaling. At 32 GPUs, 28% of peak is excellent — this is the regime where GPT-4-level models are trained.

The lesson: hardware peak TFLOPS is a marketing number. Achieved TFLOPS is a software engineering problem.
