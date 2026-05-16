# README-LOWLEVEL.md

> **VectorForge Internal Architecture & Algorithmic Foundations**
>
> This document is the technical goldmine of **VectorForge** — the mathematics,
> algorithms, heuristics, and engineering tradeoffs that power its semantic
> vector search engine.

---

# Table of Contents

1. Introduction
2. Embeddings: Semantics to Vectors
3. Similarity Metrics
4. Indexing Families and Tradeoffs
5. HNSW Deep Dive
6. Chunking and Document Retrieval
7. Search Pipeline
8. Evaluation Metrics
9. Complexity Cheat Sheet
10. Math Appendix
11. Practical Tuning
12. Future Work
13. Code Pointers

---

# 1. Introduction

VectorForge is a compact but production-style semantic search engine implemented
in C++ with a browser-based interface.

It demonstrates how modern vector databases are built from first principles
using:

- Dense embeddings
- Cosine similarity
- Approximate Nearest Neighbor (ANN) search
- Hierarchical Navigable Small World (HNSW) graphs
- Embedded HTTP APIs
- Interactive visualization

---

# 2. Embeddings: Semantics to Vectors

A text document is transformed into a vector:

$$
\mathbf{x} = [x_1, x_2, \ldots, x_d] \in \mathbb{R}^d
$$

Where:

- $d$ is the embedding dimension.
- $x_i$ is the feature value along dimension $i$.

Semantically similar texts occupy nearby locations in this high-dimensional
space.

---

# 3. Similarity Metrics

## Dot Product

$$
\mathbf{x}\cdot\mathbf{y} = \sum_{i=1}^{d} x_i y_i
$$

## Euclidean Distance

$$
\|\mathbf{x}-\mathbf{y}\|_2 = \sqrt{\sum_{i=1}^{d}(x_i-y_i)^2}
$$

## Cosine Similarity

$$
\cos(\theta) = \frac{\mathbf{x}\cdot\mathbf{y}}{\|\mathbf{x}\|_2\|\mathbf{y}\|_2}
$$

## Normalization

$$
\hat{\mathbf{x}} = \frac{\mathbf{x}}{\|\mathbf{x}\|_2}
$$

For normalized vectors:

$$
\hat{\mathbf{x}}\cdot\hat{\mathbf{y}} = \cos(\theta)
$$

---

# 4. Indexing Families and Tradeoffs

## Brute Force

- Query complexity: $O(n \cdot d)$
- Exact results
- Best for small datasets and validation

## KD-Tree

- Works well for low-dimensional data
- Average query complexity: $O(\log n)$
- Suffers from the curse of dimensionality

## HNSW

- State-of-the-art approximate nearest neighbor algorithm
- High recall with sublinear search time
- Excellent scalability to millions of vectors

---

# 5. HNSW Deep Dive

HNSW (Hierarchical Navigable Small World) organizes vectors into multiple graph
layers.

Higher layers are sparse and enable long-range jumps.

Lower layers are dense and refine local neighborhoods.

## Layer Assignment

$$
L = \left\lfloor -\ln(U)m_L \right\rfloor
$$

Where:

- $U \sim \mathrm{Uniform}(0,1)$
- $m_L$ is the level multiplier

Probability of reaching level $l$:

$$
P(L \ge l) = e^{-l/m_L}
$$

## Key Parameters

| Parameter | Meaning |
|---------:|---------|
| `M` | Maximum neighbors per node |
| `efConstruction` | Candidate pool during insertion |
| `ef` | Candidate pool during search |

## Sparse-to-Dense Routing

```text
                 ┌──────────── Layer 4 ────────────┐
                 │        Global Entry Node        │
                 └─────────────────┬───────────────┘
                                   │
                                   ▼
             ┌──────── Layer 3 (Very Sparse) ────────┐
             │      Long-distance navigation         │
             └─────────────────┬─────────────────────┘
                               │
                               ▼
             ┌──────── Layer 2 (Sparse) ─────────────┐
             │       Regional routing stage          │
             └─────────────────┬─────────────────────┘
                               │
                               ▼
             ┌────── Layer 1 (Refinement) ───────────┐
             │      Neighborhood exploration         │
             └─────────────────┬─────────────────────┘
                               │
                               ▼
┌──────────────────── Layer 0 (Dense Full Graph) ────────────────────┐
│ All vectors present; best-first search computes final top-k result. │
└─────────────────────────────────────────────────────────────────────┘

Path:
Entry → Coarse Jump → Regional Routing → Local Refinement → Top-k
```

---

# 6. Chunking and Document Retrieval

Large documents are split into smaller chunks.

Each chunk receives an embedding:

$$
\mathbf{x}_{\text{chunk}}
$$

Metadata stored with each chunk includes:

- Document ID
- Chunk index
- Character or token offsets

This enables fine-grained retrieval for RAG systems.

---

# 7. Search Pipeline

1. Convert query text into embedding $\mathbf{q}$.
2. Search HNSW for top-$k$ nearest chunk vectors.
3. Optionally re-rank candidates.
4. Return relevant chunks to the user or language model.

---

# 8. Evaluation Metrics

## Recall@k

Fraction of relevant items retrieved in top-$k$.

## MRR

Mean Reciprocal Rank of the first relevant result.

## Latency

P95 and P99 response times.

## Throughput

Queries processed per second.

---

# 9. Complexity Cheat Sheet

## Build Complexity

- Brute Force: $O(1)$ append
- KD-Tree: $O(n \log n)$
- HNSW: approximately $O(n \cdot efConstruction \cdot \log n)$

## Query Complexity

- Brute Force: $O(n \cdot d)$
- KD-Tree: $O(\log n)$ for low dimensions
- HNSW: approximately $O(ef \cdot \log n)$

## Memory Complexity

$$
O(n(d + M))
$$

---

# 10. Math Appendix

For normalized vectors:

$$
\|\hat{\mathbf{x}}-\hat{\mathbf{y}}\|_2^2 = 2 - 2\cos(\theta)
$$

Thus, sorting by cosine similarity is equivalent to sorting by Euclidean
distance when vectors are normalized.

---

# 11. Practical Tuning

- Increase `efConstruction` to improve index quality.
- Increase `M` to improve graph connectivity.
- Increase query-time `ef` to improve recall.
- Normalize embeddings to simplify metric calculations.

---

# 12. Future Work

## Performance and Scale

- Product Quantization (PQ)
- Optimized PQ (OPQ)
- FP16 and INT8 embeddings
- GPU acceleration

## Ranking

- Cross-encoder re-ranking
- Learning-to-rank

## Reliability

- Persistent serialization
- Incremental updates
- Sharding and replication

---

# 13. Code Pointers

| File | Responsibility |
|------|----------------|
| `main.cpp` | Vector math, HNSW indexing, REST endpoints |
| `index.html` | Browser UI and API integration |
| `httplib.h` | Embedded HTTP server implementation |

---

# Final Thoughts

VectorForge demonstrates how a few elegant mathematical ideas—embeddings,
cosine similarity, and graph navigation—combine to form a powerful semantic
search engine.

This project is both:

- A production-style engineering system
- A hands-on educational exploration of vector databases
- A strong showcase project for systems, search, and AI engineering