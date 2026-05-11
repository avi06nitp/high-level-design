# URL Shortener (TinyURL) — Complete Revision Notes

---

## 1. Requirements & Scale Estimation

**Functional:**
- Shorten long URL → short URL
- Redirect short URL → original URL
- Custom aliases
- TTL / expiration
- Analytics (click count, geography, device)

**Non-Functional:**
- Low latency redirects (<100ms)
- High availability (redirects must never go down)
- Shortened links should not be guessable (non-sequential)

**Scale (back-of-envelope):**
- 100M URLs created/day → ~1150 writes/sec
- Read:Write ratio = 100:1 → 115,000 reads/sec
- 7-char Base62 code → 62⁷ = 3.5 trillion unique URLs (more than enough)
- Storage per URL: ~500 bytes → 100M/day × 365 × 5 years = ~90 TB

---

## 2. API Design

```
POST /api/shorten
Body: { "long_url": "https://...", "custom_alias": "my-link", "ttl_seconds": 86400 }
Response: { "short_url": "https://tinyurl.com/aB3xY9", "expires_at": "..." }

GET /{shortCode}
Response: 302 Found, Location: https://original-url.com/...

GET /api/analytics/{shortCode}
Response: { "total_clicks": 1247, "unique_visitors": 892, "countries": {...}, ... }
```

---

## 3. Base62 Encoding — The Core Algorithm

### Why Base62?

| Encoding | Charset | URL-safe? | Why not? |
|----------|---------|-----------|----------|
| Base10 | `0-9` | Yes | Too long — 10 chars encode only 10B values |
| Base36 | `0-9, a-z` | Yes | Wastes uppercase letters |
| **Base62** | `0-9, a-z, A-Z` | **Yes** | **Largest URL-safe alphabet, no escaping needed** |
| Base64 | `0-9, a-z, A-Z, +, /` | **No** | `+` `/` `=` need percent-encoding (%2B, %2F) — defeats purpose |

**Key insight:** Base62 is NOT encoding the URL string. It's encoding the numeric ID from the database.

```
Flow:
1. User submits: https://www.example.com/very/long/path?with=params (200 chars)
2. Store in DB, get auto-increment ID: 123456789
3. Base62 encode the ID: encode(123456789) → "8m0Kx" (5 chars)
4. Short URL: https://tinyurl.com/8m0Kx
```

### Implementation (Java)

```java
private static final String CHARS = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

public static String encode(long num) {
    if (num == 0) return "0";
    StringBuilder sb = new StringBuilder();
    while (num > 0) {
        sb.append(CHARS.charAt((int)(num % 62)));
        num /= 62;
    }
    return sb.reverse().toString();  // ← DON'T forget reverse
}

public static long decode(String str) {
    long result = 0;
    for (char c : str.toCharArray()) {
        result = result * 62 + CHARS.indexOf(c);  // ← indexOf on CHARS, charAt on input
    }
    return result;
}
```

### Dry Run — encode(12345)

| Iteration | num | num % 62 | CHARS[idx] | num / 62 | Buffer |
|-----------|-----|----------|------------|----------|--------|
| 1 | 12345 | 7 | '7' | 199 | "7" |
| 2 | 199 | 13 | 'd' | 3 | "7d" |
| 3 | 3 | 3 | '3' | 0 | "7d3" |

Reverse → **"3d7"**

### Dry Run — decode("3d7")

| Char | CHARS.indexOf | Computation | result |
|------|---------------|-------------|--------|
| '3' | 3 | 0 × 62 + 3 | 3 |
| 'd' | 13 | 3 × 62 + 13 = 199 | 199 |
| '7' | 7 | 199 × 62 + 7 = 12345 | 12345 |

Round-trip confirmed: 12345 → "3d7" → 12345

### Capacity

| Code Length | Unique URLs | Enough For? |
|-------------|-------------|-------------|
| 6 chars | 62⁶ = 56.8 billion | Most services |
| 7 chars | 62⁷ = 3.5 trillion | Practically unlimited |

**Always use `long` not `int`** — int caps at 2.1 billion IDs, which you'll exceed at scale.

---

## 4. Short Code Generation Strategies

### Strategy 1: Auto-Increment ID + Base62

```
DB auto-increment → 123456789 → Base62 encode → "8m0Kx"
```

- ✅ Zero collisions (IDs are unique)
- ✅ Simple
- ⚠️ Predictable / guessable (sequential IDs → sequential codes)
- ⚠️ Single DB becomes bottleneck for ID generation

### Strategy 2: Hash-Based (MD5/SHA256 + Truncate)

```
MD5(long_url) → "5d41402abc4b2a76..." → take first 7 chars → "5d41402"
```

- ✅ Same URL always gets same code (deduplication)
- ⚠️ Collisions possible (birthday paradox) → need check + retry
- ⚠️ Collision probability increases as table fills up

### Strategy 3: Pre-Generated Key Service (KGS)

```
Separate service pre-generates millions of unique short codes
On request: fetch next unused code from KGS
Mark as used
```

- ✅ No collisions, no computation at request time
- ✅ Decoupled from DB
- ⚠️ KGS is a SPOF → need replication
- ⚠️ Wasted codes if service crashes between fetch and use

### Strategy 4: Snowflake ID + Base62 (Best for Distributed)

```
Snowflake ID structure (64 bits):
| 1 bit unused | 41 bits timestamp | 10 bits machine ID | 12 bits sequence |

Each server generates unique IDs independently
No coordination needed between servers
Base62 encode the Snowflake ID → short code
```

- ✅ No collisions across servers (machine ID differentiates)
- ✅ No central coordination
- ✅ Roughly time-ordered (useful for analytics)
- ✅ All shards active from day one (no hot shard problem)
- **This is the recommended approach for interviews**

---

## 5. Database Design

### Schema

```sql
CREATE TABLE urls (
    id          BIGINT PRIMARY KEY,        -- Snowflake ID
    short_code  VARCHAR(7) UNIQUE INDEX,   -- Base62 encoded
    long_url    TEXT NOT NULL,
    user_id     BIGINT,
    created_at  TIMESTAMP DEFAULT NOW(),
    expires_at  TIMESTAMP,                 -- NULL = never expires
    click_count BIGINT DEFAULT 0
);

CREATE TABLE click_events (
    id          BIGINT PRIMARY KEY,
    short_code  VARCHAR(7),
    clicked_at  TIMESTAMP,
    ip_address  VARCHAR(45),
    user_agent  TEXT,
    referrer    TEXT,
    country     VARCHAR(2),
    device_type VARCHAR(10)
);
```

### SQL vs NoSQL?

| Aspect | SQL (PostgreSQL) | NoSQL (DynamoDB/Cassandra) |
|--------|-----------------|---------------------------|
| Read pattern | Key lookup (short_code → long_url) | Key lookup — NoSQL excels here |
| Write pattern | Insert new URL | Simple inserts — both fine |
| Analytics | JOINs, aggregations easier | Need separate analytics store |
| Scaling | Sharding more complex | Built-in horizontal scaling |
| Consistency | Strong by default | Eventual (configurable) |

**Recommendation:** NoSQL (DynamoDB or Cassandra) for the URL mapping table (simple key-value lookups at massive scale). Separate analytics store (ClickHouse, Druid, or time-series DB) for click events.

### Sharding Strategy

**Shard by short_code hash** — even distribution, any shard can serve any read.

**With Snowflake IDs:** Each shard generates unique IDs independently (machine_id embedded in ID). No write conflicts. No hot shard. All shards active from day one.

**Contrast with range-based sharding:** If you shard by ID range (shard 1: 0–1M, shard 2: 1M–2M), only one shard is "active" at a time for writes → hot shard problem.

---

## 6. Caching Strategy — Three Layers

### Layer 1: CDN (Cloudflare, CloudFront)

```
User → CDN edge server → (cache hit?) → redirect instantly
                       → (cache miss?) → origin server → cache + redirect
```

- Handles 90%+ of redirect traffic
- Edge servers worldwide → lowest latency
- CDN caches the 302 response with the Location header

### Layer 2: Redis (between app server and DB)

```
App Server → Redis → (hit?) → return long_url
                   → (miss?) → DB → populate Redis → return
```

- Sub-millisecond lookups for cache hits
- LRU eviction for hot URLs
- Handles the 10% of traffic that misses CDN

### Layer 3: Browser (Be Very Careful Here)

This is where the nuance lives — covered in detail in the next sections.

---

## 7. 301 vs 302 Redirect — Critical Decision

| Aspect | 301 (Moved Permanently) | 302 (Found / Temporary) |
|--------|------------------------|------------------------|
| Browser behavior | Caches redirect forever | Asks server every time (by default) |
| SEO | Passes link juice to destination | Link juice stays on short URL |
| Analytics | **Broken** — browser never hits server again | **Works** — every click visible |
| URL updates | **Broken** — browser uses stale cached redirect | Works — can change destination |
| TTL/expiry | **Broken** — browser ignores expiry | Works — server can return 410 Gone |

**Always use 302 for URL shorteners.** 301 means you lose visibility and control forever.

**Why not 301?** Once a 301 with max-age=31536000 leaves your server, that browser uses it for a year. You can't track clicks (analytics dead), update the destination URL, expire the short URL, or delete it (browser still redirects).

---

## 8. TTL Management Across All Layers

**Core principle:** DB is the single source of truth for TTL. Every other layer derives its cache duration from the remaining TTL at the moment of the request, NOT the original TTL.

### Browser Layer

```
Cache-Control: no-store
```

- Browser never caches the redirect
- Every click hits the CDN (or origin)
- Safest approach — guarantees visibility on every click

**Alternative (optimization at extreme scale):**
```
Cache-Control: private, max-age=0, must-revalidate
ETag: "abc123"
```
- Browser checks CDN before using cache
- CDN responds 304 (not modified, ~200 bytes) instead of full 302 (~500 bytes)
- Only matters at billions of redirects/day
- For most systems: `no-store` is simpler and perfectly fine

### CDN Layer

```
CDN-Cache-Control: max-age=<remaining_ttl>
```

- CDN caches with max-age set to REMAINING TTL (not original)
- Auto-expires when URL's TTL runs out
- Example: URL created with 1-hour TTL, request comes 20 min later → CDN caches for 40 min

### Redis Layer

```
SET short:aB3xY9 "https://long-url.com" EX <remaining_ttl_seconds>
```

- Redis TTL = remaining time until expiry
- Auto-evicts expired keys

### Race Condition: Early Deletion

```
Minute 0: CDN caches redirect (max-age=300, i.e., 5 min)
Minute 2: User DELETES the short URL
Minute 3: Someone clicks link → CDN still has cached redirect → STALE
```

**Solution:** Use CDN purge APIs (Cloudflare, CloudFront all support this). On deletion:
1. Delete from DB
2. Delete from Redis
3. Purge from CDN via API call
4. Browser with `no-store` → next click hits CDN → 404

### What does the browser do when CDN returns 404?

Browser discards any cached redirect, shows the user a 404 page, and future clicks to that short URL know it's dead.

### Response Code for Expired URLs

```
GET /xyz123
410 Gone    ← not 404

Body: "This short link has expired. Expired on: 2026-01-20"
```

410 tells the client the resource existed but is permanently gone (vs 404 which means it may never have existed).

### Complete Response Headers

```java
return ResponseEntity.status(302)
    .header("Location", url.getLongUrl())
    .header("Cache-Control", "no-store")                        // browser: never cache
    .header("CDN-Cache-Control", "max-age=" + remainingTTL)     // CDN: cache with TTL
    .build();
```

---

## 9. Redirect Flow (End to End)

```
1. User clicks https://tinyurl.com/aB3xY9

2. DNS resolves to CDN edge server (Cloudflare)

3. CDN checks cache:
   HIT  → return 302 + Location header (< 10ms, 90% of traffic)
   MISS → forward to origin

4. Origin app server receives GET /aB3xY9

5. Check Redis cache:
   HIT  → get long_url (< 1ms)
   MISS → query DB

6. Check DB:
   FOUND + not expired → get long_url, populate Redis
   FOUND + expired     → return 410 Gone
   NOT FOUND           → return 404

7. Fire analytics event ASYNCHRONOUSLY (Kafka)
   - Don't block the redirect for analytics
   - Kafka → analytics consumer → ClickHouse/Druid

8. Return 302 + Location: long_url + Cache headers

9. CDN caches the response for future requests

10. Browser follows redirect to long_url
```

**Key insight:** Analytics tracking is async via Kafka. The redirect response returns immediately; the click event is processed separately. This ensures redirect latency stays under 100ms.

---

## 10. Analytics Pipeline

```
Click Event → Kafka Topic → Analytics Consumer → ClickHouse / Druid

Click Event payload:
{
    "short_code": "aB3xY9",
    "timestamp": "2026-01-21T10:36:15Z",
    "ip": "103.21.45.67",
    "user_agent": "Chrome/120 Android",
    "referrer": "twitter.com",
    "country": "IN",        ← derived from IP via GeoIP
    "device": "mobile",     ← parsed from user_agent
    "browser": "Chrome"     ← parsed from user_agent
}
```

**Why Kafka?**
- Decouples redirect path from analytics processing
- Handles traffic spikes (buffer)
- Multiple consumers: real-time dashboard, batch aggregation, alerts

**Why ClickHouse / Druid (not SQL)?**
- Columnar storage — fast aggregations (count by country, by day, etc.)
- Handles billions of events
- Time-series queries are the primary access pattern

**Don't increment click_count synchronously** — puts write pressure on URL table for every redirect. Use async pipeline and aggregate periodically, or Redis INCR → batch flush to DB.

---

## 11. CAP Theorem for URL Shortener

**Writes (URL creation):**
- Use Snowflake IDs → each shard generates unique IDs independently
- No coordination between shards → no write conflicts
- This sidesteps the CAP problem for writes entirely

**Reads (redirects):**
- Redis cache handles majority of reads
- Cache miss → DB read (single shard lookup by short_code hash)
- High availability: Redis replicas + DB replicas

**Multi-region deployment (if pushed by interviewer):**

| Approach | Model | Tradeoff |
|----------|-------|----------|
| Cassandra | AP + LWW | High availability, eventual consistency. Few hundred ms staleness acceptable for URL shortener |
| CockroachDB | CP + Raft | Strong consistency. Higher write latency due to cross-region consensus |

**For most URL shorteners: AP is the right choice.** URLs are write-once-read-many. Eventual consistency for a few hundred ms is perfectly acceptable.

---

## 12. Custom Aliases

**Considerations:**
- Validate: length (3–30 chars), allowed characters (alphanumeric + hyphens)
- Reserved words blocklist: "admin", "api", "login", "google", "amazon" etc.
- Case-sensitive or not? Most systems: case-sensitive (more capacity)
- If alias taken → return error + suggestions (append year, number, etc.)
- Store in same table, set short_code = custom_alias

**Race condition:** Two users request same custom alias simultaneously.
- Solution: UNIQUE constraint on short_code column + handle constraint violation error in app code

---

## 13. Handling Scale — Summary

| Component | How It Scales |
|-----------|---------------|
| URL creation (writes) | Snowflake IDs → sharded DB, no coordination |
| Redirects (reads) | CDN (90%) → Redis (9%) → DB (1%) |
| Short code generation | Snowflake: each server generates independently |
| Analytics ingestion | Kafka absorbs spikes, consumers scale horizontally |
| Analytics queries | ClickHouse / Druid — columnar, distributed |
| Storage | Sharded DB (hash by short_code) |
| Global latency | CDN edge servers worldwide |

---

## 14. Design Choices Summary

| Decision | Choice | Why |
|----------|--------|-----|
| Encoding | Base62 (7 chars) | Largest URL-safe alphabet, 3.5T capacity |
| ID generation | Snowflake IDs | No coordination, no collisions, time-ordered |
| Redirect code | 302 (not 301) | Preserves analytics, allows updates/expiry |
| Browser caching | Cache-Control: no-store | Every click visible, simplest correct approach |
| CDN caching | max-age = remaining TTL | Auto-expires, handles 90%+ traffic |
| Database | NoSQL (DynamoDB/Cassandra) | Simple key-value lookups at scale |
| Analytics | Async via Kafka → ClickHouse | Don't block redirects for analytics |
| Sharding | Hash by short_code | Even distribution |
| Expiry | DB is source of truth | All layers derive remaining TTL from it |
| Multi-region | AP (Cassandra) | Eventual consistency acceptable for URL reads |

---

## 15. Interview-Ready One-Line Answers

**"Why Base62?"**
> "Base62 uses 0-9, a-z, A-Z — the largest alphabet universally safe in URLs without escaping. Base64 adds +, /, = which need percent-encoding. With 62⁷ I can represent 3.5 trillion unique URLs."

**"How do you generate short codes in a distributed system?"**
> "Snowflake IDs — each shard generates unique IDs independently using embedded machine_id. No coordination, no collisions, all shards active from day one."

**"301 or 302?"**
> "302, always. 301 means the browser caches the redirect permanently — you lose analytics, can't update the destination, and can't expire the URL. 302 keeps every click visible."

**"How do you handle TTL across caching layers?"**
> "DB is the source of truth with an expiresAt column. Every other layer derives its cache duration from the remaining TTL at request time. Browser gets no-store. CDN gets max-age = remaining seconds. Redis uses EX with remaining seconds."

**"What about early deletion with CDN caching?"**
> "On deletion: delete from DB, delete from Redis, purge from CDN via their API. Browser with no-store always hits CDN, so next click gets 404."

**"How do you track analytics without slowing redirects?"**
> "Fire the click event to Kafka asynchronously — redirect returns immediately. Kafka consumers process events into ClickHouse for aggregation. Never block the redirect path."

**"CAP theorem for URL shortener?"**
> "Writes use Snowflake IDs — no coordination needed, sidesteps CAP entirely. For multi-region reads, AP with Cassandra and LWW — few hundred ms of eventual consistency is fine."

---

## 16. One-Liner Summaries (Quick Recall)

- **Base62:** Encode numeric DB ID (not URL string) into URL-safe chars, 62⁷ = 3.5T capacity
- **Encode/Decode:** encode = repeated mod 62 + reverse; decode = multiply-accumulate with indexOf on CHARS
- **Common bug:** In decode, `SIGNATURE.indexOf(input.charAt(i))` NOT `input.indexOf(SIGNATURE.charAt(i))`
- **Use long not int:** int caps at 2.1B, will overflow at scale
- **Snowflake ID:** 41-bit timestamp + 10-bit machine_id + 12-bit sequence → unique across servers, no coordination
- **302 not 301:** 301 = browser caches forever, lose analytics/updates/expiry. 302 = server sees every click
- **Three cache layers:** CDN (90% traffic, edge servers) → Redis (<1ms lookups) → DB (last resort)
- **TTL source of truth:** DB's expiresAt column; all caches derive remaining TTL from it
- **Browser caching:** Cache-Control: no-store (safest); must-revalidate + ETag only at extreme scale
- **CDN caching:** CDN-Cache-Control: max-age=remaining_ttl, auto-expires
- **Early deletion:** Delete from DB + Redis + CDN purge API
- **410 vs 404:** 410 Gone = existed but expired; 404 = never existed
- **Analytics:** Async via Kafka → ClickHouse/Druid; never block redirect path
- **Don't increment click_count synchronously:** Use Kafka + batch aggregation or Redis INCR
- **Sharding:** Hash by short_code for reads; Snowflake IDs eliminate write coordination
- **Custom aliases:** UNIQUE constraint + reserved words blocklist + handle race via constraint violation
- **CAP:** AP preferred (Cassandra + LWW); eventual consistency acceptable for URL lookups
- **KGS alternative:** Pre-generate codes in separate service, fetch on demand — simpler but KGS is SPOF