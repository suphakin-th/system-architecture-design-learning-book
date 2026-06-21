# Database Design — Fundamentals

> "A database schema is your most important architectural decision. It's the hardest thing to change later and the foundation everything else sits on." — Senior DBA wisdom

---

## At a Glance

| | |
|---|---|
| **Topic** | Core database design principles every developer must know |
| **Applies to** | Every application that persists data |
| **Skill level** | Zero to hero — start here before patterns |

---

## The Senior Explains It To The Junior

> "Every bug that makes it to production is either a logic bug or a data bug. Logic bugs are easy — you see them in logs, you fix the code, you redeploy. Data bugs are different. If you design your schema wrong, the bad data is IN THE DATABASE. You can't fix it with a code deploy. You need a migration, which might take hours on a 100M-row table, during which your site is slow or locked. **Schema design mistakes are the most expensive mistakes you'll ever make.**"

---

## The Three Forms of Normal Form (Normalization)

Normalization eliminates redundancy. Redundancy = data inconsistency = bugs.

### 1st Normal Form (1NF) — Atomic Values

**Rule:** Every cell must contain ONE value. No arrays, no comma-separated lists in a column.

```sql
-- ❌ WRONG: phone numbers in one column
users:
  id | name  | phones
  1  | Alice | "0812345678, 0823456789"

-- Problem: How do you query "find all users with phone 0812345678"?
-- SELECT * FROM users WHERE phones LIKE '%0812345678%' -- SLOW, wrong

-- ✅ CORRECT: 1NF — separate table for multi-valued attributes
users:
  id | name
  1  | Alice

user_phones:
  user_id | phone
  1       | 0812345678
  1       | 0823456789
```

### 2nd Normal Form (2NF) — No Partial Dependencies

**Rule:** Every non-key column must depend on the WHOLE primary key (applies to composite keys).

```sql
-- ❌ WRONG: order_items with composite key (order_id, product_id)
order_items:
  order_id | product_id | quantity | product_name | product_price

-- Problem: product_name depends only on product_id, NOT on (order_id, product_id)
-- If you update product_name, you update it in EVERY order row that has that product
-- One place gives wrong value = data inconsistency

-- ✅ CORRECT: 2NF — move product data to its own table
order_items:
  order_id | product_id | quantity | unit_price_at_time_of_order

products:
  id | name | current_price
```

Note: `unit_price_at_time_of_order` is intentional — product price can change; the order must remember what it was at purchase time.

### 3rd Normal Form (3NF) — No Transitive Dependencies

**Rule:** Non-key columns must depend on the primary key, not on OTHER non-key columns.

```sql
-- ❌ WRONG: employees table
employees:
  id | name | department_id | department_name | department_budget

-- Problem: department_name and department_budget depend on department_id
-- not directly on employee id
-- If department name changes: update EVERY employee row

-- ✅ CORRECT: 3NF — separate table
employees:
  id | name | department_id

departments:
  id | name | budget
```

### When to Denormalize (and why it's not always wrong)

Denormalization = intentionally breaking normalization rules for read performance.

```sql
-- Normalized (slow for reads — requires JOIN):
orders JOIN order_items JOIN products JOIN customers

-- Denormalized (fast for reads — one query, pre-joined):
order_summary:
  order_id | customer_name | customer_email | item_names | total

-- When to denormalize:
-- ✓ Read >> Write ratio (reads 100× more than writes)
-- ✓ The JOIN query is too slow (measured, not assumed)
-- ✓ You accept the risk of inconsistency between tables
-- ✓ The denormalized table is the READ MODEL in CQRS
```

> **The rule:** Normalize first. Denormalize only after you measure that the normalized query is too slow. "Premature denormalization is the root of many database evils."

---

## Primary Keys: A Decision That Lasts Forever

```sql
-- Option 1: Auto-increment integer (1, 2, 3, ...)
CREATE TABLE users (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  ...
);
-- ✓ Small (8 bytes), fast index, sequential writes (good for B-tree)
-- ✗ Predictable (user can guess IDs: /users/1, /users/2)
-- ✗ Doesn't work across shards (shard 1 and shard 2 both have user id=1)
-- Use when: single database, internal IDs not exposed to users

-- Option 2: UUID v4 (random)
CREATE TABLE users (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  ...
);
-- ✓ Globally unique (safe for sharding and distributed systems)
-- ✓ Unpredictable (security: can't enumerate users)
-- ✗ 16 bytes (2× larger than BIGINT)
-- ✗ Random insertion breaks B-tree locality → page fragmentation → slower writes
-- Use when: distributed systems, user-facing IDs, microservices

-- Option 3: ULID / UUID v7 (time-ordered UUID)
CREATE TABLE orders (
  id UUID DEFAULT gen_uuid_v7() PRIMARY KEY,
  ...
);
-- ✓ Globally unique
-- ✓ Sortable by time (first 48 bits = timestamp)
-- ✓ Sequential writes (no B-tree fragmentation)
-- ✓ 16 bytes
-- The best of both worlds for distributed systems
-- Use when: distributed, but you want sequential writes
```

**Real-world choices:**
| Company | PK Strategy | Reason |
|---|---|---|
| Twitter | Snowflake ID (64-bit, time+worker+seq) | Globally unique, sortable, compact |
| Instagram | Custom ID (timestamp+shard+sequence) | Similar to Snowflake |
| Stripe | "ch_", "cus_" prefixed string IDs | Human-readable type prefix + random |
| Discord | Snowflake (same as Twitter) | Sortable by time = sort messages by ID |

---

## Foreign Keys: The Referential Integrity Contract

```sql
-- Always define foreign key constraints explicitly
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT
);

-- ON DELETE options:
-- RESTRICT: prevent deletion of user if they have orders (safest)
-- CASCADE: delete orders when user is deleted (dangerous — data loss)
-- SET NULL: set user_id = NULL when user deleted (orphaned records)
-- NO ACTION: same as RESTRICT in most DBs

-- Best practice: use RESTRICT by default
-- Only use CASCADE when the child records are meaningless without the parent
-- (e.g., order_items without order, profile_photos without profile)
```

---

## Data Types: Choosing Correctly

```sql
-- ❌ Common mistakes:
VARCHAR(255)  -- for everything, including fixed-length codes
INT           -- for user_id when you might have > 2B users (INT max = 2.1B)
TEXT          -- for short strings (can't index directly)
FLOAT/DOUBLE  -- for money (floating point rounding errors!)
DATETIME      -- without timezone awareness

-- ✅ Correct choices:
CHAR(2)           -- for country codes (fixed length, faster than VARCHAR)
BIGINT            -- for IDs (safe up to 9.2 quintillion)
VARCHAR(100)      -- for names, emails (appropriate length)
DECIMAL(19, 4)    -- for money (exact decimal arithmetic)
TIMESTAMPTZ       -- for timestamps (PostgreSQL, stores with timezone)
BOOLEAN           -- not TINYINT(1)
JSONB             -- for flexible schema data (PostgreSQL, indexed)
```

**The money problem:**
```sql
-- ❌ WRONG:
price FLOAT  -- 0.1 + 0.2 = 0.30000000000000004 in IEEE 754

-- ✅ CORRECT:
price DECIMAL(19, 4)  -- exact: 0.1000 + 0.2000 = 0.3000
-- OR: store in cents as INTEGER
price_cents INT  -- $9.99 stored as 999 (no decimal arithmetic needed)
```

---

## Constraints: Your Data's Last Line of Defense

```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  price_cents INT NOT NULL CHECK (price_cents >= 0),  -- can't be negative
  stock_count INT NOT NULL DEFAULT 0 CHECK (stock_count >= 0),
  category_id BIGINT NOT NULL REFERENCES categories(id),
  sku CHAR(12) UNIQUE NOT NULL,  -- one product per SKU
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ  -- NULL = not deleted (soft delete)
);

-- The NOT NULL contract is powerful:
-- If user_id is NOT NULL, you KNOW every row has a user
-- If user_id is NULL, you're saying "this is optional"
-- Be explicit about what's required vs optional
```

---

## Soft Delete vs Hard Delete

```sql
-- Hard delete: data is gone forever
DELETE FROM orders WHERE id = 123;

-- Soft delete: mark as deleted, keep the data
ALTER TABLE orders ADD COLUMN deleted_at TIMESTAMPTZ;
UPDATE orders SET deleted_at = NOW() WHERE id = 123;

-- Query "active" orders:
SELECT * FROM orders WHERE deleted_at IS NULL;

-- When to use soft delete:
-- ✓ Audit trail required (you need to know who deleted what)
-- ✓ GDPR "right to be forgotten" (you mark deleted, then scrub PII separately)
-- ✓ Undo functionality (user accidentally deleted → restore)
-- ✗ Adds WHERE deleted_at IS NULL to EVERY query
-- ✗ Deleted rows still consume storage
-- ✗ Unique indexes need to account for deleted rows
```

---

## The N+1 Query Problem (The Most Common ORM Bug)

```typescript
// ❌ N+1 problem:
const orders = await Order.findAll();  // 1 query → 100 orders
for (const order of orders) {
  const user = await User.findById(order.userId);  // 100 queries!
  // Total: 101 queries for what could be 2
}

// ✅ Eager loading (JOIN):
const orders = await Order.findAll({
  include: [{ model: User }]  // 1 query with JOIN
});

// ✅ Or two queries (better than N+1):
const orders = await Order.findAll();  // 1 query
const userIds = orders.map(o => o.userId);
const users = await User.findAll({ where: { id: userIds } });  // 1 query with IN clause
// Map users to orders in memory
```

---

## In Clean Architecture Terms

```
Database Design = Infrastructure layer concern

IOrderRepository (port) — defined in Use Case layer
  Has no idea about:
    - Table names
    - Column types
    - Join strategies
    - Index existence

OrderRepositoryPostgres (adapter) — in Infrastructure layer
  Implements IOrderRepository
  Knows all of the above
  Contains the SQL/ORM code

Business rule: "Order total must equal sum of items"
  Lives in: Order entity (domain layer)
  NOT in: SQL constraints (though constraints can reinforce it)
  NOT in: Controller

Why this matters: If you decide to switch from PostgreSQL to MongoDB,
  you change the adapter, not the use case, not the entity.
  The business rule is preserved regardless of storage choice.
```

---

## Key Takeaways

1. **Normalize first** — 3NF eliminates most data inconsistency bugs
2. **Denormalize only when measured** — not because you "think" it'll be slow
3. **Never use FLOAT for money** — use DECIMAL or store cents as INT
4. **Always declare constraints** — NOT NULL, UNIQUE, FK, CHECK are your safety net
5. **Primary key choice is permanent** — UUID v7 (ULID) is the modern best practice for distributed systems
6. **N+1 queries will kill your app** — always think about how many queries your ORM generates
7. **Soft delete has hidden complexity** — consider carefully before adding it everywhere
