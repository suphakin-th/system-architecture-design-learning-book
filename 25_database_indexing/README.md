# Database Indexing — Complete Guide

> "An index is a promise to the database engine: 'I will pay the cost of maintaining this structure so you don't have to scan the table every time.'" — Database internals

---

## At a Glance

| | |
|---|---|
| **What it solves** | Slow queries due to full table scans |
| **Trade-off** | Faster reads ↔ Slower writes, more storage |
| **Most important skill** | Reading EXPLAIN plans to know IF an index is being used |

---

## Senior to Junior: Why Indexes Matter

> "Your app works perfectly with 100 rows in the database. You launch, get 1M users, and suddenly everything is slow. You look at the database: one query is doing a full table scan on 50M rows. That's your missing index. The query is functionally correct — it returns the right data. It's just doing it the hard way: checking every single row instead of jumping directly to what it needs. One index, added in 30 seconds, fixes it completely."

```sql
-- Without index (full table scan):
SELECT * FROM orders WHERE user_id = 12345;
-- Database engine: "I need to check all 50M rows to find user 12345's orders"
-- Time: O(n) — proportional to table size
-- 1M rows: 50ms. 50M rows: 2500ms. 500M rows: 25 seconds.

-- With index on user_id:
CREATE INDEX idx_orders_user_id ON orders(user_id);
-- Database engine: "I'll use the index to jump directly to user 12345's rows"
-- Time: O(log n) — proportional to LOG of table size
-- 1M rows: 0.5ms. 50M rows: 0.6ms. 500M rows: 0.7ms.
-- Log(n) barely grows as n grows — this is the power of indexes
```

---

## B-Tree Indexes — The Default (PostgreSQL, MySQL InnoDB)

**How it works:**

```
B-Tree for user_id column:

                   [500]
                  /     \
          [200]             [800]
         /     \           /     \
    [100]      [300]   [600]    [900]
   /    \      /   \    /  \    /   \
 [50] [150] [250] [350] [550][650] [850][950]
  ↓     ↓     ↓     ↓     ↓    ↓     ↓    ↓
 rows  rows  rows  rows  rows  rows  rows rows

Looking for user_id = 650:
  Check root: 650 > 500? yes → go right
  Check 800: 650 < 800? yes → go left
  Check 600: 650 > 600? yes → go right
  Found 650 → jump to data rows

Total comparisons: 4 (for 950 rows)
log₂(950) ≈ 10 comparisons for ANY row in 950 row table
```

**B-Tree supports:**
- Exact match: `WHERE id = 123`
- Range queries: `WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'`
- Starts-with: `WHERE email LIKE 'suphakin%'` (NOT `WHERE email LIKE '%@gmail.com'`)
- ORDER BY (if sorted same as index)

**B-Tree does NOT support:**
- Contains: `WHERE email LIKE '%gmail%'` → full scan
- Case-insensitive by default (use functional index)

---

## LSM Tree Indexes — Write-Optimized (Cassandra, RocksDB, LevelDB)

**The problem with B-Trees at high write volume:**

```
B-Tree write problem:
  Insert row with user_id = 123 → find correct leaf node → insert in place
  If the page is full → SPLIT: rearrange two pages
  Split might cascade up the tree
  Random I/O: the leaf page might be anywhere on disk
  High write load: 100K inserts/sec → 100K random I/O → disk can't keep up

LSM Tree solution: convert random writes to sequential writes
  All writes go to: in-memory buffer (MemTable)
  When buffer full: flush as sorted SSTable (Sequential String Table) to disk
  Sequential write: 10-100× faster than random write on SSD/HDD

Read: check MemTable → L0 SSTables → L1 SSTables → ... (most recent first)
Compaction: background process merges and sorts SSTables
```

**LSM Tree trade-off:**
```
B-Tree:   Reads fast (one disk seek), Writes slow (random I/O)
LSM Tree: Reads slower (check multiple levels), Writes very fast (sequential)

Use B-Tree (PostgreSQL, MySQL): OLTP, read-heavy, complex queries
Use LSM Tree (Cassandra, RocksDB): high write throughput, simple lookups
```

---

## Index Types

### Single Column Index
```sql
CREATE INDEX idx_users_email ON users(email);
-- Used for: WHERE email = 'x' · WHERE email LIKE 'x%' · ORDER BY email
```

### Composite Index (Multi-Column)
```sql
-- CRITICAL: Order matters! The leftmost columns must be in the WHERE clause
CREATE INDEX idx_orders_user_status ON orders(user_id, status, created_at);

-- ✅ Uses the index (leftmost columns present):
SELECT * FROM orders WHERE user_id = 1 AND status = 'pending';
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at;

-- ❌ Does NOT use the index (missing leftmost column):
SELECT * FROM orders WHERE status = 'pending';  -- status is not the first column
SELECT * FROM orders WHERE created_at > '2024-01-01';  -- created_at is third

-- Rule: index is used from left to right, stops at first missing column
```

### Partial Index
```sql
-- Only index rows matching a condition (smaller, faster)
CREATE INDEX idx_orders_pending ON orders(user_id)
WHERE status = 'pending';

-- Perfect for: "find all pending orders for user X" (only pending rows indexed)
-- Much smaller than full index (most orders are completed, not pending)
```

### Unique Index
```sql
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
-- Enforces: no two users can have the same email
-- Also creates an index (fast lookup by email)
-- Same as: UNIQUE constraint in CREATE TABLE
```

### Expression/Functional Index
```sql
-- Index on the RESULT of a function
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Now this query uses the index:
SELECT * FROM users WHERE LOWER(email) = 'suphakin@example.com';
-- Without this index: LOWER(email) function prevents index usage
```

### Full-Text Index
```sql
-- PostgreSQL full-text search index
CREATE INDEX idx_posts_content_fts ON posts USING GIN(to_tsvector('english', content));

-- Query:
SELECT * FROM posts WHERE to_tsvector('english', content) @@ plainto_tsquery('clean architecture');
```

### GiST / GIN — Spatial and Full-Text
```sql
-- GiST: for geometric, range, and nearest-neighbor queries
CREATE INDEX idx_restaurants_location ON restaurants USING GIST(location);

-- GIN: for full-text search and JSONB containment
CREATE INDEX idx_products_metadata ON products USING GIN(metadata);
SELECT * FROM products WHERE metadata @> '{"brand": "Apple"}';
```

---

## Reading EXPLAIN Plans — Your Most Important Skill

```sql
-- PostgreSQL EXPLAIN ANALYZE
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 12345;

-- Example output WITHOUT index (bad):
Seq Scan on orders  (cost=0.00..45231.00 rows=23 width=156) (actual time=0.123..2341.234 rows=23 loops=1)
  Filter: (user_id = 12345)
  Rows Removed by Filter: 5000000
Planning Time: 0.234 ms
Execution Time: 2341.456 ms  ← 2.3 SECONDS = BAD

-- Key terms to understand:
-- Seq Scan = full table scan = NO INDEX USED (usually bad for large tables)
-- Index Scan = using the B-tree index = GOOD
-- Bitmap Index Scan = multiple indexes combined = GOOD
-- Rows Removed by Filter = how many rows were checked and thrown away
-- Execution Time = actual time in milliseconds

-- Example output WITH index (good):
Index Scan using idx_orders_user_id on orders  (cost=0.56..12.23 rows=23 width=156)
  (actual time=0.045..0.123 rows=23 loops=1)
  Index Cond: (user_id = 12345)
Planning Time: 0.456 ms
Execution Time: 0.178 ms  ← 0.2ms = 13,000× faster!
```

**What to look for in EXPLAIN:**

```
RED FLAGS (investigate these):
  Seq Scan on large table  → might need an index
  Rows Removed by Filter: 5000000  → checking many rows to find few
  Execution Time: > 100ms for simple queries

GREEN FLAGS (working well):
  Index Scan  → using B-tree index
  Bitmap Index Scan  → using index efficiently
  Index Only Scan  → data from index itself, no table access needed!
  Execution Time: < 10ms

OPERATIONS (cost breakdown):
  Nested Loop  → good for small result sets
  Hash Join  → good for larger joins
  Merge Join  → good for pre-sorted data
  Sort  → sort needed (check if index can eliminate it)
```

---

## Index Design Patterns

### The Cover Index (Index-Only Scan)

```sql
-- If the index contains ALL columns needed by the query,
-- PostgreSQL never touches the main table at all

-- Bad: index only on user_id, but SELECT needs status and total
CREATE INDEX idx_orders_user ON orders(user_id);
SELECT status, total FROM orders WHERE user_id = 123;
-- → Index Scan (finds rows) + heap fetch (reads table for status, total)

-- Good: covering index includes queried columns
CREATE INDEX idx_orders_user_covering ON orders(user_id) INCLUDE (status, total);
SELECT status, total FROM orders WHERE user_id = 123;
-- → Index Only Scan (never touches table = fastest possible)
```

### The Cardinality Rule

```sql
-- High cardinality column = many distinct values → good index candidate
-- Low cardinality column = few distinct values → poor index candidate

-- HIGH cardinality (good for index):
  user_id: 10M distinct values in 10M rows → index is very selective
  email: unique per user → perfect
  order_id: unique → perfect

-- LOW cardinality (poor index candidate):
  status: ('pending', 'completed', 'cancelled') → 3 distinct values
  country: 10-200 distinct values → might be useful if specific country is rare
  boolean: 2 values → useless as a standalone index

-- Combining low+high cardinality:
  CREATE INDEX ON orders(status, user_id);  -- status first filters, user_id narrows
  -- Useful if: "find all pending orders for user 123" (common query)
```

### Index for Sorting

```sql
-- If ORDER BY matches index order, no sort operation needed
CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);

-- Now this query needs NO SORT:
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20;
-- Without index: sorts 50M rows to get top 20 → slow
-- With index: walks index backward, takes first 20 → instant
```

---

## When NOT to Index

```sql
-- Indexes have costs: every INSERT/UPDATE/DELETE must also update all indexes

-- Table with 5 indexes:
  INSERT → write to table + update 5 indexes = 6 I/O operations
  UPDATE → modify table + update changed columns' indexes

-- Don't index:
-- 1. Small tables (< 10K rows): full scan is faster than index + random heap fetch
-- 2. Columns rarely used in WHERE/JOIN/ORDER BY
-- 3. Columns with very low cardinality (boolean, status with 2 values)
-- 4. Tables with very high write rate (index maintenance overhead)
-- 5. Columns that are already covered by another index's leftmost prefix

-- The index maintenance trade-off:
-- Reads: faster (O(log n) vs O(n))
-- Writes: slower (update N indexes)
-- Storage: more disk space
-- For read-heavy apps (web: 95% reads): index aggressively
-- For write-heavy apps (IoT sensors: 95% writes): index minimally
```

---

## Real-World Index Examples

### E-Commerce Orders Table

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  status VARCHAR(20) NOT NULL,
  total_cents INT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index for user's order history (most common query)
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);
-- Query: SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 20;

-- Index for admin order management
CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);
-- Query: SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC;

-- Partial index for pending orders only (smaller, faster)
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';
-- Query: SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at;

-- Covering index for order list page (never touches table)
CREATE INDEX idx_orders_user_list ON orders(user_id, created_at DESC)
INCLUDE (status, total_cents);
-- Query: SELECT id, status, total_cents FROM orders WHERE user_id = ? ORDER BY created_at DESC;
-- → Index Only Scan: gets all needed data from index alone
```

---

## In Clean Architecture Terms

```
Indexes = Infrastructure layer concern (completely)

The Use Case calls:
  IOrderRepository.findByUserId(userId, { sortBy: 'created_at', order: 'desc', limit: 20 })

The Repository adapter decides HOW to execute this:
  OrderRepositoryPostgres:
    SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 20
    → The INDEX exists or not — this is infrastructure
    → The use case never knows

The business rule "get latest 20 orders" lives in Use Case layer
The performance of that query lives in Infrastructure layer

This means: you can optimize indexes WITHOUT touching Use Case or Entity code
= Clean Architecture's isolation in practice
```
