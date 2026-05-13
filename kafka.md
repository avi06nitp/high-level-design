# Message Queue (Kafka) — Complete Revision Notes

---

## 1. What and Why

**What:** A distributed, partitioned, replicated, append-only **commit log** that decouples producers from consumers. Producers write events to topics; consumers read at their own pace by tracking an offset.

**Why a message queue at all?**
- **Decouple** producers and consumers (different languages, deploy cadences, scale)
- **Absorb traffic spikes** — producer writes 10× faster than consumer processes (the prompt's nudge); buffer in the log
- **Fan-out** one event to many independent consumers (analytics + billing + email — each reads same topic)
- **Durability** — survive consumer crashes, network blips, downstream outages
- **Replay** — re-process history for backfills, bug fixes, new ML features

**Real-world scale:**
- LinkedIn: ~7 trillion msgs/day across 100+ clusters (Kafka's birthplace)
- Uber: trips, dispatch, surge pricing all flow through Kafka
- Coinbase: order events, fraud detection
- Netflix: 8M events/sec into a single Kafka cluster

---

## 2. Requirements

**Functional:**
- Producers publish messages to a named **topic**
- Consumers subscribe and receive messages
- Order preserved within a partition
- Configurable retention (time-based or size-based)

**Non-functional:**
- **Throughput:** Millions of msgs/sec per cluster (sequential disk I/O is the trick)
- **Durability:** No message loss once acknowledged
- **Low latency:** End-to-end <10 ms p99 for normal load
- **Horizontally scalable:** Add brokers → more capacity, no downtime
- **Fault tolerant:** Survive broker, disk, rack, AZ failures
- **Delivery guarantees:** At-least-once default; exactly-once available

---

## 3. Log-based vs Queue-based Messaging

This is THE conceptual question for any Kafka interview.

| Aspect | Traditional Queue (RabbitMQ, SQS) | Log-based (Kafka) |
|--------|-----------------------------------|-------------------|
| **Storage model** | Message deleted after consumed | Append-only log; retained for N days |
| **Consumer state** | Broker tracks per-consumer | Consumer tracks its own offset |
| **Fan-out** | Need separate queue per consumer, or exchange | All consumers read same log independently |
| **Replay** | Impossible (already deleted) | Trivial — seek to old offset |
| **Order** | Hard to guarantee with multiple consumers | Guaranteed within partition |
| **Throughput** | ~10–50K msgs/sec per broker | ~1M+ msgs/sec per broker (sequential I/O) |
| **Use case** | Task queues, RPC-like work distribution | Event streaming, audit logs, analytics, CDC |

**Key insight:** Kafka decouples *delivery* from *deletion*. A message lives in the log whether 0 or 1000 consumers have read it. Consumers are "lightweight" — they're just cursors over a shared log.

**Mental model:** Kafka is a **distributed tail -f over an append-only file**.

---

## 4. Core Architecture

```
                    ┌─────────────────────────────────────┐
                    │            Kafka Cluster            │
   Producers        │  ┌─────────┐ ┌─────────┐ ┌────────┐ │       Consumer Groups
   ─────────►       │  │Broker 1 │ │Broker 2 │ │Broker 3│ │       ──────────────
   (push)           │  │         │ │         │ │        │ │       (pull)
                    │  │Topic A  │ │Topic A  │ │Topic A │ │
                    │  │ P0 (L)  │ │ P0 (F)  │ │ P0 (F) │ ◄─────► Group X: 3 consumers
                    │  │ P1 (F)  │ │ P1 (L)  │ │ P1 (F) │ │
                    │  │ P2 (F)  │ │ P2 (F)  │ │ P2 (L) │ ◄─────► Group Y: 1 consumer
                    │  └─────────┘ └─────────┘ └────────┘ │
                    │              ZooKeeper / KRaft       │
                    └─────────────────────────────────────┘
                    L = Leader replica, F = Follower replica
```

**Key entities:**
- **Broker** — a Kafka server. Stores partitions on disk, serves reads/writes.
- **Topic** — logical name for a stream of events (e.g., `user-signups`).
- **Partition** — the unit of parallelism. A topic is split into N partitions, each an independent log.
- **Offset** — monotonic 64-bit integer; the position of a message within a partition.
- **Producer** — pushes records to a topic.
- **Consumer** — pulls records from a topic.
- **Consumer group** — set of consumers that cooperatively consume a topic.
- **Controller** — one broker elected to manage partition leadership and cluster metadata.
- **ZooKeeper / KRaft** — coordination layer. Old: ZK. Modern (≥2.8): KRaft (Kafka Raft) replaces ZK.

---

## 5. Partitioning — How Horizontal Scaling Works

**Why partitions exist:**
- A single log on one broker is bottlenecked by one disk's throughput
- Split topic into N partitions → spread across N brokers → N× throughput
- N partitions also = max parallelism for consumers (see consumer groups)

**Partition assignment by producer:**
```
if record.key == null:
    → round-robin across partitions (or sticky for batching)
elif:
    partition = hash(key) % num_partitions
```

**Why hash by key matters:**
- All events for `user_id=42` land on the same partition
- Same partition = ordered processing for that user
- **This is how you preserve per-entity ordering at scale**

**Example:** A ride-sharing app
- Topic: `trip-events`, 100 partitions
- Key: `trip_id`
- All status updates for trip #X go to one partition → consumer processes them in order
- 100 partitions in parallel → 100× throughput vs one big queue

**Choosing partition count — tradeoffs:**
- **Too few:** Bottleneck on throughput; can't add more consumers than partitions
- **Too many:** More open file handles, longer leader election, slower rebalancing, higher end-to-end latency
- **Rule of thumb:** Start with 2–3× the number of brokers; can only increase later (decreasing requires topic recreation)
- **Gotcha:** Increasing partition count *changes* `hash(key) % N` mapping → breaks ordering for existing keys. Plan upfront.

---

## 6. Replication — Leader / Follower / ISR

**Goal:** Durability. A broker dies, no data lost.

**Mechanism:**
- Each partition has **R** replicas (typical R=3)
- One replica is **leader**; rest are **followers**
- Producer/consumer talk **only to the leader** (followers don't serve reads in classic Kafka)
- Followers continuously fetch from leader (like a tail -f)

### ISR — In-Sync Replicas

The most important Kafka durability concept.

- **ISR** = set of replicas that are "caught up" with the leader (lag below threshold, default replica.lag.time.max.ms = 30s)
- If a follower falls behind → kicked out of ISR
- If it catches up → readmitted

**Why ISR matters for `acks=all`:**
- A write is "committed" only when **all replicas in ISR** have it
- If ISR shrinks to just the leader, writes are committed with only 1 copy → durability degraded
- **`min.insync.replicas`** setting blocks writes when ISR < threshold (e.g., min ISR = 2 → reject writes if only leader left)

### Leader Election

- When leader dies, controller picks a new leader **from the ISR**
- **Why only ISR?** Out-of-sync followers may be missing committed messages → would cause data loss
- **Unclean leader election** (config: `unclean.leader.election.enable=true`): allow non-ISR replica to become leader. Trades durability for availability. Default: **off** in modern Kafka.

```
Producer write flow (acks=all):
  Producer → Leader (Broker 1)
              ↓ append to local log
              ↓ replicate to Followers (Broker 2, 3)
              ↓ wait for ALL ISR to ack
              ↓ commit (advance high watermark)
            Send ack to Producer
```

**Producer `acks` setting — durability dial:**

| `acks` | Behavior | Latency | Durability |
|--------|----------|---------|------------|
| `0` | Fire and forget | Lowest | Can lose data on broker crash |
| `1` | Leader writes locally, acks | Medium | Lose data if leader dies before replication |
| `all` (-1) | Wait for full ISR | Highest | Survives broker loss as long as ISR ≥ min |

---

## 7. Consumer Groups — Horizontal Read Scaling

**The fundamental rule:**

> Within a consumer group, **each partition is consumed by exactly one consumer**.
> Across different groups, **the same partition is consumed independently by each group**.

```
Topic: orders, 4 partitions [P0, P1, P2, P3]

Group "billing"  (3 consumers)         Group "analytics" (1 consumer)
  C1 → P0, P3                            C1 → P0, P1, P2, P3
  C2 → P1
  C3 → P2

Group "fraud"   (5 consumers)
  C1 → P0
  C2 → P1
  C3 → P2
  C4 → P3
  C5 → idle (no partition left)
```

**Implications:**
- **Max parallelism = number of partitions.** Adding consumers beyond that → they sit idle.
- Fan-out is achieved by using **different group IDs**. Same group = work distribution. Different group = independent stream.
- Scaling out: just spin up more consumers in the group → rebalancing reassigns partitions.

### Rebalancing

Triggered when: consumer joins/leaves group, partition count changes, or session timeout.

**Classic rebalance protocol (stop-the-world):**
1. All consumers stop consuming
2. Group coordinator computes new assignment
3. Consumers reload state and resume

**Problems:** "Stop-the-world" pauses consumption. Bad for low-latency systems.

**Modern: Incremental cooperative rebalancing** (Kafka 2.4+)
- Only revoke partitions that need to move
- Other consumers keep processing
- Multi-step: revoke → assign → resume
- Default in newer clients.

**Group coordinator:** One broker manages each group's membership and offsets. Stored in the internal `__consumer_offsets` topic.

---

## 8. Offset Management

The offset is the consumer's bookmark. Where you commit it determines your delivery semantics.

```
Consumer pseudocode:
  while true:
    records = poll()                  # fetch batch from broker
    for r in records:
        process(r)                    # business logic
    commit_offsets()                  # advance the bookmark
```

**Three positions in time:**

| Term | Meaning |
|------|---------|
| **Current position** | Next offset the consumer will fetch |
| **Committed offset** | Last offset persisted to `__consumer_offsets` |
| **High watermark** | Last offset replicated to all ISR (safe to read) |
| **Log end offset** | Latest message in leader's log |

**Where you commit before vs after processing → delivery semantics:**

### Commit BEFORE processing → At-most-once
```
records = poll()
commit_offsets()         # bookmark moves first
process(records)         # if crash here, message LOST
```
Use case: metrics where losing one is fine, but duplicates are catastrophic. **Rare.**

### Commit AFTER processing → At-least-once  (default, most common)
```
records = poll()
process(records)         # if crash here, on restart we re-read
commit_offsets()         # bookmark advanced only after success
```
**Risk:** processed but crashed before commit → reprocessed on restart → **duplicates**.
**Required pattern:** consumers MUST be **idempotent**.

### Auto-commit
- `enable.auto.commit=true` (default): commits offsets in background every `auto.commit.interval.ms` (default 5s).
- Convenient but unsafe — may commit before processing finishes.
- **Production: turn this off and commit manually after processing.**

---

## 9. Delivery Guarantees

| Guarantee | What it means | How |
|-----------|---------------|-----|
| **At-most-once** | 0 or 1 deliveries | Commit before process |
| **At-least-once** | 1+ deliveries (default) | Commit after process; consumer must be idempotent |
| **Exactly-once** | Exactly 1 effective delivery | Idempotent producer + transactions |

### Idempotent Producer (`enable.idempotence=true`)
- Each producer gets a PID (producer ID) on registration
- Each message gets a sequence number per partition
- Broker dedupes on (PID, seq#) — if producer retries the same write, broker recognizes the duplicate and silently drops it
- **Solves:** producer-side duplicates from retries on flaky network
- Now **default-on** in modern Kafka.

### Transactions — Exactly-once across topics
**Use case:** Consume from topic A, do work, produce to topic B, commit offset in A — atomically.

```
producer.initTransactions()
producer.beginTransaction()
try:
    records = consumer.poll()
    for r in records:
        result = transform(r)
        producer.send("output_topic", result)
    producer.sendOffsetsToTransaction(consumer.offsets, group_id)
    producer.commitTransaction()
catch:
    producer.abortTransaction()
```

- Either everything commits or nothing does (writes + offset commit are atomic)
- Downstream consumers set `isolation.level=read_committed` to skip uncommitted records
- Cost: ~10–30% throughput overhead, higher latency

### Why exactly-once is HARD without transactions

The interview prompt: "consumer crashes mid-processing"
- Already wrote to DB? On restart, we re-poll the same offset, write again → duplicate row.
- Two practical patterns:
    - **Idempotent sink:** Use the message key (or message ID) as a unique key in your DB. Upserts. Easy with relational DBs and most KV stores.
    - **Transactional sink:** Write the offset and the result in the same DB transaction. On restart, read offset from DB, not from Kafka. (This is what Kafka Connect "exactly once" sinks do.)

**Rule of thumb:** "Exactly-once" in distributed systems means **effectively-once** — the side effect happens once, even if delivery happens multiple times. Idempotency is almost always the right answer.

---

## 10. Storage Layout — Why Kafka is FAST

Kafka brokers achieve millions of msgs/sec on commodity disks. Here's how.

**On-disk format per partition:**
```
/kafka-logs/orders-0/
    00000000000000000000.log      ← segment file (1GB chunks)
    00000000000000000000.index    ← offset → file position
    00000000000000000000.timeindex
    00000000000010234567.log
    00000000000010234567.index
    ...
```

- **Append-only log files** — never modify, only append
- Old segments **deleted in bulk** based on retention policy
- Each segment has sparse indexes for fast offset lookup

**The 3 tricks Kafka uses for throughput:**

1. **Sequential disk I/O** — HDDs/SSDs are 100–1000× faster at sequential writes than random. Appending to a log = sequential.

2. **Zero-copy reads (sendfile syscall)** — when broker serves data to consumer, it goes directly from page cache → network socket. No copy into user space, no copy back. Massive savings on read-heavy workloads.

3. **Page cache reliance** — Kafka writes to filesystem (OS page cache), not its own cache. OS handles flushing. Reads usually hit page cache → essentially free.

**Retention policies:**
- **Time-based:** delete segments older than `log.retention.hours` (default 168 = 7 days)
- **Size-based:** delete oldest segments when topic exceeds `log.retention.bytes`
- **Log compaction:** keep only the latest value per key. Used for CDC, materialized state (Kafka's own `__consumer_offsets` is a compacted topic).

---

## 11. Backpressure Handling

The interview prompt: "Producer writes 10× faster than consumer can process."

### How Kafka inherently handles it
The **log itself is the buffer.** Unlike a synchronous queue, Kafka doesn't push to consumers. Consumers **pull** at their own rate. If they fall behind, they accumulate **consumer lag** (offset distance from log end) but no messages are lost as long as retention covers the lag.

### Producer-side backpressure
- Producer has a local buffer (`buffer.memory`, default 32 MB)
- When full, producer `.send()` blocks for `max.block.ms` (default 60s) then throws
- **Knobs:**
    - `linger.ms` — wait up to N ms to batch records (latency vs throughput tradeoff)
    - `batch.size` — max batch in bytes
    - `compression.type` — gzip/snappy/lz4/zstd — fewer bytes = more throughput
- On broker overload (full disk, replication lag), broker can return throttle responses → producer slows.

### Consumer-side backpressure
- Consumer polls at its own pace — natural backpressure
- **Watch consumer lag** — your #1 health metric. Alert when lag > threshold for sustained period.
- **Scaling consumers up:** add more consumers in the group, up to partition count. Beyond that → must add partitions (with caveats).
- **Quotas:** broker can throttle a misbehaving producer/consumer per client ID.

### When the consumer simply can't keep up
1. **Add partitions + consumers** (more parallelism)
2. **Move slow work off the hot path** — consumer just writes to DB, separate worker does heavy processing
3. **Batch processing** — process records in groups (DB batch insert, vectorized ops)
4. **Skip / sample / dead-letter** — non-critical streams: drop or downsample
5. **Bigger consumer machines** — more CPU/RAM, more parallel threads per consumer

---

## 12. Failure Scenarios Quick Reference

| Failure | What happens | Mitigation |
|---------|--------------|------------|
| **Broker crash (follower)** | Removed from ISR. Cluster keeps serving. | Auto-recover when broker restarts; refetch from leader. |
| **Broker crash (leader)** | Controller elects new leader from ISR. Brief unavailability (~seconds). | `min.insync.replicas ≥ 2`, R=3 across racks/AZs. |
| **Controller crash** | New controller elected via ZK/KRaft. | Built-in. |
| **Producer crash mid-batch** | Some records may not have been sent. | Idempotent producer + retries; transactions for cross-topic atomicity. |
| **Consumer crash mid-processing** | Group coordinator reassigns its partitions after session timeout. New consumer re-reads from last committed offset → duplicate processing. | Idempotent consumer; transactional sinks. |
| **Network partition (split brain)** | ZK/KRaft ensures only one controller; only ISR leaders accept writes. | Avoid unclean leader election. |
| **Disk full** | Broker stops accepting writes for that partition. | Monitor disk, retention sized correctly, alert at 70%. |
| **Slow consumer (lag)** | Lag grows; if exceeds retention, oldest messages aged out and lost to that consumer. | Alert on lag; sufficient retention. |
| **Rebalance storm** | Frequent joins/leaves → constant rebalancing → no progress. | Tune session timeouts, use cooperative rebalancing, stable container deployments. |

---

## 13. ZooKeeper vs KRaft

**Old (pre-2.8):**
- ZooKeeper stored cluster metadata (brokers, topics, partition assignments, ACLs)
- Controller used ZK for leader election and metadata updates
- Operational pain: two distributed systems to run

**Modern (KRaft, GA in 3.3):**
- Kafka uses its **own Raft consensus** internally — no external ZK
- One set of brokers, simpler ops, faster metadata operations
- Faster recovery, larger clusters (millions of partitions feasible)
- **New designs should use KRaft.**

---

## 14. Common Interview Follow-ups

**Q: How would you ensure message ordering across partitions?**
A: You can't, by design. Order is only guaranteed within a partition. If you need global order, use 1 partition (loses scalability). Usually the right answer: partition by an entity key (user ID, account ID) so order is preserved *per entity*, which is what business requirements actually need.

**Q: How does Kafka compare to RabbitMQ / SQS?**
A: Kafka = log, optimized for throughput, replay, fan-out. RabbitMQ/SQS = traditional queues, optimized for per-message routing, work distribution, low-volume RPC-like patterns. Kafka shines for event streaming, analytics pipelines, CDC. RabbitMQ shines for task queues with complex routing.

**Q: How do you handle schema evolution?**
A: Use a **schema registry** (Confluent's, or AWS Glue). Producers register schema → broker stores a 4-byte schema ID with each record → consumers lookup ID to deserialize. Use Avro/Protobuf with rules for backward/forward compatibility (only add optional fields, etc.).

**Q: A consumer is processing too slow — what do you do?**
A: First, measure consumer lag. Then:
1. Increase partitions + consumers (if lag is partition-bound)
2. Decouple ingestion from processing (consumer writes to a faster sink, workers process async)
3. Batch + vectorize processing
4. Profile the consumer — usually a slow DB write or external API.

**Q: How does Kafka achieve "exactly-once" if the consumer crashes after writing to DB but before committing offset?**
A: Pure Kafka exactly-once only covers Kafka-to-Kafka via transactions. For Kafka-to-external-DB, you need idempotency at the sink: either dedupe by message key/ID, or write offset + data in the same DB transaction so they advance atomically.

**Q: What's the difference between high watermark and log end offset?**
A: LEO is the latest message in the leader's log. HW is the latest message replicated to all ISR — only messages ≤ HW are visible to consumers. The gap is in-flight replication.

**Q: Why can a Kafka cluster handle millions of msgs/sec on cheap disks?**
A: Sequential disk I/O + zero-copy reads + OS page cache + batching + compression. Kafka does almost no random I/O.

---

## 15. Capacity Estimation (Back-of-envelope)

**Scenario:** 1M events/sec, 1 KB each, 7-day retention, R=3

- **Write throughput:** 1M × 1 KB = 1 GB/s into the cluster
- **Replicated:** 1 GB/s × 3 = 3 GB/s of disk I/O across cluster
- **Storage:** 1 GB/s × 86,400 s/day × 7 days × 3 replicas ≈ **1.8 PB**
- **Per broker (say 10 brokers):** 180 TB/broker → need big disks or more brokers
- **Network:** 1 GB/s producers + 3 GB/s replication + N × 1 GB/s consumers (per group) → 10G NICs minimum, 25/40G typical at this scale
- **Partitions:** at 100 MB/s per partition target → 10 partitions minimum, realistically 50–200 for headroom and consumer parallelism

---

## 16. One-line Summary

**Kafka is a distributed, replicated, append-only log, partitioned for horizontal scale and consumed by independent consumer groups tracking their own offsets — turning messaging from "queue with delete-on-read" into "stream you can replay."**