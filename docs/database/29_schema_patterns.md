# Database Schema Patterns

> "Your schema is your API to the database. A well-designed schema makes impossible states unrepresentable. A poorly designed schema makes bugs inevitable." - Senior DBA

---

## At a Glance

| | |
|---|---|
| **Goal** | Design schemas that prevent invalid data and support query needs |
| **Key patterns** | Polymorphism, hierarchy, temporal, audit, multi-tenancy |
| **Golden rule** | Schema should make invalid states impossible |

---

## Pattern 1: Polymorphic Associations (Handle Multiple Entity Types)

**Problem:** A comment can belong to a Post OR a Video OR a Photo. How do you store this?

```sql
-- [X] Naive approach: nullable foreign keys (bad design)
CREATE TABLE comments (
  id BIGINT PRIMARY KEY,
  body TEXT NOT NULL,
  post_id BIGINT REFERENCES posts(id),   -- NULL if on video
  video_id BIGINT REFERENCES videos(id), -- NULL if on post
  photo_id BIGINT REFERENCES photos(id)  -- NULL if on post/video
);
-- Problem: lots of NULLs, no way to enforce "exactly one parent"
-- Can't have FK constraint when the target varies

-- [OK] Option A: Separate join tables (cleanest)
CREATE TABLE post_comments (
  comment_id BIGINT REFERENCES comments(id),
  post_id    BIGINT REFERENCES posts(id),
  PRIMARY KEY (comment_id, post_id)
);
CREATE TABLE video_comments (
  comment_id BIGINT REFERENCES comments(id),
  video_id   BIGINT REFERENCES videos(id),
  PRIMARY KEY (comment_id, video_id)
);
-- Pros: proper FK constraints, clear ownership
-- Cons: more tables, need UNION to get all comments

-- [OK] Option B: STI (Single Table Inheritance) with type discriminator
CREATE TABLE commentables (
  id          BIGINT PRIMARY KEY,
  entity_type VARCHAR(20) NOT NULL,  -- 'post', 'video', 'photo'
  entity_id   BIGINT NOT NULL,
  UNIQUE (entity_type, entity_id, comment_id)
);
CREATE TABLE comments (
  id             BIGINT PRIMARY KEY,
  commentable_id BIGINT REFERENCES commentables(id),
  body           TEXT NOT NULL
);
-- Flexible, works with any entity type, no schema change needed for new types
```

---

## Pattern 2: Hierarchical Data (Trees and Graphs in SQL)

**Problem:** Categories with subcategories: Electronics -> Phones -> Smartphones

### Adjacency List (Simple, slow for deep trees)
```sql
CREATE TABLE categories (
  id        BIGINT PRIMARY KEY,
  name      VARCHAR(100) NOT NULL,
  parent_id BIGINT REFERENCES categories(id)  -- NULL = root
);

-- Find all ancestors of category 5:
-- Requires recursive query (CTE)
WITH RECURSIVE ancestors AS (
  SELECT id, name, parent_id
  FROM categories WHERE id = 5

  UNION ALL

  SELECT c.id, c.name, c.parent_id
  FROM categories c
  JOIN ancestors a ON c.id = a.parent_id
)
SELECT * FROM ancestors;
-- Works but is slow for deep trees (one query per level)
```

### Materialized Path (Fast reads, moderate writes)
```sql
CREATE TABLE categories (
  id   BIGINT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  path VARCHAR(500) NOT NULL  -- e.g., '/1/3/7/12/' for depth-4 category
);

-- All children of category 7:
SELECT * FROM categories WHERE path LIKE '/1/3/7/%';
-- One query, index on path, O(log n)

-- Insert new category as child of 7:
INSERT INTO categories (id, name, path)
VALUES (99, 'New Sub', '/1/3/7/99/');
-- Update path if parent changes -> must update all descendants

-- Used by: Django's django-treebeard, many CMS systems
```

### Nested Sets (Fastest reads, expensive writes)
```sql
CREATE TABLE categories (
  id   BIGINT PRIMARY KEY,
  name VARCHAR(100),
  lft  INT NOT NULL,  -- left value
  rgt  INT NOT NULL   -- right value
);

-- A parent's lft and rgt bracket all its descendants:
-- Electronics: lft=1, rgt=20
--   Phones: lft=2, rgt=11
--     Smartphones: lft=3, rgt=8

-- All descendants of 'Phones' (lft=2, rgt=11):
SELECT * FROM categories WHERE lft > 2 AND rgt < 11;
-- One query, no recursion, O(log n) with index

-- Insert is expensive: must update all lft/rgt values to the right of insert point
-- Good when: read-heavy category tree that rarely changes
```

### Closure Table (Best balance - recommended)
```sql
CREATE TABLE categories (id BIGINT PRIMARY KEY, name VARCHAR(100));
CREATE TABLE category_paths (
  ancestor_id   BIGINT REFERENCES categories(id),
  descendant_id BIGINT REFERENCES categories(id),
  depth         INT NOT NULL,
  PRIMARY KEY (ancestor_id, descendant_id)
);
-- Stores ALL ancestor-descendant pairs, not just direct parent-child

-- Electronics is ancestor of Smartphones (depth=2)
-- Insert (Electronics.id, Smartphones.id, 2)

-- All descendants of Electronics:
SELECT c.* FROM categories c
JOIN category_paths cp ON c.id = cp.descendant_id
WHERE cp.ancestor_id = electronics_id;
-- One query, indexed, fast

-- Insert new category:
-- Insert all ancestor-descendant pairs from new category to all its ancestors
-- More writes than adjacency list, but reads are very fast
-- Used by: Rails acts_as_tree, SQL Server hierarchyid-alternative pattern
```

---

## Pattern 3: Temporal Data (Time-Versioned Records)

**Problem:** Product prices change. You need to see what the price WAS on any date.

### Bi-Temporal Table
```sql
CREATE TABLE product_prices (
  product_id   BIGINT NOT NULL REFERENCES products(id),
  price_cents  INT NOT NULL,
  valid_from   DATE NOT NULL,   -- when this price starts
  valid_to     DATE,            -- NULL = currently active
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  PRIMARY KEY (product_id, valid_from)
);

-- Insert new price (and close old one):
UPDATE product_prices
SET valid_to = CURRENT_DATE
WHERE product_id = 1 AND valid_to IS NULL;

INSERT INTO product_prices (product_id, price_cents, valid_from, valid_to)
VALUES (1, 4990, CURRENT_DATE, NULL);  -- new current price

-- Query: what was the price on January 15?
SELECT price_cents FROM product_prices
WHERE product_id = 1
  AND valid_from <= '2024-01-15'
  AND (valid_to IS NULL OR valid_to > '2024-01-15');

-- Current price:
SELECT price_cents FROM product_prices
WHERE product_id = 1 AND valid_to IS NULL;
```

### PostgreSQL Temporal (using ranges)
```sql
-- Using PostgreSQL's DATERANGE type
CREATE TABLE product_prices (
  product_id BIGINT NOT NULL REFERENCES products(id),
  price_cents INT NOT NULL,
  valid_during DATERANGE NOT NULL,  -- '[2024-01-01,2024-06-01)' = Jan to June
  EXCLUDE USING GIST (product_id WITH =, valid_during WITH &&)  -- no overlaps
);

-- Exclusion constraint prevents overlapping price periods automatically!
-- Find price on a date:
SELECT price_cents FROM product_prices
WHERE product_id = 1 AND valid_during @> '2024-01-15'::date;
```

---

## Pattern 4: Audit Trail (Who Changed What When)

```sql
-- Option A: Audit columns on every table (simple)
ALTER TABLE orders ADD COLUMN created_by BIGINT REFERENCES users(id);
ALTER TABLE orders ADD COLUMN updated_by BIGINT REFERENCES users(id);
ALTER TABLE orders ADD COLUMN created_at TIMESTAMPTZ DEFAULT NOW();
ALTER TABLE orders ADD COLUMN updated_at TIMESTAMPTZ DEFAULT NOW();
-- Limitation: only tracks last update, not full history

-- Option B: Shadow/Audit table (full history)
CREATE TABLE orders_audit (
  audit_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id    BIGINT NOT NULL,
  operation   CHAR(1) NOT NULL CHECK (operation IN ('I', 'U', 'D')),  -- Insert/Update/Delete
  changed_by  BIGINT,
  changed_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  old_data    JSONB,  -- previous values
  new_data    JSONB   -- new values
);

-- Trigger to populate audit table automatically:
CREATE OR REPLACE FUNCTION audit_orders()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'DELETE' THEN
    INSERT INTO orders_audit (order_id, operation, old_data)
    VALUES (OLD.id, 'D', row_to_json(OLD));
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO orders_audit (order_id, operation, old_data, new_data)
    VALUES (NEW.id, 'U', row_to_json(OLD), row_to_json(NEW));
  ELSIF TG_OP = 'INSERT' THEN
    INSERT INTO orders_audit (order_id, operation, new_data)
    VALUES (NEW.id, 'I', row_to_json(NEW));
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION audit_orders();
```

---

## Pattern 5: Multi-Tenancy (One DB, Many Customers)

**Three approaches to multi-tenant data isolation:**

### Shared Schema (most efficient, hardest to isolate)
```sql
-- Every table has tenant_id
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  tenant_id BIGINT NOT NULL REFERENCES tenants(id),
  user_id BIGINT NOT NULL,
  total_cents INT NOT NULL
);

-- Row-level security (PostgreSQL)
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

-- Application sets tenant context:
SET LOCAL "app.current_tenant_id" = '123';
SELECT * FROM orders;  -- automatically filtered to tenant 123

-- Used by: Shopify (early), most SaaS applications
-- Risk: one bug = one tenant sees another's data
```

### Separate Schema (medium isolation)
```sql
-- Each tenant gets their own PostgreSQL schema
-- tenant_123.orders, tenant_456.orders

CREATE SCHEMA tenant_123;
CREATE TABLE tenant_123.orders (...);  -- isolated

-- Application sets search path:
SET search_path TO tenant_123, public;
SELECT * FROM orders;  -- searches tenant_123.orders first

-- Used by: medium-scale SaaS (hundreds of tenants)
-- Better isolation, harder to query across tenants
```

### Separate Database (highest isolation)
```sql
-- Each tenant gets their own database
-- Shopify uses this: one MySQL pod per ~500K merchants
-- Shard by tenant ID: merchant_123 -> Pod 3

-- Pros: complete isolation, easier compliance (GDPR per tenant)
-- Cons: more databases to manage, no cross-tenant queries
-- Used by: Shopify (MySQL pods), enterprise SaaS
```

---

## Pattern 6: The EAV Anti-Pattern (and When It's OK)

**EAV = Entity-Attribute-Value** - the temptation to make everything flexible:

```sql
-- [X] EAV anti-pattern:
CREATE TABLE product_attributes (
  product_id BIGINT NOT NULL,
  name       VARCHAR(100) NOT NULL,  -- 'color', 'size', 'weight'
  value      TEXT NOT NULL           -- 'red', 'large', '500g'
);

-- Problems:
-- Can't type-check values (weight stored as text, not decimal)
-- Queries are nightmarish:
SELECT p.*,
  MAX(CASE WHEN a.name = 'color' THEN a.value END) AS color,
  MAX(CASE WHEN a.name = 'size' THEN a.value END) AS size
FROM products p
LEFT JOIN product_attributes a ON p.id = a.product_id
GROUP BY p.id;
-- vs just: SELECT id, color, size FROM products;

-- [OK] Better alternatives:

-- Option 1: Separate tables for different product types
CREATE TABLE electronics (color VARCHAR(20), battery_mah INT);
CREATE TABLE clothing (size CHAR(3), material VARCHAR(50));

-- Option 2: JSONB for truly flexible attributes
CREATE TABLE products (
  id         BIGINT PRIMARY KEY,
  name       VARCHAR(200) NOT NULL,
  attributes JSONB  -- {"color": "red", "size": "L", "weight_g": 500}
);
CREATE INDEX ON products USING GIN(attributes);  -- indexed for queries
SELECT * FROM products WHERE attributes->>'color' = 'red';
-- Used by: Shopify (product metafields), WooCommerce
```

---

## Pattern 7: Denormalization Patterns for Read Performance

```sql
-- Computed column (PostgreSQL GENERATED column)
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  subtotal_cents INT NOT NULL,
  tax_cents INT NOT NULL,
  -- computed, always stays in sync:
  total_cents INT GENERATED ALWAYS AS (subtotal_cents + tax_cents) STORED
);
-- No need to update total manually; always correct

-- Materialized View (pre-computed complex query)
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT
  DATE(created_at) AS date,
  COUNT(*) AS order_count,
  SUM(total_cents) AS revenue_cents
FROM orders
WHERE status = 'completed'
GROUP BY DATE(created_at);

CREATE UNIQUE INDEX ON daily_revenue(date);
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;  -- refresh daily

-- Dashboard query (instant, not scanning 50M orders):
SELECT * FROM daily_revenue WHERE date >= CURRENT_DATE - 30 ORDER BY date;
```

---

## In Clean Architecture Terms

```
Schema patterns = Infrastructure layer decisions

The Use Case layer defines:
  - What data is needed: "I need a product with its current price"
  - What the data means: "current price = the price active today"

The Infrastructure layer decides:
  - How to store "current price":
    Option A: bi-temporal table (valid_from, valid_to)
    Option B: current_prices table with archived_prices table
    Option C: computed column from history table

  - How to enforce constraints:
    NOT NULL, FK constraints, CHECK constraints, exclusion constraints

  - How to design for the access pattern:
    What indexes to add
    Whether to denormalize
    Whether to use a materialized view

Domain Entity (Product):
  currentPrice(): Money -> business concept
  priceAt(date: Date): Money -> business concept

Repository Implementation:
  Decides whether to use temporal table, bi-temporal, or materialized view
  Use Case never changes when you change the storage strategy
```

---

## Schema Design Checklist

Before finalizing a schema:

- [ ] Every table has a PRIMARY KEY
- [ ] Foreign keys defined with explicit ON DELETE behavior
- [ ] NOT NULL where data is required (be explicit)
- [ ] Appropriate data types (DECIMAL for money, not FLOAT)
- [ ] Timestamps with timezone (TIMESTAMPTZ not TIMESTAMP)
- [ ] Indexes on all JOIN conditions and common WHERE columns
- [ ] Unique constraints where uniqueness is required
- [ ] CHECK constraints for business rules (balance >= 0)
- [ ] Soft delete strategy decided (is deleted_at needed?)
- [ ] Audit columns added (created_at, updated_at, created_by)
- [ ] Multi-tenancy isolation level decided
- [ ] Primary key strategy decided (UUID v7 for distributed, BIGINT for simple)

**Remember:** Schema mistakes are the most expensive mistakes. Get a code review on schema changes before running migrations on production.
