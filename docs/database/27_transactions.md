# Database Transactions - ACID, BASE, CAP, PACELC

> "ACID is a guarantee. BASE is an attitude. CAP is a law of physics. Understanding which one applies to your system is the difference between building a bank and building a chatbot." - Senior architect

---

## At a Glance

| | |
|---|---|
| **ACID** | Atomicity, Consistency, Isolation, Durability - the gold standard |
| **BASE** | Basically Available, Soft state, Eventually consistent - the scalability trade-off |
| **CAP** | You can only guarantee 2 of: Consistency, Availability, Partition Tolerance |
| **PACELC** | Extension of CAP: even without partition, choose Latency vs Consistency |

---

## ACID - The Bank Account Standard

### A - Atomicity

"All or nothing. Either ALL operations in the transaction succeed, or NONE of them happen."

```sql
-- Bank transfer: debit Alice, credit Bob
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 1000 WHERE user_id = 'alice';
-- If this succeeds but the next fails...

UPDATE accounts SET balance = balance + 1000 WHERE user_id = 'bob';
-- ...without atomicity, Alice loses $1000 and Bob gets nothing

COMMIT;
-- Only if BOTH succeed does the commit happen
-- If any fails: ROLLBACK -> both operations undone -> Alice keeps her money

-- This is why banks use relational databases with ACID, not Cassandra
```

### C - Consistency

"The database moves from one valid state to another valid state. Constraints are never violated."

```sql
-- Constraint example:
CREATE TABLE accounts (
  id BIGINT PRIMARY KEY,
  balance INT NOT NULL CHECK (balance >= 0)  -- balance can't be negative
);

-- Transaction that would violate consistency:
BEGIN;
UPDATE accounts SET balance = balance - 5000 WHERE id = 1;  -- balance becomes -1000
-- PostgreSQL: ERROR: new row for relation "accounts" violates check constraint
-- Transaction rolled back automatically
-- The database REFUSED to enter an invalid state
ROLLBACK;
```

### I - Isolation

"Concurrent transactions don't interfere with each other. Each transaction sees the database as if it were running alone."

```
Isolation problems without proper levels:

Dirty Read: Transaction A reads data modified by Transaction B which hasn't committed yet
  A reads "balance = 500" (B is in the middle of updating to 200)
  B fails and rolls back -> actual balance is 700, but A read 500
  A made a decision based on phantom data

Non-Repeatable Read: Same row gives different results in same transaction
  A reads "product price = 100"
  B updates price to 150 and commits
  A reads same row again -> "product price = 150"
  A made inconsistent calculations

Phantom Read: Same query returns different set of rows in same transaction
  A: "SELECT COUNT(*) WHERE status='pending'" -> 5 orders
  B: INSERT new pending order, COMMIT
  A: "SELECT COUNT(*) WHERE status='pending'" -> 6 orders
  A's aggregate is inconsistent

PostgreSQL Isolation Levels:
  READ UNCOMMITTED: no protection (dirty reads possible)
  READ COMMITTED: prevents dirty reads (default in most DBs)
  REPEATABLE READ: prevents non-repeatable reads
  SERIALIZABLE: prevents all, including phantom reads (slowest)
```

```sql
-- Most operations: READ COMMITTED (default)
BEGIN;
SELECT * FROM orders WHERE user_id = 123;  -- sees only committed data
COMMIT;

-- Critical operations: SERIALIZABLE
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT balance FROM accounts WHERE id = 1;
-- Even if another transaction commits a change, this transaction
-- will see consistent state throughout
UPDATE accounts SET balance = balance - 100;
COMMIT;
```

### D - Durability

"Once committed, data survives crashes."

```
How PostgreSQL ensures durability:

1. Write to WAL (Write-Ahead Log) FIRST
   The WAL is sequential writes -> very fast
   If crash: replay WAL -> recover all committed transactions

2. Fsync: WAL data physically written to disk before COMMIT returns
   fsync = flush OS buffer to actual disk
   Without fsync: OS might have data in memory -> power loss = data loss

3. Checkpoints: periodically flush dirty pages from memory to data files
   Between checkpoints: WAL ensures recovery

Trade-off: fsync costs ~5ms per COMMIT
  For banking: acceptable (durability is mandatory)
  For logging: might turn off fsync (risk data loss for performance)
```

---

## BASE - The NoSQL Philosophy

**B**asically **A**vailable **S**oft state **E**ventually consistent

```
Basically Available:
  System stays up even during partial failures
  Some data might be stale, some nodes might be unavailable
  BUT: the system as a whole responds (no 503)

  Example: During a Cassandra node failure:
    That node's data is served from a replica (possibly stale)
    No total outage - just "eventually consistent" responses

Soft State:
  System state may change over time even without new inputs
  (Replication is still propagating, eventually converging)

  Example: After writing to Cassandra node 1:
    Node 2 and 3 haven't received the update yet
    State is "soft" - will converge when replication catches up

Eventually Consistent:
  Given no new updates, all replicas will eventually agree
  "Eventually" = milliseconds to seconds in practice

  Example: After a write to DynamoDB primary:
    Read from the same region: consistent (reads from primary)
    Read from another region: might see old data for 100-500ms
    After replication: all regions consistent
```

---

## ACID vs BASE Decision Guide

```
Use ACID (PostgreSQL, MySQL with InnoDB) when:
  [OK] Financial transactions: money transfer, payment processing
  [OK] Inventory: can't oversell stock (check stock, decrement, or fail)
  [OK] User accounts: creating account must be atomic (profile + settings + preferences)
  [OK] Any operation where partial completion = corrupt state

Use BASE (Cassandra, DynamoDB, MongoDB default) when:
  [OK] Social media feeds: showing slightly old "like count" is fine
  [OK] Product recommendations: stale recommendations don't hurt the user
  [OK] Analytics data: approximate counts are acceptable
  [OK] Session data: if session data is briefly inconsistent, user just re-logs in
  [OK] Shopping cart: temporary inconsistency during heavy traffic is OK
```

**Real example - Uber surge pricing:**
```
Strict ACID for surge pricing?
  Every surge price update requires locking the entire city's data
  One slow transaction blocks all price reads in Bangkok
 -> Not feasible at 15M location updates/second

BASE approach:
  Surge multiplier stored in Redis (eventually consistent)
  Calculated by Flink stream processor every 30 seconds
  May be 30 seconds stale - but that's fine
  Driver gets $X.XX/km with 30s-old surge factor - acceptable

ACID for payment:
  When the trip ends and payment is charged:
 -> PostgreSQL, ACID, synchronous replication
 -> Can't be eventually consistent here - money is involved
```

---

## The Two-Phase Commit (2PC) - Distributed ACID

When a transaction spans MULTIPLE databases (microservices), you need coordination:

```
Scenario: PlaceOrder
  1. Deduct inventory (InventoryDB)
  2. Charge payment (PaymentDB)
  3. Create order record (OrderDB)

  All three MUST succeed or ALL MUST fail (atomicity across services)

Two-Phase Commit Protocol:

Phase 1 - PREPARE:
  Coordinator -> InventoryDB: "Prepare to deduct stock 5"
  InventoryDB: checks, locks stock, replies "PREPARED"

  Coordinator -> PaymentDB: "Prepare to charge $99.90"
  PaymentDB: checks, holds auth, replies "PREPARED"

  Coordinator -> OrderDB: "Prepare to create order"
  OrderDB: checks constraints, replies "PREPARED"

Phase 2 - COMMIT (or ABORT):
  If ALL said "PREPARED":
    Coordinator -> all: "COMMIT"
    Each commits their local transaction

  If ANY said "ABORT":
    Coordinator -> all: "ABORT"
    Each rolls back their local transaction

Problems with 2PC:
  [X] Blocking: if coordinator crashes after "PREPARE" but before "COMMIT",
    all participants hold locks forever (blocking protocol)
  [X] Latency: 2 round-trips to all participants before commit
  [X] Not partition-tolerant: if network partition -> participants wait forever

Why most microservices DON'T use 2PC:
 -> Too slow, too fragile
 -> Use SAGA pattern instead (see Pattern 06)
```

---

## SAGA - Distributed Transactions Without 2PC

```
Instead of locking across services, use compensating transactions:

PlaceOrder Saga:
  Step 1: Deduct inventory -> SUCCESS
  Step 2: Charge payment -> FAILURE (card declined)

  Compensate in reverse:
    Step 2 compensation: do nothing (payment never happened)
    Step 1 compensation: restore inventory (+5 units back)

  Eventual result: inventory restored, order not created, user not charged

This is ACID per service, but eventual consistency across services
Each step is atomic locally; the saga is eventually consistent globally
```

---

## Transaction Isolation Anomalies in Practice

```sql
-- The classic race condition:

-- Thread 1:                          -- Thread 2:
SELECT balance FROM accounts          SELECT balance FROM accounts
WHERE id = 1;  -- reads: 1000         WHERE id = 1;  -- reads: 1000
-- (thinks there's 1000 to spend)      -- (also thinks there's 1000 to spend)

UPDATE accounts SET balance = 0        UPDATE accounts SET balance = 0
WHERE id = 1;                          WHERE id = 1;
-- Both succeed! Account now 0 but TWO withdrawals of 1000 happened!

-- Solution 1: SELECT FOR UPDATE (pessimistic locking)
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- This LOCKS the row: Thread 2 must wait until Thread 1 commits
-- Safe but can cause contention

-- Solution 2: Optimistic concurrency (version number)
-- Add version column to table
UPDATE accounts
SET balance = balance - 1000, version = version + 1
WHERE id = 1 AND version = 5;  -- only if version hasn't changed
-- If another transaction updated first, version != 5 -> 0 rows updated -> retry

-- Solution 3: Single UPDATE (atomic)
UPDATE accounts
SET balance = balance - 1000
WHERE id = 1 AND balance >= 1000;  -- only if there's enough balance
-- The CHECK happens atomically in the DB -> no race condition
```

---

## Database Locking Strategies

```sql
-- Row-level lock (minimal blocking):
SELECT * FROM orders WHERE id = 123 FOR UPDATE;
-- Locks only row 123, not the whole table

-- Shared lock (read lock - multiple readers, block writes):
SELECT * FROM orders WHERE id = 123 FOR SHARE;

-- Table lock (maximum blocking - avoid):
LOCK TABLE orders IN ACCESS EXCLUSIVE MODE;
-- Blocks ALL reads and writes -> use only for maintenance

-- Deadlock example:
-- Transaction A: locks order 1, then tries to lock order 2
-- Transaction B: locks order 2, then tries to lock order 1
-- -> Deadlock: each waiting for the other
-- PostgreSQL detects and kills one transaction

-- Deadlock prevention:
-- Always acquire locks in the same order
-- Minimize transaction duration
-- Use application-level locking (Redis distributed lock)
```

---

## In Clean Architecture Terms

```
Transaction management = boundary between Use Case and Infrastructure

Use Case boundary pattern:
  PlaceOrderUseCase.execute() = one unit of work = one transaction

Repository implementation wraps the transaction:
  class OrderRepositoryPostgres {
    async saveWithInventoryUpdate(order: Order, inventory: InventoryChange) {
      await this.db.transaction(async (trx) => {
        await trx.query('INSERT INTO orders ...');
        await trx.query('UPDATE inventory ...');
        // Both or neither - atomicity guaranteed by the DB transaction
      });
    }
  }

The Use Case says WHAT is one atomic operation
The Repository decides HOW to implement the atomicity
= Clean separation of concerns

For cross-service transactions (SAGA):
  SagaOrchestrator = Use Case that spans multiple service ports
  Each service port = interface
  Each compensation = a method on the interface
  Use Case orchestrates the happy path + compensation path
```
