# Learning with Neural Networks

Neural networks are not magic. They are compositions of differentiable functions, trained by gradient descent. The "deep" in deep learning just means "many layers." What makes them work — and what makes them computationally demanding — is that each layer is a matrix multiplication followed by a nonlinearity, and the gradient must flow backward through all layers.

This chapter assumes you've read the linear algebra, optimization, and statistical computation chapters. It focuses on the computational engineering that makes neural networks trainable: backpropagation, automatic differentiation, and the hardware tricks (mixed precision, kernel fusion, flash attention) that reduce training from weeks to hours.

The centerpiece is a step-by-step explanation of backpropagation — the algorithm that computes gradients through arbitrary computation graphs. It is simpler than most textbooks make it seem.

## What This Chapter Covers

1. **Backpropagation Explained Simply** — The chain rule applied to computation graphs. Forward pass stores intermediate values. Backward pass multiplies local gradients. Why it's O(f + b) where f is the forward cost. A worked example from scratch.
2. **Neural Network Primitives** — Fully-connected, convolutional, attention, normalization layers. Each as a forward/backward pair. Their computational bottlenecks (matmul, im2col, softmax).
3. **Automatic Differentiation** — Wengert lists, operator overloading, and source transformation. Why PyTorch builds a DAG. How `backward()` traverses it in reverse topological order.
4. **Hardware-Aware Training** — Mixed precision (FP16/BF16), gradient accumulation, kernel fusion, flash attention. How a single A100 achieves 312 TFLOPS for training — and why naive PyTorch achieves 30 TFLOPS.
5. **Distributed Training** — Data parallelism, model parallelism, pipeline parallelism. Ring all-reduce. ZeRO optimizer. How GPT-4 was trained on 25,000 GPUs.

## Recommended Reading Order

Read **Backpropagation Explained Simply** first — it's the foundation for everything else. The primitives article connects backprop to the matrix multiplication case study (Chapter 11). Autodiff explains how frameworks like PyTorch implement backprop automatically. Hardware-aware training and distributed training are advanced topics for production-scale models.

Cross-reference with Chapter 11 (Matrix Multiplication) for the core kernel, Chapter 17 (Global Optimizations) for the optimizers, and Chapter 13 (Parallel Computing) for the GPU/CUDA foundations.
