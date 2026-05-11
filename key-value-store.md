# Distributed Key-Value Store — Complete Revision Notes

---

## 1. Requirements Recap

- Highly available (always respond, even during failures)
- Configurable consistency (strong ↔ eventual, tunable per request)
- Reliable (handle temporary + permanent failures)
- Scalable (handle massive data, petabyte scale)
- Low latency (single-digit millisecond reads/writes)

**Core tradeoff (CAP theorem):** Can't have all three of Consistency, Availability, Partition tolerance. Since network partitions are inevitable, real choice is CP vs AP. This system is AP-leaning with configurable consistency.

---

## 2. Architecture Overview

```
Client → Any Node (coordinator for this request)
              ↓
         Hash Ring (consistent hashing with virtual nodes)
              ↓
         K replica nodes (first K distinct physical nodes clockwise)
              ↓
         Each node: WAL → Memtable → SSTables (LSM tree)
```

- Decentralized / peer-to-peer (no master node)
- Any node can coordinate any request
- Gossip protocol for cluster membership + failure detection

---

## 3. Partitioning — Consistent Hashing

**Why not key % N?**
- Adding/removing node remaps nearly all keys
- Massive data movement, unacceptable for large clusters

**Consistent Hashing**
- Both keys and nodes mapped to same hash ring (0 to 2³²−1)
- Key assigned to first node clockwise from its hash position
- Adding/removing a node moves only ~1/N of keys

**Virtual Nodes**
- Each physical node gets 100–200 positions on ring
- Why needed:
    - Without vnodes: random placement → uneven data distribution
    - With vnodes: load spreads evenly across all nodes
- Heterogeneous hardware: assign more vnodes to powerful machines
- When a node leaves: its vnodes scattered across ring → load spreads to many nodes (not just one neighbor)

**Client Routing**
- Any node can be coordinator (peer-to-peer, no SPOF)
- Client options:
    - Smart client: caches ring topology, routes directly to replica
    - Dumb client: hits any node via load balancer, that node forwards
- If coordinator dies mid-request → client retries on different node (all nodes have equal capability)

---

## 4. Replication

**Goal:** Durability + availability (data MUST NOT be lost)

**Mechanism:**
- After hashing key to a position, replicate to first K distinct physical nodes clockwise
- ⚠️ Must skip vnodes belonging to same physical node
    - Otherwise: 3 vnodes on 2 physical nodes = only 2 real replicas, not 3

**Typical K = 3** (three replicas across different nodes/racks)

**Preference list:** Ordered list of nodes responsible for a key. First K distinct physical nodes on the ring. Coordinator sends read/write to all nodes in preference list.

**Key difference from cache replication:**
- Cache: data is reconstructable, losing replica = cache miss
- KV Store: data is the source of truth, losing all replicas = DATA LOSS
- Must actively recover/resync after failures

---

## 5. Consistency — Quorum Consensus

**Parameters:**
- N = total replicas (typically 3)
- W = write quorum (nodes that must ACK a write)
- R = read quorum (nodes that must respond to a read)

**Rule: R + W > N guarantees at least one overlapping node between any read and write → strong consistency**

**Configurations and tradeoffs:**

| Config | Property | Use Case |
|--------|----------|----------|
| W=2, R=2, N=3 | Strong consistency, balanced | General purpose |
| W=1, R=N (R=3) | Fast writes, slow reads | Write-heavy (logging, metrics) |
| W=N (W=3), R=1 | Slow writes, fast reads | Read-heavy (user profiles) |
| W=1, R=1 | Eventual consistency, fastest | When staleness is acceptable |

**What happens during quorum?**
- Coordinator sends request to all N replicas in parallel
- Waits for W (or R) responses
- Returns success/data from fastest responders
- Remaining responses processed asynchronously (read repair, etc.)

**Configurable per request:** Client can specify W and R per operation, not just globally. Some writes can be strongly consistent, others eventually consistent.

---

## 6. Conflict Resolution

**Why conflicts happen:**
- With W < N, concurrent writes to same key on different replicas
- During network partitions, both sides may accept writes
- After partition heals, replicas have divergent values

### Detection: Vector Clocks

- Each node maintains a vector clock: `[NodeA: 3, NodeB: 1, NodeC: 2]`
- On every write, node increments its counter
- Two versions are:
    - **Causally ordered** if one vector dominates the other (every component ≥)
    - **Concurrent/conflicting** if neither dominates

**Example:**
```
Version X: [A:2, B:1] — written by A
Version Y: [A:1, B:2] — written by B
Neither dominates → CONFLICT
```

**Problem with vector clocks:** Size grows with number of nodes that have written the key. Mitigation: truncate oldest entries (Dynamo uses timestamp-based truncation), accepting rare false conflicts.

### Resolution Strategies

| Strategy | How | Tradeoff |
|----------|-----|----------|
| **Last-Write-Wins (LWW)** | Attach wall-clock timestamp, highest wins | Simple but loses data silently |
| **Application-level** | Return all conflicting versions to client, client merges | Complex but no data loss |
| **CRDTs** | Data structures that auto-merge without conflicts | Limited to specific types (counters, sets, registers) |

**LWW detail:** Cassandra uses this. Simple but dangerous — concurrent writes to same key, one is silently dropped. Fine for idempotent/overwrite workloads, bad for increment-style operations.

**Application-level detail:** Dynamo does this. Shopping cart example: return all versions, client takes union of items. Increases client complexity.

**CRDTs:** Conflict-free Replicated Data Types
- G-Counter: grow-only counter, each node tracks its own count, merge = take max per node
- PN-Counter: two G-Counters (one for increments, one for decrements)
- OR-Set: observed-remove set, add/remove without conflicts
- No coordination needed, always converge — but limited to specific data structures

---

## 7. Failure Handling — Temporary Failures

### Detection: Gossip Protocol

- Every node periodically (e.g., every 1 second) picks a random node and exchanges membership list
- Each entry: `{node_id, heartbeat_counter, timestamp}`
- Node increments its own heartbeat counter periodically
- If a node's heartbeat hasn't increased for threshold time → marked suspicious → eventually marked down
- Information propagates in O(log N) rounds to all nodes

### Handling: Sloppy Quorum + Hinted Handoff

**Strict quorum:** Only the K designated replicas can serve reads/writes for a key
- Problem: if one of K nodes is down, can't reach quorum → request fails → reduces availability

**Sloppy quorum:** If a designated replica is down, include next healthy node on ring
- Maintains availability at cost of potential inconsistency

**Hinted handoff:**
```
Normal: Key X → Nodes [A, B, C]
Node C down: Key X → Nodes [A, B, D] (D is temporary stand-in)
D stores data with hint: "this belongs to C"
When C recovers: D sends hinted data to C, then deletes its copy
```

- ⚠️ D's disk can fill up with hints if C is down for long → need hint TTL and cleanup
- ⚠️ This is temporary — if C never returns, need permanent failure handling

---

## 8. Failure Handling — Permanent Failures

### Anti-Entropy with Merkle Trees

**Problem:** After a long outage or permanent replacement, a replica's data has diverged. Need to find exactly which keys are inconsistent and sync only those.

**Naive approach:** Compare every key between two replicas → O(N) network transfer. Too expensive.

**Merkle tree approach:**
```
        Root Hash
       /         \
   Hash(L)     Hash(R)
   /    \      /    \
 H(A)  H(B)  H(C)  H(D)    ← leaf = hash of a key range
```

- Each node maintains a Merkle tree per key range it owns
- Leaf = hash of keys in a sub-range
- Internal nodes = hash of children
- Two nodes compare root hashes:
    - Same → data is identical, done
    - Different → recurse into children to find divergent sub-ranges
    - Only sync keys in divergent leaves

**Efficiency:** O(log N) hash comparisons to find exactly which keys differ, then sync only those.

**When to rebuild:** Merkle tree must be recalculated when a node joins/leaves (key ranges change). With virtual nodes, this can affect many trees — significant cost.

### Permanent node replacement:
- New node added to ring, takes over key ranges
- Streams data from existing replicas
- Merkle tree sync ensures correctness after bulk transfer

---

## 9. Storage Engine — LSM Tree (Log-Structured Merge Tree)

### Write Path

```
Client Write
    ↓
1. Write-Ahead Log (WAL) — append to disk (sequential I/O, fast)
    ↓
2. Memtable — in-memory sorted structure (Red-Black tree or Skip List)
    ↓
3. When memtable hits threshold (e.g., 64 MB):
   - Current memtable becomes immutable
   - New empty memtable created for incoming writes
   - Immutable memtable flushed to disk as SSTable
```

**WAL purpose:** Crash recovery. If node dies after step 1 but before step 3, replay WAL to reconstruct memtable.

**Why sorted in memory?** Enables efficient range queries and ordered flush to SSTables.

### Read Path

```
Client Read
    ↓
1. Check Memtable (current + immutable)
    ↓ (miss)
2. Check Bloom Filters for each SSTable
    ↓ (Bloom filter says "maybe present")
3. Search that SSTable (binary search on sorted keys)
    ↓
4. Return value, optionally add to cache
```

**Bloom Filter:**
- Probabilistic data structure: can say "definitely not present" or "maybe present"
- False positives possible, false negatives impossible
- Saves disk I/O by skipping SSTables that definitely don't contain the key
- Typical false positive rate: 1% with ~10 bits per key

### SSTables (Sorted String Tables)

- Immutable, sorted by key
- Structure: data blocks + index block + Bloom filter + metadata
- Once written, never modified (append-only design)
- Multiple SSTables accumulate over time → need compaction

### Compaction

**Problem:** Without compaction:
- SSTables accumulate → read must check many files → read amplification
- Deleted/overwritten keys waste space → space amplification
- Same key may exist in multiple SSTables → wasted disk

**Strategy 1: Size-Tiered Compaction**
```
Small SSTables → merge when enough of similar size → larger SSTable
```
- Good for write-heavy workloads (fewer compactions)
- Bad: space amplification (large SSTables coexist during compaction)
- Temporary disk usage can spike to 2x

**Strategy 2: Leveled Compaction**
```
Level 0: flushed from memtable (may overlap)
Level 1: non-overlapping, ~10 MB each
Level 2: non-overlapping, ~100 MB each (10x of previous level)
...
Compaction: pick SSTable from level L, merge with overlapping SSTables in L+1
```
- Good for read-heavy workloads (bounded number of SSTables to check)
- Better space amplification (only ~10% temporary overhead)
- More write amplification (same key rewritten across levels)

| Property | Size-Tiered | Leveled |
|----------|-------------|---------|
| Write amplification | Lower | Higher |
| Read amplification | Higher | Lower |
| Space amplification | Higher | Lower |
| Best for | Write-heavy | Read-heavy |

### Deletes in LSM Tree

- Can't delete in-place (SSTables are immutable)
- Write a **tombstone** marker for the key
- Tombstone suppresses older values during reads
- Actual removal happens during compaction (when tombstone meets the original entry)
- ⚠️ Tombstones must propagate to all SSTables before removal, otherwise deleted data can "resurrect"

---

## 10. Read Repair

**When:** During a quorum read, coordinator gets responses from R replicas

**If responses disagree:**
- Coordinator picks the latest version (by vector clock or timestamp)
- Returns latest to client
- Pushes latest version to replicas that had stale data (asynchronous)

**Cost:** Almost free — happens during normal read path
**Benefit:** Gradually heals inconsistencies without explicit anti-entropy

**Limitation:** Only repairs keys that are actually read. Unread stale keys remain inconsistent until anti-entropy (Merkle tree sync) runs.

---

## 11. Coordinator Role

- **Not a fixed role** — any node can coordinate any request
- Every node has full ring topology (via gossip)
- Coordinator for a request:
    1. Receives client request
    2. Hashes key → determines preference list (K replicas)
    3. Forwards read/write to all K replicas
    4. Collects quorum responses (W or R)
    5. Returns result to client

**No SPOF:** If coordinator dies mid-request:
- Client retries on a different node
- Smart client knows ring → picks another node from preference list
- Dumb client → load balancer routes to another healthy node

**Idempotency concern:** Coordinator dies after sending writes to some replicas but before ACK to client. Client retries → duplicate writes. Solution: unique request ID, replicas deduplicate.

---

## 12. Data Flow Summary

### Write Flow (end to end)
```
Client → Coordinator → hash(key) → identify K replicas
    → send write to all K replicas in parallel
    → each replica: WAL append → memtable insert
    → wait for W acknowledgments
    → return success to client
    → (async) remaining replicas acknowledge later
```

### Read Flow (end to end)
```
Client → Coordinator → hash(key) → identify K replicas
    → send read to all K replicas in parallel
    → each replica: check memtable → bloom filter → SSTable
    → wait for R responses
    → resolve conflicts (latest version by vector clock/timestamp)
    → return to client
    → (async) read repair stale replicas
```

---

## 13. Consistency Spectrum at a Glance

```
Strong ←————————————————————————→ Eventual

R+W>N          R+W=N           R=1,W=1
(overlap       (no guarantee   (fastest,
guaranteed)    but likely)     most stale)
```

**Tuning knobs:**
- Increase W → stronger write durability, slower writes
- Increase R → fresher reads, slower reads
- N=3 is the sweet spot (tolerates 1 failure with W=2,R=2)

---

## 14. Failure Handling Summary Table

| Failure Type | Detection | Mechanism | Key Detail |
|-------------|-----------|-----------|------------|
| Temporary node down | Gossip (heartbeat timeout) | Sloppy quorum + hinted handoff | Hints replayed on recovery |
| Permanent node loss | Gossip (prolonged absence) | Merkle tree anti-entropy + data streaming | O(log N) to find divergent keys |
| Network partition | Gossip (unreachable subset) | Sloppy quorum maintains availability | Conflicts resolved post-partition via vector clocks |
| Coordinator failure | Client timeout | Retry on different node | Idempotent writes via request ID |

---

## 15. Key Design Decisions Summary

| Decision | Choice | Why |
|----------|--------|-----|
| Architecture | Peer-to-peer, no master | No SPOF, any node can serve any request |
| Partitioning | Consistent hashing + vnodes | Minimal data movement on topology change |
| Replication | K physical nodes clockwise | Durability + availability |
| Consistency | Tunable quorum (R, W, N) | Different workloads need different guarantees |
| Conflict detection | Vector clocks | Captures causality, detects concurrent writes |
| Conflict resolution | LWW or app-level merge | Tradeoff: simplicity vs correctness |
| Failure detection | Gossip protocol | Decentralized, no SPOF, O(log N) propagation |
| Temp failure handling | Sloppy quorum + hinted handoff | Maintains availability during failures |
| Perm failure handling | Merkle trees | Efficient sync of only divergent keys |
| Storage engine | LSM tree | Write-optimized, sequential I/O |
| Delete mechanism | Tombstones | Immutable SSTables can't do in-place delete |
| Read optimization | Bloom filters | Skip SSTables that definitely don't have key |

---

## 16. One-Liner Summaries (Quick Recall)

- **CAP:** Partitions are inevitable; real choice is CP (reject writes during partition) vs AP (accept writes, resolve conflicts later)
- **Consistent hashing:** Keys + nodes on same ring, ~1/N keys move on topology change
- **Virtual nodes:** 100–200 positions per physical node for even distribution + heterogeneous hardware support
- **Replication:** First K distinct physical nodes clockwise; skip vnodes on same physical node
- **Quorum:** R+W>N = strong consistency; configurable per request
- **Vector clocks:** Per-node counters detect causality; neither dominates = conflict
- **LWW:** Timestamp-based, simple but loses concurrent writes silently
- **CRDTs:** Auto-merging data structures (G-Counter, OR-Set), no coordination needed
- **Gossip:** Random pairwise heartbeat exchange, failure detection in O(log N) rounds
- **Sloppy quorum:** Use next healthy node as stand-in when designated replica is down
- **Hinted handoff:** Stand-in stores data with hint, replays to recovered node
- **Merkle tree:** Hierarchical hashes, O(log N) comparisons to find divergent keys
- **LSM tree:** WAL → memtable (sorted, in-memory) → SSTable (sorted, on disk)
- **Bloom filter:** "Definitely not here" or "maybe here" — saves disk I/O on reads
- **Compaction:** Size-tiered = write-optimized; leveled = read-optimized
- **Tombstone:** Soft delete marker, physically removed during compaction
- **Read repair:** During quorum read, push latest version to stale replicas (free consistency fix)
- **Coordinator:** Not a fixed role — any node can coordinate, retry on failure, deduplicate via request ID