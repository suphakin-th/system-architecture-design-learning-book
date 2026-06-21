# Architecture 22 — Database Sharding

> "Sharding is what you do when your data is too big for one machine. It's not a first resort — it's a last resort." — Senior DBA wisdom

---

## At a Glance

| | |
|---|---|
| **Type** | Horizontal database partitioning across multiple nodes |
| **Complexity** | Very High |
| **Best for** | Massive datasets that outgrow a single DB node (100M+ rows, 1TB+) |
| **Avoid when** | Your DB fits on one machine — sharding always comes at a cost |

---

## What Problem Does Sharding Solve?

**The single database bottleneck.** Every database has limits:
- Maximum disk space per machine
- Maximum CPU for query processing
- Maximum RAM for buffer pool (hot data)
- Maximum I/O throughput (reads + writes per second)

When your data grows beyond what ONE machine can handle, you have three escalating options:

```
Option 1: Vertical Scaling (Scale Up)
  → Buy a bigger machine (8 CPU → 64 CPU, 64GB RAM → 1TB RAM)
  → Limit: the biggest machine AWS offers is finite; cost exponential
  → Used by: most apps for years before they need sharding

Option 2: Read Replicas (Scale Reads)
  → Add read-only copies; writes still go to one primary
  → Limit: write throughput still capped by one primary
  → Used by: most apps at medium scale

Option 3: Sharding (Scale Both Reads and Writes)
  → Split data across N machines; each machine owns a subset
  → Each machine handles reads AND writes for its subset
  → No limit: add more shards as you grow
  → Cost: massive complexity
```

---

## Diagram Reference
`./diagram.svg`

---

## Senior Architect Explains It to a Junior

> **Junior:** "Why don't we just get a bigger database server?"
>
> **Senior:** "We did. Then we got the biggest one AWS has. Then we ran out. Vertical scaling hits a physical ceiling. The moment your dataset is larger than what one machine's disk can hold, or your write throughput exceeds what one CPU can process, you MUST split the data. That's sharding."
>
> **Junior:** "Can't we just use read replicas?"
>
> **Senior:** "Read replicas only help reads. Our write rate is 50,000 inserts/second. One primary can only do about 20,000. So 30,000 writes are failing. We need to split writes too. That means sharding."
>
> **Junior:** "How do we decide which data goes to which machine?"
>
> **Senior:** "That's the shard key. Choosing it wrong is the #1 mistake. Pick a key with low cardinality (like country) and one shard gets all the traffic. Pick a key that changes (like status) and you're re-sharding constantly. The ideal shard key: high cardinality, never changes, evenly distributed, and is included in most queries."

---

## Sharding Strategies

### 1. Range-Based Sharding

Split data by value ranges of the shard key.

```
Shard 1: user_id 1 – 10,000,000
Shard 2: user_id 10,000,001 – 20,000,000
Shard 3: user_id 20,000,001 – 30,000,000

Pros: Range queries are fast (all user 1-100 data is on Shard 1)
Cons: Hotspots — new users always go to the last shard; it gets overloaded
Used by: MongoDB (range sharding), MySQL range partitioning
```

### 2. Hash-Based Sharding

Apply a hash function to the shard key; route to shard = hash(key) % N.

```
user_id = 12345 → hash(12345) % 4 = shard 2
user_id = 99999 → hash(99999) % 4 = shard 0

Pros: Even distribution; no hotspots
Cons: Range queries span all shards; adding a shard requires rehashing ALL data
Used by: Early Cassandra, early Redis Cluster
```

### 3. Consistent Hashing (Virtual Nodes)

Map both data and nodes to a hash ring. Data is assigned to the nearest clockwise node.

```
Hash ring (0 to 2^32):
  Node A owns: 0 – 100
  Node B owns: 101 – 200
  Node C owns: 201 – 300

user_id hashes to 150 → goes to Node B

Adding Node D between B and C:
  Node D owns: 201 – 250 (half of C's range)
  Only data in 201-250 moves; everything else stays
  ~25% of data moves vs 75% with simple hash-mod

Pros: Adding/removing nodes moves minimal data
Used by: Amazon DynamoDB, Cassandra, Discord, Akamai CDN
```

### 4. Directory-Based Sharding (Lookup Table)

A central lookup table maps each key to its shard.

```
Lookup table (Redis):
  user_id 1–1M      → shard_1
  user_id 1M–5M     → shard_2
  user_id 5M+       → shard_3

Pros: Flexible; easy to rebalance by updating the table
Cons: Lookup table is a SPOF; adds one extra lookup per query
Used by: Shopify (per-merchant lookup), early Instagram
```

### 5. Geographic Sharding

Route data based on user's physical location.

```
Shard EU:   all users with country IN ('DE','FR','UK','NL'...)
Shard US:   all users with country IN ('US','CA','MX'...)
Shard APAC: all users with country IN ('TH','JP','SG','AU'...)

Pros: Data residency compliance (GDPR); low latency (serve from region)
Cons: Uneven size if US has 10x more users than APAC
Used by: WhatsApp, Meta, Grab
```

---

## Real Code: How Sharding Works in Practice

```typescript
// Shard Router — determines which DB connection to use
class ShardRouter {
  private shards: DatabaseConnection[];

  constructor(shards: DatabaseConnection[]) {
    this.shards = shards;
  }

  // Consistent hash: which shard owns this userId?
  getShardForUser(userId: string): DatabaseConnection {
    const hash = this.fnv32a(userId);
    const shardIndex = hash % this.shards.length;
    return this.shards[shardIndex];
  }

  // For cross-shard queries (expensive — try to avoid)
  async fanOutQuery<T>(query: string, params: unknown[]): Promise<T[]> {
    const results = await Promise.all(
      this.shards.map(shard => shard.query<T>(query, params))
    );
    return results.flat();
  }

  private fnv32a(str: string): number {
    let hash = 2166136261;
    for (let i = 0; i < str.length; i++) {
      hash ^= str.charCodeAt(i);
      hash = (hash * 16777619) >>> 0;
    }
    return hash;
  }
}

// Usage in Repository (Use Case knows nothing about sharding)
class UserRepositorySharded implements IUserRepository {
  constructor(private router: ShardRouter) {}

  async findById(userId: string): Promise<User> {
    const shard = this.router.getShardForUser(userId);  // find the right shard
    const row = await shard.query('SELECT * FROM users WHERE id = $1', [userId]);
    return UserMapper.toDomain(row);
  }

  async findByEmail(email: string): Promise<User | null> {
    // Email is NOT the shard key — must query all shards (scatter-gather)
    const rows = await this.router.fanOutQuery(
      'SELECT * FROM users WHERE email = $1', [email]
    );
    return rows.length > 0 ? UserMapper.toDomain(rows[0]) : null;
  }
}
```

**The Use Case never knows about sharding — it just calls `IUserRepository.findById()`. This is Clean Architecture protecting business logic from infrastructure decisions.**

---

## The Shard Key Problem: Everything Depends on It

```
Bad shard key: "status" (active/inactive)
  → 95% of queries are for "active" users
  → One shard gets 95% of traffic: HOT SPOT

Bad shard key: "created_at" (date range)
  → Today's shard gets ALL new users
  → Old shards are idle: UNEVEN DISTRIBUTION

Bad shard key: "country" (low cardinality)
  → US = 40% of users, all on one shard
  → Thailand = 0.1% of users, one whole shard mostly empty

Good shard key: "user_id" (UUID or auto-increment)
  → High cardinality (billions of unique values)
  → Even distribution with consistent hashing
  → Immutable (user_id never changes)
  → In most queries ("get user by id" is the most common)
```

---

## Hotspot Problem and Solutions

A hotspot = one shard getting disproportionate traffic.

```
Problem: Viral post on social media
  Post ID 12345 gets 10M views/hour → all on one shard → overload

Solution 1: Celebrity shards
  Put viral content on dedicated "hot" shards with more resources

Solution 2: Cell-based caching
  Put a Redis cache in front of the hot shard; only cache misses hit the DB

Solution 3: Time-based TTL
  Viral content is hot for 24h; cache it aggressively during that window

Solution 4: Read replicas per shard
  Add 3-5 read replicas for the overloaded shard; route reads to replicas
```

---

## Cross-Shard Queries: The Biggest Pain

```
Easy (single shard): "Get user 12345"
  hash(12345) → Shard 2 → one query

Hard (cross-shard): "Find all users who signed up this week"
  Must query ALL shards → collect results → merge → sort → paginate
  Called: Scatter-Gather query

Very Hard: "Join orders with users where user.country = 'TH'"
  Orders on shard by order_id; Users on shard by user_id
  No shard has both → must denormalize or use a separate query service

Solution: Design your schema around the shard key
  If you shard by user_id, store user_id IN the orders table
  Then query: orders WHERE user_id = 12345 → same shard as user
```

---

## Resource Consumption

| Resource | Per Shard | Total Impact |
|---|---|---|
| **CPU** | Lower per machine | Distributed; each machine handles its subset |
| **Memory** | Lower per machine | Each shard's buffer pool fits its data subset |
| **Disk** | Lower per machine | Data evenly split across N machines |
| **Network** | Higher | Cross-shard queries add latency |
| **Ops cost** | Very High | N databases to backup, monitor, patch, tune |
| **Developer complexity** | Very High | Every query must consider the shard key |

---

## Benefits

1. **Write scalability** — 10 shards = 10× write throughput (each handles its own writes)
2. **No single bottleneck** — data spread across N machines
3. **Failure isolation** — Shard 3 going down only affects ~10% of users (1 of 10 shards)
4. **Geographic distribution** — shards in different regions = low latency for local users
5. **Theoretically infinite scale** — add more shards as data grows

---

## Problems Solved Best

| Problem | Why Sharding Wins |
|---|---|
| "Our 50TB user table doesn't fit on the biggest RDS instance" | Split into 10 shards × 5TB each |
| "We're doing 80K writes/second but MySQL tops out at 20K" | 4 shards × 20K writes = 80K |
| "GDPR requires EU user data stays in EU" | EU shard in Frankfurt; US shard in Virginia |
| "One bad query from analytics team slows down production" | Separate analytics shard (read replica); isolate blast radius |

---

## Costs / Tradeoffs

1. **Cross-shard queries are expensive** — scatter-gather across all shards
2. **Distributed transactions are hard** — no ACID across shards without 2PC or Saga
3. **Operational nightmare** — N databases to manage, backup, restore
4. **Re-sharding is painful** — changing shard count requires moving data
5. **Schema changes across all shards** — `ALTER TABLE` must run on every shard
6. **Application logic changes** — every query must include shard routing

---

## Big Tech Examples

### Instagram (2012)
- **Problem:** 30TB of photo data, growing 500GB/day; one PostgreSQL couldn't hold it
- **Solution:** Sharded PostgreSQL by user_id using a logical sharding layer called "Django ORM sharding"
- **Shard key:** `user_id` — every query for a user's photos is on the same shard
- **Scale:** 12 shards initially; scaled to hundreds
- **Lesson:** They sharded BEFORE migrating to microservices; monolith + sharding worked at massive scale

### WhatsApp
- **Problem:** 2 billion users, message history; Mnesia (Erlang DB) doesn't scale horizontally
- **Solution:** Sharded by phone number hash; each shard = one Erlang node cluster
- **Geographic:** Shards assigned to regions (EU, US, APAC) for GDPR compliance
- **Lesson:** Simple hash-based sharding + geographic routing covers most cases

### Shopify
- **Problem:** 2M+ merchants, each with their own store data; one MySQL was a bottleneck
- **Solution:** Sharded MySQL by `shop_id`; each shard = one MySQL pod
- **Tool:** Internal routing layer maps `shop_id → shard_id → MySQL host`
- **Guarantee:** One merchant's traffic NEVER affects another merchant's shard
- **Scale:** Handles 30TB of data per minute across all shards during peak

### Discord
- **Problem:** Trillions of messages in Cassandra; hot partitions causing p99 latency spikes
- **Solution:** Migrated to ScyllaDB (Cassandra-compatible, written in C++); consistent hashing maintained
- **Hot partition fix:** Added "bucket" layer — channel_id + time_bucket as shard key; spreads popular channels across shards
- **Result:** p99 read latency dropped from 40–125ms to 15ms

### Pantip (Thailand's Largest Community Site)
- **Context:** Pantip is Thailand's oldest and largest internet community — like Reddit for Thai users
- **Problem:** Decades of forum posts, comments, and user data; single MySQL becoming a bottleneck
- **Solution:** Vertical partitioning first (separate tables by feature area), then read replicas for popular boards
- **Shard strategy:** Board-based sharding — each major board category routes to its own MySQL cluster
- **Lesson:** For content communities, topic/category sharding maps naturally to the data access pattern (users browse by category, not across all content)
- **Thai context:** During major events (elections, floods, disasters), specific boards spike 100x while others are quiet — category sharding isolates these spikes

### Amazon DynamoDB (the gold standard)
- **Problem:** Amazon needed a key-value store that could handle any traffic at any scale
- **Solution:** Designed consistent hashing into DynamoDB's DNA; every table is automatically sharded
- **Virtual nodes:** Each physical node manages multiple virtual nodes; rebalancing is automatic
- **Result:** Scales from 1 req/second to millions without application changes
- **Lesson:** The best sharding is invisible — abstract it into the DB layer itself

---

## Database Sharding vs Alternatives

| Approach | When to Use | Limit |
|---|---|---|
| **Single DB** | <500GB, <10K writes/sec | Machine capacity |
| **Read Replicas** | Read-heavy, write manageable | Write throughput still bottlenecked |
| **Vertical Scale** | Moderate growth, simple ops | Largest machine in cloud |
| **Horizontal Sharding** | 1TB+, 50K+ writes/sec, compliance | Cross-shard query complexity |
| **NewSQL (CockroachDB)** | Want SQL + auto-sharding | Slightly higher latency |
| **Distributed NoSQL** | Key-value/document scale | Limited query flexibility |

---

## In Clean Architecture Terms

Sharding is entirely in the **Infrastructure / Frameworks & Drivers layer**. The Use Case sees only `IUserRepository`. The ShardRouter is an implementation detail of `UserRepositorySharded`. Business logic never changes when you add a new shard.

```
Use Case: findUserById(userId)
  → calls: IUserRepository.findById(userId)         [domain port]
    → ShardRouter.getShardForUser(userId)            [infrastructure]
      → PostgreSQL Shard 3: SELECT * FROM users...   [infrastructure]
```

---

## Key Takeaway

> Sharding is the last resort when vertical scaling and read replicas are exhausted. The shard key decision is permanent and wrong choices are catastrophically expensive to fix. Design around the shard key from day one. Keep sharding logic in the infrastructure layer (adapter) — your use cases should never know your database is sharded. Pantip's category-based approach, Instagram's user_id approach, and Shopify's shop_id approach all share the same principle: choose a key that matches your most common query pattern and distributes load evenly.
