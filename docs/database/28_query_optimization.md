# Database Query Optimization

> "The fastest query is the one that never hits the database." - Cache-first mindset
> "The second fastest query is the one that reads the fewest rows." - Index mindset

---

## At a Glance

| | |
|---|---|
| **Goal** | Reduce query time from seconds to milliseconds |
| **Primary tools** | EXPLAIN ANALYZE, indexes, query rewriting |
| **Process** | Measure first, then optimize - never assume |

---

## Senior to Junior: The Optimization Process

> "Never optimize without measurement. I've seen juniors spend three days rewriting a query to be 'more efficient', only to discover it runs twice a week on a 100-row table. That's three days wasted. The first step is always: identify which queries are actually slow. Then measure. Then fix. In that order."

**Optimization workflow:**

```
1. IDENTIFY: Which queries are slow?
 -> pg_stat_statements (PostgreSQL): shows all queries ranked by total time

2. MEASURE: How slow exactly?
 -> EXPLAIN ANALYZE SELECT ...
 -> Look for: Seq Scan on large tables, high row counts, slow execution time

3. UNDERSTAND: Why is it slow?
 -> Missing index? Bad query structure? Wrong join order? N+1?

4. FIX: Apply the specific fix
 -> Add index / rewrite query / add cache / denormalize

5. VERIFY: Is it actually faster?
 -> EXPLAIN ANALYZE again
 -> Production metrics (p50, p99 latency)
```

---

## Finding Slow Queries

```sql
-- PostgreSQL: enable pg_stat_statements extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find the slowest queries by total execution time:
SELECT
  round(total_exec_time::numeric, 2) AS total_ms,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS percent,
  LEFT(query, 100) AS query_preview
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Find queries doing the most sequential scans:
SELECT schemaname, tablename, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
ORDER BY seq_tup_read DESC
LIMIT 10;
-- High seq_tup_read = reading many rows without index = likely needs index
```

---

## Query Optimization Patterns

### Pattern 1: Add the Missing Index

```sql
-- Slow: full scan
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 12345;
-- -> Seq Scan, 2.3 seconds

-- Fix:
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
-- CONCURRENTLY: builds index without locking the table (safe for production)

-- Fast: index scan
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 12345;
-- -> Index Scan, 0.18ms
```

### Pattern 2: Fix the N+1 Query

```sql
-- N+1 anti-pattern (100 separate queries):
SELECT id FROM orders LIMIT 100;  -- 1 query -> 100 order IDs
-- For each order:
SELECT * FROM users WHERE id = ?;  -- 100 queries!

-- Fix: JOIN in single query
SELECT o.*, u.name, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
LIMIT 100;
-- 1 query, same result

-- OR: IN clause (better when you already have IDs)
SELECT * FROM users WHERE id IN (1, 2, 3, ..., 100);
-- 1 query instead of 100
```

### Pattern 3: Select Only What You Need

```sql
-- Bad: SELECT * fetches all columns
SELECT * FROM products;
-- Fetches: id, name, description (TEXT, 10KB), metadata (JSONB, 5KB),
--          images_json (20KB), ...
-- Total: maybe 40KB per row x 1000 rows = 40MB transferred

-- Good: SELECT only columns you need
SELECT id, name, price_cents, thumbnail_url FROM products;
-- Fetches: maybe 200 bytes per row x 1000 rows = 200KB
-- 200x less data transferred and processed

-- Very good: covering index means no table access at all
CREATE INDEX idx_products_list ON products(category_id)
INCLUDE (id, name, price_cents, thumbnail_url);

SELECT id, name, price_cents, thumbnail_url
FROM products WHERE category_id = 5;
-- Index Only Scan: data comes from index, table never touched
```

### Pattern 4: Use LIMIT and OFFSET Correctly

```sql
-- Bad: OFFSET 1000000 (very slow for deep pagination)
SELECT * FROM orders ORDER BY created_at DESC OFFSET 1000000 LIMIT 20;
-- Database must count 1M rows before returning 20
-- Gets slower as OFFSET grows: O(n)

-- Good: Cursor-based pagination (keyset pagination)
SELECT * FROM orders
WHERE created_at < '2024-01-15 10:30:00'  -- last item from previous page
ORDER BY created_at DESC
LIMIT 20;
-- Uses index on created_at -> O(log n) regardless of page depth
-- This is how Twitter, Instagram, and Facebook paginate feeds
```

### Pattern 5: Rewrite Subqueries as JOINs

```sql
-- Slow: correlated subquery (runs once per row)
SELECT *
FROM orders o
WHERE user_id IN (
  SELECT id FROM users WHERE country = 'TH'
);
-- Subquery may run once per order row

-- Fast: JOIN (usually faster, optimizer can choose best strategy)
SELECT o.*
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.country = 'TH';
```

### Pattern 6: Avoid Functions on Indexed Columns

```sql
-- Bad: function on column prevents index usage
SELECT * FROM users WHERE UPPER(email) = 'SUPHAKIN@EXAMPLE.COM';
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15';
-- -> Seq Scan: can't use index when function applied to column

-- Fix 1: Store in normalized form
-- Always store email as lowercase on insert -> index on lowercase column works

-- Fix 2: Functional index
CREATE INDEX idx_users_email_upper ON users(UPPER(email));
SELECT * FROM users WHERE UPPER(email) = 'SUPHAKIN@EXAMPLE.COM';
-- -> Index Scan: now uses functional index

-- Fix 3: Rewrite to avoid function on column
SELECT * FROM orders
WHERE created_at >= '2024-01-15 00:00:00'
  AND created_at < '2024-01-16 00:00:00';
-- -> Index Scan on created_at (range query, no function)
```

### Pattern 7: Batch Operations

```sql
-- Bad: 1000 individual INSERTs
for row in data:
  INSERT INTO logs (user_id, event, timestamp) VALUES (?, ?, ?);
-- 1000 round-trips, 1000 transaction commits

-- Good: batch INSERT (1 round-trip)
INSERT INTO logs (user_id, event, timestamp) VALUES
  (1, 'login', '2024-01-15'),
  (2, 'purchase', '2024-01-15'),
  (3, 'logout', '2024-01-15'),
  ...  -- 1000 rows in one statement;
-- 100-1000x faster

-- Good: COPY for bulk data loading
COPY logs FROM '/tmp/data.csv' WITH (FORMAT CSV, HEADER true);
-- Fastest possible bulk insert: bypasses row-by-row processing
-- Used by: data imports, ETL pipelines, initial data loads
```

### Pattern 8: Partial Indexes for Common Filters

```sql
-- Full index on status column (every row indexed):
CREATE INDEX idx_orders_status ON orders(status);
-- Size: all 50M orders

-- Partial index (only pending orders):
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';
-- Size: only 10K pending orders
-- Query: WHERE status = 'pending' ORDER BY created_at
-- 5000x smaller index -> faster scan, less memory

-- When to use partial indexes:
-- Active records only (soft-deleted rows excluded)
-- Specific status values (pending, processing, failed)
-- Date ranges (last 30 days only)
CREATE INDEX idx_recent_orders ON orders(created_at)
WHERE created_at > NOW() - INTERVAL '30 days';
```

---

## Query Optimization for Common Operations

### Pagination

```sql
-- [X] SLOW: traditional LIMIT/OFFSET degrades with depth
-- Page 1: OFFSET 0 -> fast (reads 20 rows)
-- Page 1000: OFFSET 19980 -> reads 20,000 rows, returns 20

-- [OK] FAST: Keyset pagination
-- First page:
SELECT id, title, created_at FROM posts ORDER BY created_at DESC LIMIT 20;

-- Next page (pass last item's created_at and id as cursor):
SELECT id, title, created_at FROM posts
WHERE (created_at, id) < ('2024-01-15 10:30:00', 12345)  -- cursor
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Uses composite index on (created_at DESC, id DESC)
-- Always O(log n) regardless of page number
```

### Counting

```sql
-- [X] SLOW: COUNT(*) on large tables (full scan)
SELECT COUNT(*) FROM orders;  -- scans all 50M rows -> slow

-- [OK] FAST: Use estimates for large counts
SELECT reltuples::BIGINT AS estimate FROM pg_class WHERE relname = 'orders';
-- PostgreSQL keeps approximate row count -> instant, not exact

-- [OK] FAST: Maintain a counter table
CREATE TABLE counters (
  name VARCHAR(50) PRIMARY KEY,
  value BIGINT NOT NULL DEFAULT 0
);
-- Update counter on each INSERT/DELETE (application-level)
INSERT INTO counters (name, value) ON CONFLICT (name) DO UPDATE SET value = counters.value + 1;
-- Read counter: SELECT value FROM counters WHERE name = 'orders_total';
-- Twitter, Reddit, Stack Overflow use counter tables
```

### Aggregations at Scale

```sql
-- [X] SLOW: Real-time aggregation over 50M rows
SELECT user_id, SUM(total_cents) AS lifetime_value
FROM orders
GROUP BY user_id
ORDER BY lifetime_value DESC
LIMIT 100;  -- scan 50M rows every time

-- [OK] FAST: Pre-computed table (materialized view or explicit table)
-- Option 1: PostgreSQL Materialized View
CREATE MATERIALIZED VIEW user_lifetime_value AS
SELECT user_id, SUM(total_cents) AS lifetime_value
FROM orders
GROUP BY user_id;

CREATE INDEX ON user_lifetime_value(lifetime_value DESC);
REFRESH MATERIALIZED VIEW CONCURRENTLY user_lifetime_value;  -- refresh periodically

-- Query: instant (reads pre-computed table)
SELECT user_id, lifetime_value FROM user_lifetime_value ORDER BY lifetime_value DESC LIMIT 100;
```

---

## The Query Cache Pattern

```typescript
// Application-level caching (most effective for read-heavy data)
class ProductRepository implements IProductRepository {
  constructor(
    private db: DatabaseConnection,
    private cache: ICacheService,  // Redis
  ) {}

  async findById(id: string): Promise<Product> {
    // L1: Check Redis cache (sub-millisecond)
    const cached = await this.cache.get(`product:${id}`);
    if (cached) return JSON.parse(cached) as Product;

    // L2: Query database
    const row = await this.db.query(
      'SELECT id, name, price_cents, description FROM products WHERE id = $1', [id]
    );
    if (!row) return null;

    const product = ProductMapper.toDomain(row);

    // Cache for 1 hour (product data changes infrequently)
    await this.cache.setex(`product:${id}`, 3600, JSON.stringify(product));

    return product;
  }

  async invalidate(id: string): Promise<void> {
    await this.cache.del(`product:${id}`);
    // Call this whenever a product is updated
  }
}
```

---

## EXPLAIN ANALYZE - Reading the Output

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, u.name, o.total_cents
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
ORDER BY o.created_at DESC
LIMIT 20;

-- Sample output:
Limit  (cost=0.99..153.04 rows=20 width=24) (actual time=0.234..2.341 rows=20 loops=1)
 -> Nested Loop  (cost=0.99..40234.56 rows=5249 width=24) (actual time=0.229..2.334 rows=20 loops=1)
 -> Index Scan Backward using idx_orders_status_created on orders o
              (cost=0.56..35234.45 rows=5249 width=16)
              (actual time=0.215..0.845 rows=20 loops=1)
              Index Cond: ((status)::text = 'pending')
 -> Index Scan using users_pkey on users u
              (cost=0.43..0.95 rows=1 width=16)
              (actual time=0.073..0.073 rows=1 loops=20)
              Index Cond: (id = o.user_id)
  Buffers: shared hit=66
Planning Time: 1.234 ms
Execution Time: 2.456 ms <- FAST!

-- What this tells us:
-- [OK] Using Index Scan Backward (our index + backward order = no sort needed)
-- [OK] Nested Loop (good for small result sets from index)
-- [OK] Users table: index scan by primary key (fast)
-- [OK] Buffers: shared hit=66 (all from cache, no disk reads)
-- [OK] 2.456ms total (excellent)

-- If you saw Seq Scan instead of Index Scan -> missing index
-- If you saw Sort -> add ORDER BY to your index
-- If you saw Hash Join instead of Nested Loop -> larger result set (may be OK)
```

---

## In Clean Architecture Terms

```
Query optimization = Infrastructure layer concern

IOrderRepository.findPendingOrders(limit: number) = port (interface)
  Defined in Use Case layer
  Has NO opinion about:
    - Which indexes exist
    - How the SQL query is written
    - Whether there's a cache

OrderRepositoryPostgres.findPendingOrders(limit: number) = adapter
  Lives in Infrastructure layer
  Makes ALL the optimization decisions:
    - Uses idx_orders_status_created index
    - Uses covering index to avoid table access
    - Checks Redis cache first
    - Uses keyset pagination

Business logic: "I need the 20 most recent pending orders" = Use Case concern
How to get them efficiently = Infrastructure concern

Changing query optimization strategy -> change the adapter only
Business logic never changes
= Clean Architecture in practice
```
