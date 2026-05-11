# Rate Limiter — Complete Revision Notes

---

## 1. What and Why

**What:** A mechanism that controls the number of requests a client can make within a time window. Requests exceeding the limit are throttled (rejected or delayed).

**Why:**
- Prevent resource starvation (one client hogging everything)
- Protect against DDoS and brute-force attacks
- Control costs (limit expensive API calls, third-party API usage)
- Ensure fair usage across clients
- Prevent cascading failures (upstream overload → downstream crash)

**Real-world examples:**
- Twitter: 300 tweets/3 hours, 900 read requests/15 minutes
- Google Cloud: varies per API, e.g., 600 requests/minute
- Stripe: 100 read requests/second per API key

---

## 2. Requirements

- **Accurate:** Exactly limit requests within the defined window
- **Low latency:** Rate check must add negligible overhead (<1 ms)
- **Low memory:** Handle millions of users without excessive storage
- **Distributed:** Work across multiple servers/data centers
- **Configurable:** Rules changeable without redeployment
- **Fault tolerant:** If rate limiter goes down, system should degrade gracefully
- **Clear feedback:** Clients should know they're throttled (HTTP 429) and when to retry

---

## 3. Where to Place the Rate Limiter

| Placement | How | Pros | Cons |
|-----------|-----|------|------|
| **Client-side** | Client self-throttles | No server load | Easily bypassed, unreliable |
| **Server-side** | Check on each app server | Simple | No shared state across servers |
| **Middleware / API Gateway** | Centralized check before reaching app | Shared state, single policy | Extra network hop |
| **Sidecar proxy** | Per-pod proxy (e.g., Envoy) | Decoupled from app code | Complexity in mesh |

**Most common in production:** API Gateway / middleware layer (e.g., Kong, NGINX, AWS API Gateway, Envoy)

**Why middleware?**
- Centralized: single place to define and update rules
- Shared state: all servers see the same counters
- Decoupled: app code doesn't need rate limiting logic
- Can also handle auth, logging, routing

---

## 4. Rate Limiting Algorithms

### Algorithm 1: Token Bucket

```
Bucket has capacity C tokens
Tokens added at rate R tokens/second
Each request consumes 1 token
If bucket has tokens → allow, decrement
If bucket empty → reject (HTTP 429)
```

**Parameters:** bucket size (C), refill rate (R)

**Properties:**
- ✅ Allows bursts up to C (then smooths to R/sec)
- ✅ Memory efficient: 2 values per bucket (tokens, last_refill_time)
- ✅ Simple to implement
- Used by: Amazon, Stripe

**Burst behavior:** If bucket is full (C=10) and 10 requests arrive simultaneously → all allowed. Then must wait for refill. Good for APIs that tolerate short bursts.

**Implementation detail:** Don't run a background refill thread. On each request, calculate tokens to add based on elapsed time since last refill:
```
elapsed = now - last_refill_time
tokens = min(C, tokens + elapsed * R)
last_refill_time = now
```

### Algorithm 2: Leaky Bucket

```
Queue (bucket) with fixed capacity C
Requests enter queue
Processed at fixed rate R (like water leaking from bucket)
If queue full → reject
```

**Parameters:** bucket size (C), outflow rate (R)

**Properties:**
- ✅ Smooths traffic to constant rate (no bursts at all)
- ✅ Useful when downstream needs steady throughput
- ⚠️ Burst of requests fills queue → newer requests rejected even if old ones are slow
- ⚠️ Old requests in queue may become stale

**Difference from Token Bucket:**
- Token bucket: allows bursts up to capacity, then rate-limits
- Leaky bucket: strictly constant output rate, queue absorbs bursts
- Token bucket controls *average rate with burst tolerance*
- Leaky bucket controls *output rate strictly*

### Algorithm 3: Fixed Window Counter

```
Time divided into fixed windows (e.g., 0:00–1:00, 1:00–2:00)
Counter per window
Each request increments counter
If counter > limit → reject
Counter resets at window boundary
```

**Parameters:** window size (e.g., 1 minute), limit per window

**Properties:**
- ✅ Simple, memory efficient (1 counter + timestamp per user)
- ⚠️ **Boundary problem:**
    - Limit = 100/minute
    - 100 requests at 0:59, 100 requests at 1:01
    - 200 requests in 2 seconds, but each window sees only 100
    - 2x burst at window boundary

### Algorithm 4: Sliding Window Log

```
Store timestamp of every request in a sorted log
On new request:
  1. Remove all timestamps older than (now - window_size)
  2. If log size < limit → allow, add timestamp
  3. Else → reject
```

**Parameters:** window size, limit

**Properties:**
- ✅ Perfectly accurate, no boundary problem
- ⚠️ Memory expensive: stores every request timestamp
- ⚠️ For high-traffic users: millions of timestamps per window

### Algorithm 5: Sliding Window Counter (Hybrid — Best of Both)

```
Combines fixed window counter with sliding window concept

current_window_count = requests in current window
previous_window_count = requests in previous window
overlap = % of previous window that overlaps with sliding window

weighted_count = previous_window_count × overlap + current_window_count

If weighted_count < limit → allow
Else → reject
```

**Example:**
```
Window = 1 minute, Limit = 100
Current time: 1:15 (15 seconds into current window)
Previous window (0:00–1:00): 84 requests
Current window (1:00–2:00): 36 requests

Overlap of previous window = (60 - 15) / 60 = 75%
Weighted = 84 × 0.75 + 36 = 63 + 36 = 99 → ALLOW (< 100)
```

**Properties:**
- ✅ Smooths the boundary spike of fixed window
- ✅ Memory efficient (2 counters per user, same as fixed window)
- ✅ Reasonably accurate (not perfect, but close enough)
- ⚠️ Assumes requests in previous window were evenly distributed (approximation)
- **Most practical choice for production systems**

### Algorithm Comparison

| Algorithm | Accuracy | Memory | Burst Handling | Complexity |
|-----------|----------|--------|----------------|------------|
| Token Bucket | Good | Very low (2 values) | Allows controlled bursts | Low |
| Leaky Bucket | Good | Low (queue) | No bursts (constant rate) | Low |
| Fixed Window | Boundary issue | Very low (1 counter) | 2x burst at boundary | Very low |
| Sliding Window Log | Perfect | High (all timestamps) | None | Medium |
| Sliding Window Counter | Good (approx) | Very low (2 counters) | Smoothed boundary | Low |

**Interview recommendation:** Know Token Bucket and Sliding Window Counter in depth. Token Bucket for its simplicity and burst tolerance, Sliding Window Counter for its practicality.

---

## 5. Rate Limiting Rules & Configuration

**Rules stored externally** (config file, database, config service) — changeable without redeployment.

**Example rules (YAML-like):**
```
rules:
  - key: user_id
    endpoint: /api/messages
    limit: 100
    window: 60s

  - key: ip_address
    endpoint: /api/login
    limit: 5
    window: 300s      # brute-force protection

  - key: api_key
    endpoint: /api/*
    limit: 1000
    window: 3600s     # hourly quota per API key

  - key: global
    endpoint: /api/search
    limit: 10000
    window: 60s       # system-wide protection
```

**Rate limit key options:**
- Per user ID (authenticated requests)
- Per IP address (unauthenticated, login endpoints)
- Per API key (B2B SaaS usage)
- Per endpoint (protect expensive operations)
- Combination: user_id + endpoint (granular control)
- Global (system-wide safety cap)

---

## 6. Distributed Rate Limiting

### The Problem

Single-server rate limiter is simple — use an in-memory counter. But with multiple servers:
```
Server A: sees 50 requests from User X → allows (limit 100)
Server B: sees 50 requests from User X → allows (limit 100)
Total: 100 requests, but each server thought it was under limit
Without shared state: 2x over-limit
```

### Solution 1: Centralized Counter (Redis)

```
All servers check/increment counter in Redis
Redis: single-threaded → atomic operations → no race conditions

INCR user:123:minute:1715400000    → returns new count
EXPIRE user:123:minute:1715400000 60  → auto-cleanup
```

**Redis commands for Token Bucket:**
```
-- Lua script (atomic in Redis)
local tokens = tonumber(redis.call('GET', key) or bucket_size)
local last_refill = tonumber(redis.call('GET', key..':time') or now)
local elapsed = now - last_refill
tokens = math.min(bucket_size, tokens + elapsed * refill_rate)
if tokens >= 1 then
    tokens = tokens - 1
    redis.call('SET', key, tokens)
    redis.call('SET', key..':time', now)
    return 1  -- allowed
else
    return 0  -- rejected
end
```

**Why Lua script?** Redis executes Lua atomically — no race condition between GET and SET. Without Lua, two concurrent requests could both read tokens=1, both decrement, both succeed (should have rejected one).

**Properties:**
- ✅ Strongly consistent counts
- ⚠️ Redis becomes SPOF → need Redis Cluster or Sentinel for HA
- ⚠️ Network latency per request (typically <1ms within same DC)
- ⚠️ Cross-datacenter: requests must reach centralized Redis → higher latency

### Solution 2: Local Counter + Sync (Approximate)

```
Each server maintains local counter
Periodically sync with central store (e.g., every 1 second)
Between syncs: local decisions may over-allow
```

**Properties:**
- ✅ No per-request network call
- ✅ Faster
- ⚠️ Can exceed limit by up to (num_servers × sync_interval × request_rate)
- Good when approximate limiting is acceptable

### Solution 3: Sticky Sessions

```
Load balancer routes all requests from same user to same server
Each server tracks its own users locally → no shared state needed
```

**Properties:**
- ✅ No distributed state
- ⚠️ Uneven load distribution
- ⚠️ Server failure → user's state lost → resets counter
- ⚠️ Doesn't work with multiple data centers

### Race Condition Deep Dive

**Problem with naive Redis GET-then-SET:**
```
Time 0: Server A does GET counter → 99
Time 1: Server B does GET counter → 99
Time 2: Server A does SET counter = 100 → allows
Time 3: Server B does SET counter = 100 → allows (SHOULD REJECT)
```

**Solutions:**
1. **Lua scripts** (recommended): atomic read-check-increment in Redis
2. **Redis INCR**: single atomic command, returns new value → check after increment
3. **Redis SET with NX + MULTI/EXEC**: optimistic locking

---

## 7. Multi-Datacenter Rate Limiting

**Challenge:** User hits DC-East and DC-West simultaneously. Each DC has its own Redis.

**Option 1: Global Redis (single source of truth)**
- All DCs check one Redis cluster
- ⚠️ Cross-DC latency (50–200ms) → unacceptable for rate limiting
- Only viable if DCs are in same region

**Option 2: Local Redis + Async Sync**
- Each DC has local Redis
- Periodically sync counters across DCs (e.g., every 5 seconds)
- Accept eventual consistency: can temporarily exceed limit by ~(N_datacenters × sync_delay × rate)
- **Most practical for global systems**

**Option 3: Partitioned Limits**
- Split limit across DCs: if global limit = 1000/min and 2 DCs → 500/min each
- ⚠️ Doesn't adapt to traffic skew (one DC might get 80% of traffic)
- Can use weighted split based on traffic patterns

---

## 8. Response Design

### HTTP Response for Throttled Requests

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1715400060

{
    "error": "rate_limit_exceeded",
    "message": "Too many requests. Please retry after 30 seconds."
}
```

**Standard headers (return on EVERY response, not just 429):**

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | Max requests allowed in window |
| `X-RateLimit-Remaining` | Requests left in current window |
| `X-RateLimit-Reset` | Unix timestamp when window resets |
| `Retry-After` | Seconds to wait before retrying (on 429) |

**Why return on every response?** Clients can proactively slow down as they approach the limit, avoiding 429s entirely. Good API design.

### Alternative to Hard Rejection

**Soft throttling / degraded response:**
- Instead of 429, return cached/simplified response
- Reduce response quality rather than rejecting entirely
- Example: search returns cached results instead of live query

---

## 9. Handling Rate Limiter Failure

**What if Redis / rate limiter service goes down?**

| Strategy | Behavior | Risk |
|----------|----------|------|
| **Fail-open** | Allow all requests through | System unprotected, potential overload |
| **Fail-closed** | Reject all requests | Total service outage for clients |
| **Local fallback** | Switch to in-memory per-server limiting | Approximate but functional |

**Recommended: Fail-open with local fallback**
- If Redis is unreachable, fall back to local in-memory rate limiting
- Less accurate (each server limits independently) but prevents both extremes
- Alert ops team immediately for Redis recovery

**Rate limiter itself needs:**
- Health checks and monitoring
- Redis Sentinel or Cluster for HA
- Circuit breaker: if Redis latency spikes, bypass rate limiting temporarily

---

## 10. Detailed Architecture

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐
│  Client  │────→│  API Gateway  │────→│  App Servers │────→│ Database │
└─────────┘     │  / Middleware │     └─────────────┘     └──────────┘
                └──────┬───────┘
                       │
                ┌──────▼───────┐     ┌────────────────┐
                │ Rate Limiter │────→│  Redis Cluster  │
                │   Service    │     │ (shared counter │
                └──────────────┘     │   store)        │
                       ↑             └────────────────┘
                ┌──────┴───────┐
                │ Rules Config │
                │  (DB/File)   │
                └──────────────┘
```

**Request flow:**
1. Client sends request to API Gateway
2. Gateway extracts rate limit key (user ID, IP, API key)
3. Rate limiter service checks counter in Redis
4. If under limit → increment counter, forward to app server
5. If over limit → return 429 with headers
6. Rate limit rules loaded from config (cached locally, refreshed periodically)

---

## 11. Advanced Topics

### Tiered / Hierarchical Rate Limiting

Multiple limits applied simultaneously:
```
Per-second:  10 req/sec   (burst protection)
Per-minute: 200 req/min   (sustained rate)
Per-hour:  5000 req/hour  (quota enforcement)
Per-day:  50000 req/day   (billing tier)
```
Request must pass ALL tiers. Lowest tier that's exceeded triggers 429.

### Adaptive Rate Limiting

- Dynamically adjust limits based on system health
- If CPU > 80% or response latency > p99 threshold → tighten limits
- If system is healthy → relax limits
- Requires feedback loop from monitoring system

### Rate Limiting by Cost (Weighted Requests)

Not all requests are equal:
```
GET  /users     → costs 1 token
POST /users     → costs 5 tokens
GET  /search    → costs 10 tokens (expensive query)
POST /export    → costs 50 tokens (heavy operation)
```
Token bucket works naturally here — deduct variable tokens per request type.

### Client-Side Backoff

Good clients implement exponential backoff on 429:
```
retry_delay = base_delay × 2^attempt + random_jitter
```
- Prevents retry storm (all clients retrying at same time)
- Jitter prevents synchronization of retries
- Max retry cap (e.g., 5 retries, then fail)

---

## 12. Design Choices Summary

| Decision | Choice | Why |
|----------|--------|-----|
| Placement | API Gateway / Middleware | Centralized, shared state, decoupled from app |
| Algorithm | Token Bucket or Sliding Window Counter | Burst tolerance + memory efficiency |
| Shared state | Redis (Lua scripts for atomicity) | Fast, atomic, widely supported |
| HA for Redis | Redis Cluster + Sentinel | No SPOF for rate limiter |
| Multi-DC | Local Redis + async sync | Avoid cross-DC latency penalty |
| Failure mode | Fail-open + local fallback | Balance between protection and availability |
| Config | External rules store | Hot-reloadable without deployment |
| Client communication | 429 + standard rate limit headers | Industry standard, enables client-side backoff |

---

## 13. One-Liner Summaries (Quick Recall)

- **Token Bucket:** Tokens refill at steady rate, each request consumes a token, allows bursts up to bucket size
- **Leaky Bucket:** Fixed outflow rate, queue absorbs bursts, strictly constant output
- **Fixed Window:** Simple counter per time window, 2x burst problem at boundaries
- **Sliding Window Log:** Store every timestamp, perfectly accurate, memory expensive
- **Sliding Window Counter:** Weighted average of current + previous window, practical best choice
- **Redis atomicity:** Use Lua scripts or INCR — naive GET-then-SET has race conditions
- **Distributed:** Centralized Redis for accuracy, local counters for speed, sticky sessions for simplicity
- **Multi-DC:** Local Redis + async sync, accept temporary over-limit during sync gaps
- **Fail-open:** If rate limiter dies, allow requests through (with local fallback)
- **429 headers:** Always return X-RateLimit-Limit, Remaining, Reset on every response (not just 429)
- **Weighted limiting:** Different endpoints cost different tokens (GET=1, search=10, export=50)
- **Adaptive:** Tighten limits when system health degrades (CPU, latency thresholds)
- **Client backoff:** Exponential backoff + jitter on 429 to prevent retry storms
- **Tiered limits:** Per-second, per-minute, per-hour applied simultaneously — request must pass all