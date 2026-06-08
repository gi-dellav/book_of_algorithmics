# Learning Without Neural Networks

Before deep learning ate the world, machine learning meant linear models, trees, kernels, and ensembles. These methods remain superior for tabular data, small datasets, and problems where interpretability matters. They're also the foundation — understanding why logistic regression works prepares you to understand why a neural network works.

This chapter covers the core non-neural ML algorithms with a focus on computational efficiency. Each method is presented with its computational bottleneck identified and optimized using the techniques from earlier chapters.

## What This Chapter Covers

1. **Linear Models** — Logistic regression, SVM, and linear regression as optimization problems. The computational difference between solving the primal and the dual. When to use SGD, L-BFGS, or coordinate descent.
2. **Tree-Based Methods** — Decision trees, random forests, and gradient boosting (XGBoost, LightGBM). Histogram-based splitting, GPU-accelerated tree building, and why XGBoost is 100× faster than scikit-learn's GBM.
3. **Kernel Methods** — The kernel trick, the Gram matrix, and the Nyström approximation. Why kernel SVMs are O(n²) and how to make them O(n).
4. **Nearest Neighbors and Clustering** — k-d trees, ball trees, and approximate nearest neighbors (HNSW, IVF-PQ). k-means with SIMD distance computations. DBSCAN with spatial indexing.

## Recommended Reading Order

Read **Linear Models** first — the optimization perspective connects directly to the Global Optimization chapter. Then trees (the most practical method for tabular data), kernel methods (for when you need theoretical guarantees), and nearest neighbors (for when you need no training at all).

Cross-reference with Chapter 15 (Linear Algebra) for the matrix operations, Chapter 16 (Statistical Computation) for the probabilistic foundations, and Chapter 17 (Global Optimizations) for the optimizers used throughout.
