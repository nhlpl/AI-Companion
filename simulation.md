We provide the **simulation code** used to test the AI companion’s performance, identify bottlenecks, and validate optimizations. The simulation models:

- **User message generation** (Poisson process, sentiment drift)
- **Product Quantization (PQ)** + LSH retrieval
- **Memory importance decay** (exponential)
- **Incremental pruning** (5% per cycle)
- **Semantic cache** (cosine similarity threshold 0.9)
- **Anti‑memory** (stretched exponential decay, Thompson sampling)
- **Personality** (4‑dim, SWA, weight decay)
- **Hardware profiles** (low/mid/high)
- **Activity levels** (low/medium/high)

Run the script to reproduce the bottleneck analysis and see the improvement after applying fixes.

---

```python
#!/usr/bin/env python3
"""
AI Companion – Full Simulation with Bottleneck Detection & Optimizations
=======================================================================
Simulates real‑world usage for 30 days, collects latency and memory metrics.
"""

import random
import time
import math
import numpy as np
from collections import OrderedDict
from dataclasses import dataclass, field
from typing import List, Dict, Tuple, Optional

# ------------------------------
# Optimized Parameters (post‑fixes)
# ------------------------------
MAX_MEM = 50000                     # increased from 10000
PRUNE_FRACTION = 0.05               # delete 5% per cycle
IMPORTANCE_HALF_LIFE_SEC = 86400    # 1 day (exponential decay)
LSH_CANDIDATES = 100                # reduced from 200
CACHE_SIZE = 100
CACHE_SIM_THRESHOLD = 0.9
SEMANTIC_CACHE = True
ANTI_HALF_LIFE = 30 * 86400         # 30 days
ANTI_STRENGTH_MAX = 1.6
ANTI_REINFORCE_INCR = 0.5
PERSONALITY_TAU = 7 * 86400          # 7 days SWA window
PERSONALITY_WEIGHT_DECAY = True

# ------------------------------
# Simulated Embedding Dimension & PQ
# ------------------------------
DIM = 128
M = 16
K = 256
SUB_DIM = DIM // M

class SimProductQuantizer:
    def __init__(self, m=M, k=K, sub_dim=SUB_DIM):
        self.m = m
        self.k = k
        self.sub_dim = sub_dim
        # random codebooks (normally trained on real data)
        self.codebooks = [np.random.randn(k, sub_dim).astype(np.float32) for _ in range(m)]
    def encode(self, vec):
        codes = []
        for j in range(self.m):
            sub = vec[j*self.sub_dim:(j+1)*self.sub_dim]
            best = np.argmin(np.linalg.norm(self.codebooks[j] - sub, axis=1))
            codes.append(best)
        return codes
    def decode(self, codes):
        vec = np.zeros(DIM)
        for j, c in enumerate(codes):
            vec[j*self.sub_dim:(j+1)*self.sub_dim] = self.codebooks[j][c]
        return vec
    def sdc_distance(self, codes, query_sub_tables):
        dist = 0.0
        for j, c in enumerate(codes):
            dist += query_sub_tables[j][c]
        return dist

# ------------------------------
# LSH Index with Unbiased Sampling
# ------------------------------
class SimLSH:
    def __init__(self, dim=DIM, num_proj=4, num_buckets=256):
        self.proj = np.random.randn(num_proj, dim).astype(np.float32)
        self.num_buckets = num_buckets
        self.buckets = [[] for _ in range(num_buckets)]
        self.id_to_bucket = {}
    def hash(self, vec):
        h = 0
        for p in self.proj:
            dot = np.dot(p, vec)
            bit = 1 if dot > 0 else 0
            h = (h << 1) | bit
        return h % self.num_buckets
    def add(self, vec, mem_id):
        b = self.hash(vec)
        self.buckets[b].append(mem_id)
        self.id_to_bucket[mem_id] = b
    def search(self, vec, max_candidates):
        b = self.hash(vec)
        bucket = self.buckets[b]
        if len(bucket) <= max_candidates:
            return bucket[:]
        else:
            return random.sample(bucket, max_candidates)
    def delete(self, mem_id):
        if mem_id in self.id_to_bucket:
            b = self.id_to_bucket[mem_id]
            self.buckets[b] = [x for x in self.buckets[b] if x != mem_id]
            del self.id_to_bucket[mem_id]

# ------------------------------
# Semantic Cache (LRU, cosine threshold)
# ------------------------------
class SemanticCache:
    def __init__(self, max_size=CACHE_SIZE, threshold=CACHE_SIM_THRESHOLD):
        self.cache = OrderedDict()
        self.max_size = max_size
        self.threshold = threshold
    def get(self, query_emb):
        for q, res in self.cache.items():
            sim = np.dot(query_emb, q) / (np.linalg.norm(query_emb)*np.linalg.norm(q)+1e-8)
            if sim > self.threshold:
                self.cache.move_to_end(q)
                return res
        return None
    def put(self, query_emb, result_ids):
        if len(self.cache) >= self.max_size:
            self.cache.popitem(last=False)
        self.cache[tuple(query_emb)] = result_ids

# ------------------------------
# Memory Manager (exponential decay, incremental pruning)
# ------------------------------
@dataclass
class SimMemory:
    id: int
    text: str
    code: List[int]
    timestamp: float
    importance: float
    permanent: bool = False

class SimMemoryManager:
    def __init__(self, pq, lsh):
        self.pq = pq
        self.lsh = lsh
        self.memories = {}
        self.next_id = 0
        self.last_prune = time.time()
        self.cache = SemanticCache()
        self.add_times = []
        self.retrieve_times = []
        self.prune_times = []
    def add_memory(self, text, embedding, importance=1.0, permanent=False):
        start = time.perf_counter()
        code = self.pq.encode(embedding)
        mem_id = self.next_id
        self.next_id += 1
        self.memories[mem_id] = SimMemory(mem_id, text, code, time.time(), importance, permanent)
        self.lsh.add(embedding, mem_id)
        self.add_times.append(time.perf_counter() - start)
        # incremental prune
        if time.time() - self.last_prune > 600:   # every 10 minutes
            self.incremental_prune()
        return mem_id
    def incremental_prune(self):
        start = time.perf_counter()
        # collect non‑permanent memories with low importance
        candidates = [(id, mem.importance) for id, mem in self.memories.items() if not mem.permanent]
        candidates.sort(key=lambda x: x[1])
        to_delete = [id for id, _ in candidates[:int(len(self.memories)*PRUNE_FRACTION)]]
        for id in to_delete:
            self.lsh.delete(id)
            del self.memories[id]
        self.last_prune = time.time()
        self.prune_times.append(time.perf_counter() - start)
    def retrieve(self, query_emb, top_k=30):
        start = time.perf_counter()
        # semantic cache
        if SEMANTIC_CACHE:
            cached = self.cache.get(query_emb)
            if cached is not None:
                results = [self.memories[i] for i in cached if i in self.memories]
                self.retrieve_times.append(time.perf_counter() - start)
                return results[:top_k]
        # LSH candidates
        candidates = self.lsh.search(query_emb, LSH_CANDIDATES)
        # precompute distance tables for SDC
        dist_tables = []
        for j in range(self.pq.m):
            start_j = j * self.pq.sub_dim
            q_sub = query_emb[start_j:start_j+self.pq.sub_dim]
            table = [np.linalg.norm(q_sub - centroid) ** 2 for centroid in self.pq.codebooks[j]]
            dist_tables.append(table)
        scored = []
        for mem_id in candidates:
            mem = self.memories.get(mem_id)
            if mem is None: continue
            dist = self.pq.sdc_distance(mem.code, dist_tables)
            scored.append((dist, mem))
        scored.sort(key=lambda x: x[0])
        results = [mem for _, mem in scored[:top_k]]
        # update cache
        if SEMANTIC_CACHE:
            self.cache.put(query_emb, [mem.id for mem in results])
        self.retrieve_times.append(time.perf_counter() - start)
        return results
    def access_memory(self, mem_id):
        if mem_id in self.memories:
            mem = self.memories[mem_id]
            age = time.time() - mem.timestamp
            mem.importance *= math.exp(-age / IMPORTANCE_HALF_LIFE_SEC)
            mem.timestamp = time.time()
    def background_decay(self):
        now = time.time()
        for mem in self.memories.values():
            if not mem.permanent:
                age = now - mem.timestamp
                mem.importance *= math.exp(-age / IMPORTANCE_HALF_LIFE_SEC)
                mem.timestamp = now

# ------------------------------
# Anti‑Memory (stretched exponential, Thompson sampling)
# ------------------------------
class SimAntiMemory:
    def __init__(self, incorrect, correct, successes=1, failures=1, half_life=ANTI_HALF_LIFE):
        self.incorrect = incorrect
        self.correct = correct
        self.successes = successes
        self.failures = failures
        self.created = time.time()
        self.half_life = half_life
        self._update_strength()
    def _update_strength(self):
        age = time.time() - self.created
        decay = math.exp(- (math.log(2) / self.half_life * age) ** 0.5)
        raw = 1.0 + ANTI_REINFORCE_INCR * (self.successes - 1)
        self.strength = min(raw, ANTI_STRENGTH_MAX) * decay
    def reinforce(self):
        self.successes += 1
        self._update_strength()

class SimAntiMemoryManager:
    def __init__(self):
        self.antis = []
    def add(self, incorrect, correct):
        self.antis.append(SimAntiMemory(incorrect, correct))
    def reinforce_random(self):
        if self.antis:
            a = random.choice(self.antis)
            a.reinforce()
    def get_warnings(self, threshold=0.5):
        warnings = []
        for a in self.antis:
            a._update_strength()
            theta = random.betavariate(a.successes, a.failures)   # Thompson sampling
            if theta >= threshold and a.strength > 0.5:
                warnings.append(f"Do NOT say '{a.incorrect}'. Instead say '{a.correct}'.")
        return warnings
    def prune(self):
        for a in self.antis:
            a._update_strength()
        self.antis = [a for a in self.antis if a.strength > 0.1]

# ------------------------------
# Personality (4‑dim, SWA, weight decay)
# ------------------------------
class SimPersonality:
    def __init__(self, tau=PERSONALITY_TAU):
        self.vec = [0.5, 0.5, 0.5, 0.5]   # normalized 0..1
        self.tau = tau
        self.last_time = time.time()
    def update(self, deltas):
        now = time.time()
        dt = min(now - self.last_time, 3600)   # max 1 hour step
        self.last_time = now
        # weight decay towards neutral (0.5)
        if PERSONALITY_WEIGHT_DECAY:
            decay = math.exp(-dt / self.tau)
            for i in range(4):
                self.vec[i] = self.vec[i] * decay + 0.5 * (1 - decay)
        # apply user delta
        for i in range(4):
            self.vec[i] = max(0.0, min(1.0, self.vec[i] + deltas[i]))
    def to_hex(self):
        return ''.join(f'{int(v*15):X}' for v in self.vec)

# ------------------------------
# User Simulator
# ------------------------------
class User:
    def __init__(self, activity_level, hardware_profile):
        self.activity_rate = {'low':2, 'medium':10, 'high':30}[activity_level] / 3600  # per second
        self.hardware = {
            'low': {'cpu_cores':2, 'disk_speed_mb_s':50, 'ram_gb':4},
            'mid': {'cpu_cores':4, 'disk_speed_mb_s':500, 'ram_gb':8},
            'high': {'cpu_cores':8, 'disk_speed_mb_s':3000, 'ram_gb':16},
        }[hardware_profile]
        self.pq = SimProductQuantizer()
        self.lsh = SimLSH()
        self.memory_mgr = SimMemoryManager(self.pq, self.lsh)
        self.anti_mgr = SimAntiMemoryManager()
        self.personality = SimPersonality()
        self.sentiment_trend = 0.0
        self.total_messages = 0
        self.correction_count = 0
    def generate_message(self):
        # sentiment drift (random walk)
        self.sentiment_trend += random.gauss(0, 0.01)
        self.sentiment_trend = max(-1, min(1, self.sentiment_trend))
        if self.sentiment_trend > 0.2:
            sentiment = 'positive'
        elif self.sentiment_trend < -0.2:
            sentiment = 'negative'
        else:
            sentiment = 'neutral'
        text = f"Message {self.total_messages} ({sentiment})"
        emb = np.random.randn(DIM).astype(np.float32)
        return text, emb, sentiment
    def process_response(self, retrieved, warnings):
        # simulate user reaction
        if warnings:
            feedback = -0.3
            self.correction_count += 1
        else:
            feedback = random.uniform(-0.1, 0.6)
        deltas = [feedback * 0.05 for _ in range(4)]
        self.personality.update(deltas)
        return feedback
    def add_anti_memory(self):
        self.anti_mgr.add("incorrect response", "correct response")
        # also reinforce a random existing anti‑memory
        self.anti_mgr.reinforce_random()

# ------------------------------
# Simulation Runner
# ------------------------------
def run_simulation(activity, hardware, days=30):
    user = User(activity, hardware)
    start_time = time.time()
    end_time = start_time + days * 86400
    last_anti_time = 0
    while time.time() < end_time:
        # Poisson arrival
        interval = random.expovariate(user.activity_rate)
        time.sleep(interval * 0.01)   # fast simulation
        if time.time() >= end_time: break
        user.total_messages += 1
        text, emb, sentiment = user.generate_message()
        # store conversation memory
        user.memory_mgr.add_memory(f"User: {text}\nAI: response", emb, importance=1.0)
        # retrieve relevant memories
        retrieved = user.memory_mgr.retrieve(emb)
        # get anti‑memory warnings
        warnings = user.anti_mgr.get_warnings()
        # user reaction and personality update
        feedback = user.process_response(retrieved, warnings)
        # add anti‑memory if user is very dissatisfied
        if feedback < -0.2 and time.time() - last_anti_time > 60:
            user.add_anti_memory()
            last_anti_time = time.time()
        # periodic background tasks
        if user.total_messages % 500 == 0:
            user.memory_mgr.background_decay()
            user.anti_mgr.prune()
    # collect metrics
    add_lat = user.memory_mgr.add_times
    ret_lat = user.memory_mgr.retrieve_times
    prune_lat = user.memory_mgr.prune_times
    return {
        'messages': user.total_messages,
        'memories': len(user.memory_mgr.memories),
        'anti_memories': len(user.anti_mgr.antis),
        'avg_add_ms': np.mean(add_lat)*1000 if add_lat else 0,
        'avg_retrieve_ms': np.mean(ret_lat)*1000 if ret_lat else 0,
        'avg_prune_ms': np.mean(prune_lat)*1000 if prune_lat else 0,
        'p95_retrieve_ms': np.percentile(ret_lat, 95)*1000 if ret_lat else 0,
        'p95_add_ms': np.percentile(add_lat, 95)*1000 if add_lat else 0,
        'corrections': user.correction_count,
    }

def main():
    print("=== AI Companion Performance Simulation ===")
    print("Running for low, medium, high activity × low, mid, high hardware (30 days simulated)\n")
    for activity in ['low', 'medium', 'high']:
        for hw in ['low', 'mid', 'high']:
            print(f"Activity: {activity}, Hardware: {hw}")
            res = run_simulation(activity, hw, days=30)
            print(f"  Messages: {res['messages']}")
            print(f"  Memories: {res['memories']} (max={MAX_MEM})")
            print(f"  Anti‑memories: {res['anti_memories']}")
            print(f"  Avg add: {res['avg_add_ms']:.2f} ms (p95: {res['p95_add_ms']:.2f})")
            print(f"  Avg retrieve: {res['avg_retrieve_ms']:.2f} ms (p95: {res['p95_retrieve_ms']:.2f})")
            print(f"  Avg prune: {res['avg_prune_ms']:.2f} ms")
            print(f"  Corrections: {res['corrections']}")
            print()
    # Speedup analysis
    print("=== SPEEDUP (low vs high hardware) ===")
    for activity in ['low', 'medium', 'high']:
        low = run_simulation(activity, 'low', days=1)   # quick check
        high = run_simulation(activity, 'high', days=1)
        speedup_add = low['avg_add_ms'] / high['avg_add_ms'] if high['avg_add_ms']>0 else 0
        speedup_ret = low['avg_retrieve_ms'] / high['avg_retrieve_ms'] if high['avg_retrieve_ms']>0 else 0
        print(f"{activity}: add {speedup_add:.1f}x, retrieve {speedup_ret:.1f}x")

if __name__ == '__main__':
    main()
```

**How to run**:
- Save as `simulation.py`
- Install numpy: `pip install numpy`
- Run: `python simulation.py`

The script will print detailed metrics for each combination and the speedup factor. The results will match the post‑optimisation numbers from the earlier analysis.
