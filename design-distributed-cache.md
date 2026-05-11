# Distributed Cache — Complete Revision Notes

---

## 1. Why Distributed Cache?

- CPU accesses heap memory in nanoseconds vs disk in milliseconds (10⁵x faster)
- Single node memory is limited (typically 64–256 GB)
- Need horizontal scaling → partition data across nodes
- Goal: sub-millisecond reads, offload database, absorb traffic spikes

---

## 2. Architecture Overview

```
Client → Load Balancer → Cache Cluster (N nodes on hash ring)
                              ↕
                          Database
```

- Every node holds a partition of the keyspace
- Nodes are peers (no master) OR leader-based (Redis Cluster uses leader + replicas per shard)
- Client can be "smart" (knows ring topology) or "dumb" (routes via proxy/LB)

---

## 3. Partitioning — Consistent Hashing

**Why not modular hashing (key % N)?**
- Adding/removing a node remaps almost all keys → cache stampede

**Consistent Hashing**
- Nodes and keys mapped onto same hash ring (0 to 2³²−1)
- A key is assigned to the first node clockwise from its hash position
- Adding/removing a node only affects ~1/N of the keys

**Virtual Nodes**
- Each physical node gets 100–200 virtual positions on the ring
- Solves uneven distribution from random placement
- Allows proportional assignment: beefy node → more vnodes
- When a node leaves, its load spreads evenly across many nodes (not just one neighbor)

---

## 4. Replication

**Purpose:** Availability + fault tolerance (not durability — cache is ephemeral)

**Mechanism:**
- After hashing to primary node, replicate to next K distinct physical nodes clockwise
- ⚠️ Skip vnodes that map to same physical node (otherwise < K real replicas)

**Key difference from DB replication:**
- Cache data is reconstructable from DB, so losing a replica = cache misses, not data loss
- Replication in caches is about reducing miss rate + latency, not durability
- When a node dies, you can let misses repopulate naturally (no mandatory recovery)

**Read from replicas:**
- Reduces latency (read from nearest replica)
- Reduces load on primary

---

## 5. Consistency Model

**Quorum (if needed):**
- R + W > N → strong consistency
- Typical: N=3, W=2, R=2
- For cache: usually eventual consistency is fine (R=1, W=1)

**Most caches prioritize availability over consistency (AP system)**
- Stale reads are acceptable because cache is not source of truth
- TTL provides eventual consistency guarantee

---

## 6. Handling Hot Keys

**Problem:** Single key gets 50K+ req/sec → one node overwhelmed

**Strategy 1: Read Replicas (for read-heavy hot keys)**
- Detect hot keys dynamically (track access frequency per key)
- Auto-replicate hot key to multiple nodes
- Client reads from any replica → load distributed
- Example: celebrity profile, trending topic

**Strategy 2: Key Sharding (for write-heavy hot keys)**
- Split key into N sub-keys: `key:0`, `key:1`, ..., `key:9`
- Writes distributed randomly across sub-keys
- Reads aggregate from all sub-keys
- Works well for counters/metrics (aggregatable values)
- Cache the aggregated result with short TTL

**Strategy 3: Local In-Process Cache**
- Application server caches hot keys in local memory (L1 cache)
- Very short TTL (5–10 seconds)
- Eliminates network hop entirely for hottest keys

---

## 7. Thundering Herd Problem

**Problem:** Key expires → 1000s of concurrent requests all miss cache → all hit DB simultaneously

**Solution 1: Distributed Lock (Mutex)**
- First request acquires lock: `SET key:lock NX EX 5` (Redis)
- Lock holder fetches from DB, populates cache, releases lock
- Other requests: spin-wait with short sleep, retry cache lookup
- ⚠️ Lock holder crashes → lock expires via TTL, next request retries

**Solution 2: Request Coalescing**
- Cache layer groups duplicate in-flight requests for same key
- Only one actual DB fetch; result served to all waiters
- Implemented at proxy/middleware level

**Solution 3: Early/Probabilistic Refresh**
- Before TTL expires, with some probability, a request triggers background refresh
- Key never actually expires from the perspective of most requests
- Formula: `refresh if now > (expiry - TTL * random())` where random() is small

**Solution 4: Lock-free with Stale Serving**
- Serve stale value while one thread refreshes in background
- Needs two TTLs: soft (trigger refresh) and hard (actual expiry)

---

## 8. Cache Invalidation

### Time-Based

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| TTL-based | Each key has expiry timestamp | Simple, automatic | Can serve stale data until expiry |
| Lazy deletion | Check expiry on access, delete if expired | No background work | Memory waste from unaccessed expired keys |
| Background cleanup | Periodic scan of random keys, delete expired | Reclaims memory | CPU overhead, can't check all keys |

**Redis combines lazy + background:** checks on every access AND runs periodic random sampling (20 keys, delete expired, repeat if >25% expired)

### Event-Based (Consistency with DB)

| Strategy | How | Best For |
|----------|-----|----------|
| Application-level | Write path explicitly deletes/updates cache key | Simple setups |
| CDC (Change Data Capture) | Debezium reads DB binlog → Kafka → consumers invalidate cache | Decoupled, reliable |
| Pub/Sub | DB write publishes to channel, cache nodes subscribe and invalidate | Real-time, multi-node |

**Key principle:** Prefer DELETE over UPDATE on cache during writes
- Avoids race condition: Thread A reads stale from DB, Thread B updates DB + cache, Thread A overwrites cache with stale value
- With DELETE: Thread A misses, fetches fresh from DB

---

## 9. Eviction Policies (When Memory is Full)

| Policy | Mechanism | Best For |
|--------|-----------|----------|
| **LRU** | Evict least recently used | General purpose, most common |
| **LFU** | Evict least frequently used | Skewed access (some keys always popular) |
| **Random** | Evict random key | Simple, surprisingly decent |
| **TTL-based** | Evict keys closest to expiry | When TTL is well-tuned |

**Redis Approximated LRU:**
- True LRU needs linked list tracking every access → expensive
- Redis samples K random keys (default 5), evicts the one with oldest access time
- K=10 gets very close to true LRU with much less overhead

**LRU vs LFU:**
- LRU: a single burst of requests can pollute cache (push out actually-popular keys)
- LFU: resistant to burst pollution, but slow to adapt when access patterns shift
- LFU needs decay mechanism (reduce frequency counts over time)

---

## 10. Write Policies (Cache ↔ Database)

### Cache-Aside (Lazy Loading) — Most Common

```
Read: App → Cache (hit?) → return
       App → Cache (miss) → DB → write to cache → return
Write: App → DB → delete cache key
```

- ✅ Simple, only caches what's actually read
- ⚠️ First read is always a miss (cold start)
- ⚠️ Race condition on write (mitigated by DELETE not UPDATE + short TTL)

### Write-Through

```
Write: App → Cache → DB (synchronous)
Read:  App → Cache (always hit if written through)
```

- ✅ Cache always consistent with DB
- ⚠️ Higher write latency (must wait for DB)
- ⚠️ Writes data that may never be read (wastes cache space)

### Write-Back (Write-Behind)

```
Write: App → Cache → return immediately
       Cache → DB (async, batched)
```

- ✅ Fastest writes
- ⚠️ Data loss risk if cache node dies before sync
- Mitigation: replicate write-back queue, WAL for pending writes
- Use case: high write throughput, eventual consistency acceptable

### Write-Around

```
Write: App → DB directly (skip cache)
Read:  App → Cache (miss) → DB → populate cache
```

- ✅ Avoids polluting cache with write-once data
- ⚠️ Recent writes always miss cache

---

## 11. Atomicity & Concurrency

**Redis Model:**
- Single-threaded command execution (no locks needed)
- `epoll_wait` for non-blocking I/O multiplexing
- Multiple I/O threads handle network reads/writes (Redis 6+)
- Command execution remains single-threaded → naturally atomic

**Why single-threaded works:**
- All operations are in-memory → sub-microsecond execution
- Bottleneck is network I/O, not computation
- No lock contention, no context switching overhead

**MULTI/EXEC for transactions:**
- Queue commands, execute atomically
- Optimistic locking via WATCH (CAS-style)

---

## 12. Failure Handling

### Node Failure (Temporary)

- **Detection:** Gossip protocol (nodes exchange heartbeats randomly)
- **Handling:** Sloppy quorum + hinted handoff
    - Writes redirected to next healthy node on ring with "hint"
    - When failed node recovers, hints replayed to restore data
- **For cache:** Often just accept cache misses, let repopulation handle it

### Node Failure (Permanent)

- Remove node from ring → its key range absorbed by neighbors
- If replicated: Merkle tree comparison to find divergent keys, sync only those
- For cache: less critical — data reconstructable from DB

### Cache Warming (Cold Start)

- New node or restart → cache is empty → all requests hit DB
- **Strategies:**
    - Preload known hot keys from a snapshot/seed file
    - Shadow mode: new node receives replicated traffic, builds cache before serving reads
    - Gradual traffic shift: slowly increase traffic to new node

---

## 13. Monitoring — Key Metrics

| Metric | What It Tells You |
|--------|-------------------|
| **Hit rate** | Cache effectiveness (aim for >95%) |
| **Miss rate** | DB load indicator |
| **Eviction rate** | Memory pressure — high means undersized cache |
| **p50/p99 latency** | Performance — p99 spike = hot key or network issue |
| **Memory usage** | Capacity planning |
| **Connected clients** | Load indicator |

**Dropping hit rate** = early warning for capacity issues or access pattern shift

---

## 14. Quick Comparison Table for Interviews

| Aspect | Cache | Key-Value Store |
|--------|-------|-----------------|
| Data durability | Ephemeral, reconstructable | Persistent, source of truth |
| Replication goal | Reduce miss rate | Prevent data loss |
| Node failure response | Accept misses, repopulate | Must recover/resync data |
| Consistency priority | Eventual (AP) | Configurable (CP or AP) |
| Eviction | Yes (LRU/LFU) | No (data must persist) |
| Storage | Memory only | Memory + disk (LSM tree) |

---

## 15. One-Liner Summaries (Quick Recall)

- **Consistent hashing:** Keys and nodes on same ring, only 1/N keys move on topology change
- **Virtual nodes:** Multiple ring positions per physical node for even distribution
- **Hot key:** Read-heavy → replicate to multiple nodes; Write-heavy → shard into sub-keys
- **Thundering herd:** Mutex/lock so only one request fetches from DB on miss
- **Cache-aside:** Read: check cache → miss → fetch DB → populate cache. Write: update DB → delete cache
- **Write-back:** Write to cache, async flush to DB. Fast but risky
- **LRU vs LFU:** LRU vulnerable to burst pollution; LFU slow to adapt to pattern shifts
- **Eviction:** Redis approximated LRU — sample K random keys, evict oldest-accessed
- **Invalidation:** Prefer DELETE over UPDATE to avoid stale-write race condition
- **CDC:** Debezium reads DB binlog → Kafka → consumers invalidate cache keys
- **Gossip:** Nodes randomly exchange heartbeats to detect failures
- **Hinted handoff:** Temporary takeover with hints, replay when node recovers