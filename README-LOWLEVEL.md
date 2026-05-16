# README — Low Level (VectorForge)

This document is the technical goldmine: the math, the tradeoffs, the heuristics, and the "why" behind the implementation in `main.cpp` and the indexing components that make VectorForge work. Read this if you want to understand vector databases from first principles and be inspired to extend the system.

## Why you should care?
- If you build search or retrieval systems, embedding vectors are the lingua franca for meaning. This repo is a compact playground showing how raw math + simple engineering produce practical semantic search.
- Think of VectorForge like a small ICPC problem: clever algorithms, sharp tradeoffs, and tight code. If you enjoy the satisfying click when math meets speed, you'll love the deep-dive below.

## 1. Embeddings: semantics → vectors
- Intuition: embeddings map text into a high-dimensional vector space so that semantically similar texts map to nearby vectors.
- Vector $\mathbf{x}\in\mathbb{R}^d$ represents a piece of text. Similarity is measured by geometric closeness.

Distance and similarity measures
- Euclidean distance (L2):
  $$\mathrm{dist}_2(\mathbf{x},\mathbf{y}) = \|\mathbf{x}-\mathbf{y}\|_2 = \sqrt{\sum_{i=1}^d (x_i-y_i)^2}$$
- Cosine similarity (commonly used for embeddings):
  $$\cos(\theta)=\frac{\mathbf{x}\cdot\mathbf{y}}{\|\mathbf{x}\|\,\|\mathbf{y}\|}$$
  For retrieval we often use cosine distance: $1-\cos(\theta)$, or equivalently compute cosine similarity and sort descending.
- Dot product: $\mathbf{x}\cdot\mathbf{y}=\sum_i x_i y_i$ is used by some models directly; for fixed-length vectors and normalized embeddings, dot product and cosine are proportional.

Normalization
- Many systems normalize embeddings to unit length ($\|\mathbf{x}\|_2=1$) so that cosine similarity = dot product. This simplifies math and index design.
- Normalization: $\hat{\mathbf{x}} = \mathbf{x}/\|\mathbf{x}\|_2$.

## 2. Vector indexing families and tradeoffs
- BruteForce (linear scan)
  - Complexity: $O(n\cdot d)$ per query. Exact; trivially implemented.
  - Use when $n$ is small (hundreds to low thousands) or for correctness tests.
- KD-Tree
  - Partitioning by axis-aligned hyperplanes. Works well for low dimensions (typically $d\le 20$).
  - Complexity: average $O(\log n)$ for point queries, but degrades with dimension (curse of dimensionality).
  - Strategy: split on the dimension with largest variance; build recursively until leaf size small.
- HNSW (Hierarchical Navigable Small World)
  - State-of-the-art tradeoff: very fast approximate nearest neighbors with high recall.
  - Key ideas: navigable small-world graphs + hierarchical levels for fast search.
  - Construction hyperparameters:
    - $M$ — max neighbor count per node on each layer (controls graph degree and memory).
    - $efConstruction$ — size of the candidate pool during insertion (controls index quality/construct time).
  - Search hyperparameters:
    - $ef$ — size of the dynamic candidate list at query time (tradeoff: recall vs latency).

### HNSW: how it works (brief math)
- The HNSW index maintains multiple layers $L_0, L_1, \ldots, L_m$ with decreasing vertex sets: higher layers are sparser.
- Each node has neighbors selected by a heuristic (typically "select the best neighbors by distance with diversification").
- Search: start from an entry point at top layer, greedily walk to nearest neighbors until local minima, then descend layers refining the search with an $ef$-sized candidate buffer and a priority queue.

### Why approximate is fine
- Exact nearest neighbors often unnecessary — in retrieval we care that a good set of candidates is returned for re-ranking.
- HNSW produces near-perfect recall for practical $ef$ and $M$ values while being orders of magnitude faster than brute force at scale.

## 3. Vector aggregation and chunking for documents
- Documents are split into chunks (heuristics: sentence boundaries, fixed character length, or token count) so long documents don't collapse semantics into a single vector.
- For each chunk, we compute an embedding $\mathbf{x}_{chunk}$.
- Document-level retrieval: store each chunk in the index and attach metadata (doc id, chunk offset). Retrieval returns chunks which are then reassembled or re-ranked.

### Chunk embedding strategies
- Per-chunk embedding: best practice — embed each chunk independently.
- Average pooling: sometimes systems average token embeddings or chunk vectors to produce a document vector; this reduces recall for fine-grained retrieval and is not recommended for RAG scenarios where chunk-level relevance matters.

## 4. Similarity search pipeline in VectorForge
- Query flow:
  1. Convert query text $q$ to embedding $\mathbf{q}$ using the embedding model.
  2. Search the HNSW (or chosen index) for top-$k$ candidate chunk vectors by cosine or dot product similarity.
  3. Optionally re-rank the candidates using a stronger scoring function (cross-encoder or semantic similarity score) — not implemented here but discussed in Future Work.
  4. Construct a prompt with retrieved chunks and send to the local generator for a concise answer.

## 5. Evaluation metrics and monitoring
- Recall@k: fraction of true relevant items present in the top-$k$ results.
- MRR (Mean Reciprocal Rank): measures how far down the list the first relevant item appears.
- Latency P95/P99: for production, track tail latency.
- Throughput (queries/sec): consider batching and concurrency.

## 6. Numeric stability and precision
- Many embeddings are 32-bit floats (single precision). For large indexes, memory dominates — consider quantization (see Future Work).
- Avoid subtractive cancellation by normalizing early if your pipeline mixes metrics.

## 7. Implementation notes and heuristics used in this repo
- When building the KD-tree: choose split dimension by variance and use median split for balanced trees.
- HNSW neighbor selection: use the standard heuristic that picks neighbors by lowest distance and attempts to diversify by rejecting neighbors that are too close to already chosen neighbors.
- Use a fixed seed for reproducibility in demos (so HNSW level assignment and RNG are deterministic).

## 8. Complexity cheat-sheet
- Build:
  - BruteForce: $O(1)$ insert (append to array)
  - KD-Tree: $O(n\log n)$ to build balanced tree
  - HNSW: roughly $O(n\cdot efConstruction \cdot \log n)$ depending on implementation details
- Query:
  - BruteForce: $O(n\cdot d)$
  - KD-Tree: expected $O(\log n)$ for low $d$, worse for high $d$
  - HNSW: $O(ef\cdot \log n)$ for candidate expansion; empirically sublinear in practice

## 9. Math appendix (small proofs and formulae)
- Relationship between normalized dot product and cosine similarity:
  If $\hat{\mathbf{x}}$ and $\hat{\mathbf{y}}$ are unit vectors, then:
  $$\hat{\mathbf{x}}\cdot\hat{\mathbf{y}} = \cos(\theta)$$
- Converting cosine to Euclidean distance for normalized vectors:
  $$\|\hat{\mathbf{x}}-\hat{\mathbf{y}}\|_2^2 = 2 - 2\cos(\theta)$$
  So, sorting by cosine similarity is equivalent to sorting by Euclidean distance when vectors are normalized.

## 10. Practical tradeoffs and tuning knobs
- Increase $efConstruction$ for higher index quality at the cost of build time and memory.
- Increase $M$ to increase connectivity (memory up), query latency down (usually) and recall up.
- Adjust query-time $ef$ to trade speed vs recall.
- Metric choice: cosine vs dot product depends on embedding normalization and model specifics.

## 11. Future work (ideas to extend this project)
- Productive performance and scale
  - Add on-disk index formats (IVF + PQ) for massive datasets.
  - Integrate with FAISS, Hnswlib, or Annoy for optimized BLAS/GPU-backed search.
- Compression & efficiency
  - Product Quantization (PQ) and Optimized PQ (OPQ) to reduce memory by 4x–8x with small recall drop.
  - Use 8-bit/fp16 quantized embeddings if the model supports it.
- Ranking & quality
  - Add a cross-encoder re-ranker (heavy but accurate) to re-score top candidates.
  - Add learning-to-rank pipelines with relevance feedback.
- Reliability and operations
  - Persistent index serialization and safe checkpoints for incremental updates.
  - Sharding and replication for throughput and availability.
- Features & tooling
  - Active learning loop to improve embeddings for a domain.
  - Build an evaluation harness (datasets, scripts) for offline benchmarking (recall@k, latency, memory).

## 12. Code pointers (where to look in this repo)
- `main.cpp` — index implementations, `DocumentDB`, `ServiceClient` and REST endpoints.
- `index.html` — UI and example usage patterns.
- `httplib.h` — lightweight embedded HTTP implementation used for the REST layer.

## 13. Parting words 
- The joy of building retrieval systems is the repeated refinement: a tiny tweak to $M$ or $ef$ and you suddenly have a faster, more accurate search. Build experiments in small iterations, measure recall and latency, and keep the user in the loop.
- If this doc got you excited: star and fork the repo, add a fast re-ranker, or experiment with PQ — share your findings with me on LinkedIn - https://www.linkedin.com/in/shivansh2344/ 

### References and further reading
- Malkov, Yu. A., & Yashunin, D. (2018). Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs (HNSW).
- Johnson, J., Douze, M., & J egou, H. (2019). Billion-scale similarity search with GPUs.
- FAISS, Hnswlib, Annoy documentation and source code.

