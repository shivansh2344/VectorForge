# VectorForge Low-Level Architecture

This document explains the math, indexing tradeoffs, and retrieval pipeline behind VectorForge. It is the technical companion to [README.md](README.md) and focuses on how the system works under the hood.

## What this covers

- Embeddings and similarity metrics
- Brute force, KD-tree, and HNSW indexing
- Chunking strategy for document retrieval
- Search and ranking flow
- Evaluation, tuning, and future work

## Embeddings and similarity

Text is mapped into a vector space so that similar meanings end up near each other.

If $\mathbf{x}$ is an embedding in $\mathbb{R}^d$, common similarity measures are:

- Dot product: $\mathbf{x}\cdot\mathbf{y} = \sum_i x_i y_i$
- Euclidean distance: $\|\mathbf{x}-\mathbf{y}\|_2$
- Cosine similarity: $\cos(\theta)=\frac{\mathbf{x}\cdot\mathbf{y}}{\|\mathbf{x}\|_2\|\mathbf{y}\|_2}$

Normalization is often useful:

$$
\hat{\mathbf{x}} = \frac{\mathbf{x}}{\|\mathbf{x}\|_2}
$$

For unit vectors, dot product and cosine similarity are equivalent, which simplifies retrieval logic.

## Indexing families

| Index | Strengths | Tradeoffs |
|---|---|---|
| Brute force | Exact results, simple implementation | $O(n\cdot d)$ per query |
| KD-tree | Good for low-dimensional data | Degrades in high dimensions |
| HNSW | Fast approximate nearest neighbors with high recall | More complex and memory-heavy |

### HNSW in practice

HNSW organizes vectors into multiple layers. Higher layers are sparse and help with long-range navigation; lower layers are denser and refine the final answer.

Key parameters:

- `M`: maximum neighbors per node
- `efConstruction`: candidate pool during insertion
- `ef`: candidate pool during query time

The usual search pattern is:

1. Start from the top layer entry point.
2. Greedily move toward promising neighbors.
3. Descend layer by layer.
4. Use the bottom layer for final top-k selection.

## Chunking and retrieval

Long documents are split into smaller chunks so one embedding does not mix unrelated ideas.

Each chunk stores:

- Document ID
- Chunk index
- Character or token offsets

This lets VectorForge retrieve precise passages instead of returning an entire document.

## Search pipeline

1. Convert the query into an embedding.
2. Search the index for the nearest chunk vectors.
3. Optionally re-rank the candidates.
4. Return the best passages to the user or language model.

## Evaluation metrics

- Recall@k: how many relevant items appear in the top-k list
- MRR: how early the first relevant item appears
- Latency: especially P95 and P99 response times
- Throughput: queries processed per second

## Complexity cheat sheet

| Operation | Brute force | KD-tree | HNSW |
|---|---:|---:|---:|
| Build | $O(1)$ insert | $O(n\log n)$ | Approximately $O(n\cdot efConstruction\cdot\log n)$ |
| Query | $O(n\cdot d)$ | Roughly $O(\log n)$ in low dimensions | Approximately $O(ef\cdot\log n)$ |
| Memory | Low | Medium | Higher |

## Math appendix

For unit vectors, cosine similarity and Euclidean distance are tightly related:

$$
\|\hat{\mathbf{x}}-\hat{\mathbf{y}}\|_2^2 = 2 - 2\cos(\theta)
$$

That is why normalized embeddings let you use simpler scoring logic without changing the ranking.

## Practical tuning

- Increase `efConstruction` to improve index quality.
- Increase `M` to improve connectivity and recall.
- Increase query-time `ef` to improve recall at the cost of latency.
- Normalize embeddings when you want cosine and dot product to behave consistently.

## Future work

- Product quantization and optimized product quantization
- FP16 or INT8 embeddings
- Cross-encoder re-ranking
- Persistent serialization and incremental updates
- Sharding and replication for larger workloads

## Code pointers

| File | Responsibility |
|---|---|
| `main.cpp` | Vector math, HNSW indexing, REST endpoints |
| `index.html` | Browser UI and API integration |
| `httplib.h` | Embedded HTTP server implementation |

## References

- Malkov, Yu. A., and Yashunin, D. (2018). Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs.
- Johnson, J., Douze, M., and Jegou, H. (2019). Billion-scale similarity search with GPUs.
- FAISS, Hnswlib, and Annoy documentation.

