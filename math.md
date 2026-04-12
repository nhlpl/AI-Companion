## 🧠 Mathematics Used in the AI Companion Project

The AI Companion integrates a wide range of mathematical concepts from linear algebra, probability, optimization, information theory, and algorithmic complexity. Below is a comprehensive list, organized by component.

---

### 1. Vector Spaces & Embeddings

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Euclidean vector space** | \(\mathbb{R}^{128}\) | Embedding space for memories and queries. |
| **Cosine similarity** | \(\text{sim}(x,y) = \frac{x \cdot y}{\|x\|\|y\|}\) | Retrieval relevance scoring. |
| **Euclidean distance** | \(d(x,y) = \sqrt{\sum (x_i - y_i)^2}\) | Used in k‑means clustering for PQ codebooks. |
| **L2 normalization** | \(\hat{x} = x / \|x\|\) | Normalizes embeddings before similarity computation. |
| **Principal Component Analysis (PCA)** | \(\text{Proj}(x) = W^T (x - \mu)\), \(W\) from eigen decomposition of covariance matrix | Reduces embedding dimension (384 → 128) and 2D personality map. |
| **Random projection** | \(h(x) = \text{sign}(R x)\) where \(R \in \mathbb{R}^{4 \times 128}\) is random | LSH hashing for fast approximate retrieval. |

---

### 2. Locality‑Sensitive Hashing (LSH)

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Random hyperplane hashing** | \(h(x) = \text{sign}(w \cdot x)\), bits concatenated to form bucket index | Maps vectors to buckets; similar vectors collide with high probability. |
| **Hamming distance** | \(d_H(b_1, b_2) = \sum \text{XOR}(b_1[i], b_2[i])\) | Approximate cosine similarity via binary codes. |
| **Bucket sampling** | Unbiased random sample when bucket size > limit | Controls retrieval latency while maintaining fairness. |
| **Independence of projections** | \(P(h(x) = h(y)) = 1 - \frac{1}{\pi} \cos^{-1}(\text{sim}(x,y))\) | Probability of collision given cosine similarity. |

---

### 3. Product Quantization (PQ)

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Subvector decomposition** | \(x = (x^{(1)}, x^{(2)}, \dots, x^{(M)})\), each \(x^{(j)} \in \mathbb{R}^{d/M}\) | Splits embedding into \(M\) subvectors. |
| **k‑means clustering** | \(\min_{c_j} \sum_{i} \|x_i - c_{k(i)}\|^2\) | Learns codebooks (256 centroids per subvector). |
| **Centroid index** | \(q_j = \arg\min_{k} \|x^{(j)} - c_{jk}\|\) | Encodes each subvector as 1 byte. |
| **Symmetric Distance Computation (SDC)** | \(d(x,y) \approx \sum_{j=1}^M \|c_{j q_j(x)} - c_{j q_j(y)}\|^2\) | Avoids decoding; precomputed distance tables. |
| **Quantization error** | \(E = \mathbb{E}[\|x - \hat{x}\|^2]\) | Trade‑off between accuracy and compression ratio. |

---

### 4. Importance Decay & Memory Pruning

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Exponential decay** | \(I(t) = I_0 e^{-t / \tau}\), \(\tau = 1 \text{ day}\) | Memory importance half‑life. |
| **Stretched exponential** | \(I(t) = I_0 \exp(-(\lambda t)^\beta)\), \(\beta = 0.5\) | Anti‑memory strength decay (matches human forgetting). |
| **Power law (alternative)** | \(I(t) = I_0 / (1 + t/\tau)^{0.5}\) | Previously considered; slower than exponential. |
| **Pruning threshold** | Delete if \(I < 0.1\) and not permanent | Removes low‑value memories. |
| **Incremental pruning** | Delete fraction \(p = 0.05\) of low‑importance memories per cycle | Smooth I/O, avoids latency spikes. |

---

### 5. Personality & Learning

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **4‑dimensional personality** | \(p \in [0,1]^4\) | Formality, creativity, empathy, verbosity. |
| **Hex encoding** | \(h = \sum_{i=0}^3 \lfloor 15 p_i \rfloor \cdot 16^{3-i}\) | Compact storage (4 hex digits). |
| **Stochastic Weight Averaging (SWA)** | \(\bar{p}_t = \beta \bar{p}_{t-1} + (1-\beta) p_t\), \(\beta = e^{-\Delta t / \tau}\) | Smooths personality updates over 7 days. |
| **Weight decay** | \(p_{t+1} = (1 - \eta \lambda) p_t + \eta \lambda p_{\text{neutral}}\) | Prevents permanent extreme values. |
| **Gradient ascent (simplified)** | \(p \leftarrow p + \eta \nabla R(p)\) where \(R\) is user reward | Online personality adaptation. |
| **Learning rate schedule** | \(\eta_t = \eta_0 / (1 + t/T_0)\) | Decaying learning rate for stability. |

---

### 6. Anti‑Memory & Thompson Sampling

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Thompson sampling** | Sample \(\theta \sim \text{Beta}(\alpha, \beta)\); keep if \(\theta > 0.5\) | Decides which anti‑memories to include. |
| **Beta distribution** | \(f(\theta; \alpha, \beta) = \frac{\theta^{\alpha-1}(1-\theta)^{\beta-1}}{B(\alpha,\beta)}\) | Models success probability of an anti‑memory. |
| **Stretched exponential decay** | \(s(t) = s_0 \exp(-(\lambda t)^\beta)\), \(\lambda = \ln 2 / 30\,\text{days}\), \(\beta = 0.5\) | Anti‑memory strength fading. |
| **Saturation** | \(s_{\max} = 1.6\) | Maximum effectiveness after reinforcement. |
| **Reinforcement** | \(s \leftarrow \min(s + 0.5, 1.6)\) | Strengthens anti‑memory after repeated corrections. |

---

### 7. Retrieval & Caching

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Top‑k retrieval** | Return \(k = 30\) memories with highest similarity | Balances recall and latency. |
| **Semantic cache** | Cache hit if \(\text{sim}(q, q_{\text{cached}}) > 0.9\) | Reuses retrieval results for similar queries. |
| **LRU eviction** | Remove least recently used cache entry | Limits cache size to 100 entries. |
| **LSH candidate limit** | \(C = 100\) (configurable) | Caps the number of candidates to re‑rank. |

---

### 8. Hardware & Performance Modelling

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Exponential speedup** | \(\text{speedup} \approx \frac{C \cdot F_{\text{CPU}}}{100 \cdot B_{\text{mem}}} \cdot \text{VRAM\_factor}\) | Estimates GPU vs. CPU performance. |
| **Disk speed adaptation** | \(T_{\text{prune}} = 600 \cdot (100 / S_{\text{disk}})\) | Adjusts prune interval to HDD/SSD/NVMe. |
| **Memory bandwidth scaling** | \(\tau = 7\,\text{days} \cdot (20 / B_{\text{mem}})\) | Slows personality adaptation on slow RAM. |
| **Optimal batch size** | \(B^* = \sqrt{2 C_{\text{overhead}} / C_{\text{entry}}}\) | Balances overhead and throughput. |
| **Queueing theory (Poisson process)** | \(P(\text{arrival in dt}) = \lambda dt\) | Simulates user message arrival. |

---

### 9. Information Theory & Bounds

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Entropy** | \(H(X) = -\sum p(x) \log p(x)\) | Measures diversity of memories / personality. |
| **Mutual information** | \(I(P; M) = H(P) + H(M) - H(P,M)\) | Upper bound on how much personality can capture from memories (≈13 bits). |
| **Kullback‑Leibler divergence** | \(D_{KL}(P\|Q) = \sum p \log(p/q)\) | Compares memory importance distributions. |
| **Information bottleneck** | \(\min I(M; Z) - \beta I(Z; S)\) | Compression vs. prediction trade‑off. |

---

### 10. Probability & Statistics

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Gaussian distribution** | \(\mathcal{N}(\mu, \sigma^2)\) | Noise model for embeddings, step sizes. |
| **Beta distribution** | Used in Thompson sampling | Models anti‑memory effectiveness. |
| **Poisson distribution** | \(P(k) = \frac{\lambda^k e^{-\lambda}}{k!}\) | Simulates user message arrival. |
| **Percentiles (p95)** | \(F^{-1}(0.95)\) | Latency monitoring. |
| **Sample variance** | \(\sigma^2 = \frac{1}{n-1} \sum (x_i - \bar{x})^2\) | Measures stability of performance. |

---

### 11. Optimization & Control

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Kalman filter** | Predict: \(x_{t|t-1} = A x_{t-1}\), \(P_{t|t-1} = A P_{t-1} A^T + Q\); Update: \(K = P H^T (H P H^T + R)^{-1}\), \(x = x + K(z - Hx)\), \(P = (I - KH)P\) | Detects personality drift. |
| **Gradient ascent** | \(p \leftarrow p + \eta \nabla R(p)\) | Personality update (simplified). |
| **Weight decay** | \(p \leftarrow p - \eta \lambda p\) | Regularises personality. |
| **Exponential moving average** | \(s_t = \alpha s_{t-1} + (1-\alpha) x_t\) | Smoothing for SWA and performance metrics. |

---

### 12. Algorithmic Complexity

| Operation | Time Complexity | Space Complexity |
|-----------|----------------|------------------|
| LSH hash | \(O(\text{num\_proj} \cdot d)\) = \(O(4 \cdot 128)\) | \(O(1)\) |
| LSH bucket retrieval | \(O(\text{bucket size})\) | \(O(\text{candidates})\) |
| PQ encoding | \(O(M \cdot K \cdot \text{sub\_dim})\) = \(O(16 \cdot 256 \cdot 8)\) | \(O(1)\) |
| PQ SDC (per candidate) | \(O(M)\) = \(O(16)\) | \(O(1)\) (precomputed tables) |
| Cosine similarity | \(O(d)\) = \(O(128)\) | \(O(1)\) |
| Incremental pruning | \(O(|\text{memories}| \cdot \log|\text{memories}|)\) | \(O(1)\) extra |
| Semantic cache hit | \(O(\text{cache size} \cdot d)\) = \(O(100 \cdot 128)\) | \(O(\text{cache size} \cdot d)\) |

---

### 13. Geometry & Trigonometry

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Angle between vectors** | \(\theta = \arccos(\text{sim}(x,y))\) | Used in collision probability for LSH. |
| **Hyperplane equation** | \(w \cdot x = 0\) | Random hyperplane LSH. |
| **Sphere packing bound** | \(\epsilon \sim 2^{-d H(\delta)}\) | Lower bound on quantization error. |

---

### 14. Miscellaneous

| Concept | Formula / Description | Application |
|---------|----------------------|-------------|
| **Hash functions (fxhash)** | \(h(x) = \text{fxhash}(x)\) | Fast, non‑cryptographic hashing for LRU cache keys. |
| **Unix timestamp** | \(t = \text{seconds since 1970-01-01}\) | Time decay calculations. |
| **Clamping** | \(x_{\text{clamped}} = \max(\min(x, x_{\max}), x_{\min})\) | Keeps personality and importance within bounds. |
| **Modulo arithmetic** | \(h(x) \bmod N\) | Reduces hash to bucket index. |

---

This mathematical foundation enables the AI Companion to be **fast, private, adaptive, and memory‑efficient**. The hive mind has synthesised results from linear algebra, probability, optimisation, information theory, and algorithmic complexity to create a production‑ready system.
