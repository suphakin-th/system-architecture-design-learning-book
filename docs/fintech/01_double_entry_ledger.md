# Double-Entry Ledger - The Foundation of Financial Systems

> "Double-entry bookkeeping is the only invention that can claim to have changed the world more than gunpowder." - Johann Wolfgang von Goethe (1795)
>
> "It's 500 years old and we use it every day. Because it works." - Modern fintech architect

---

## At a Glance

| | |
|---|---|
| **Invented** | 1494 by Luca Pacioli (mathematician, friend of Leonardo da Vinci) |
| **Used by** | Every bank, payment processor, and serious fintech company |
| **Key property** | For every transaction: Debits = Credits (mathematically provable) |
| **Why it matters** | Makes fraud, corruption, and accounting errors detectable |

---

## Senior Explains to Junior: The Core Concept

> "Forget how banks traditionally teach accounting. Here's the programmer's view:
>
> Your database has ACCOUNTS (buckets that hold money) and JOURNAL ENTRIES (records of money moving between buckets). Money never appears or disappears - it only moves. Every movement has two sides: money leaves one bucket (debit) and enters another (credit). If the sum of debits != sum of credits, something is wrong - mathematically impossible to balance the books with an error."

```
Old way (single-entry, what most beginners code):
  Alice.balance = Alice.balance - 100
  Bob.balance = Bob.balance + 100
  Problem: if the system crashes after line 1, $100 disappears!

Double-entry way:
  Record in journal:
    DR Alice:    -100  (Alice's account debited)
    CR Bob:      +100  (Bob's account credited)
  Total debit = Total credit = ZERO NET CHANGE TO SYSTEM
  If system crashes: replay the journal entry -> consistent state restored
```

---

## The Database Schema

### Core Tables

```sql
-- Chart of accounts: all the "buckets" money can be in
CREATE TABLE accounts (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code        VARCHAR(20) NOT NULL UNIQUE,  -- e.g., 'ALICE_CHECKING', 'STRIPE_LIABILITY'
  name        VARCHAR(200) NOT NULL,
  type        VARCHAR(20) NOT NULL
              CHECK (type IN ('asset','liability','equity','revenue','expense')),
  currency    CHAR(3) NOT NULL DEFAULT 'THB',
  is_system   BOOLEAN NOT NULL DEFAULT FALSE,  -- internal system accounts
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- The immutable journal: ALL money movements recorded here
CREATE TABLE journal_entries (
  id              BIGSERIAL PRIMARY KEY,
  transaction_id  UUID NOT NULL,           -- groups related debit+credit entries
  account_id      UUID NOT NULL REFERENCES accounts(id),
  type            CHAR(1) NOT NULL CHECK (type IN ('D','C')),  -- Debit or Credit
  amount_cents    BIGINT NOT NULL CHECK (amount_cents > 0),    -- always positive!
  currency        CHAR(3) NOT NULL,
  description     TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_by      UUID NOT NULL,           -- which service/user created this
  idempotency_key VARCHAR(128) UNIQUE,     -- prevents duplicate entries
  -- Cryptographic hash chain (tamper-evident)
  entry_hash      CHAR(64),               -- SHA-256 of this entry
  prev_hash       CHAR(64)                -- SHA-256 of previous entry (chain!)
);

-- Transactions: the business event that triggered journal entries
CREATE TABLE transactions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type            VARCHAR(50) NOT NULL,   -- 'transfer', 'payment', 'refund', 'fee'
  status          VARCHAR(20) NOT NULL CHECK (status IN ('pending','completed','failed','reversed')),
  description     TEXT NOT NULL,
  metadata        JSONB,                  -- extra context (order_id, customer_id, etc.)
  idempotency_key VARCHAR(128) UNIQUE NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);

-- Constraint: enforce double-entry balance (debits = credits per transaction)
-- This is enforced at application level + verified by reconciliation job
```

### The Balance Sheet View (computed from journal)

```sql
-- Current balance of any account (computed from journal entries)
-- NEVER store balance directly -- always compute from the journal
CREATE VIEW account_balances AS
SELECT
  a.id,
  a.code,
  a.name,
  a.type,
  a.currency,
  COALESCE(SUM(CASE
    WHEN je.type = 'D' AND a.type IN ('asset','expense') THEN je.amount_cents
    WHEN je.type = 'C' AND a.type IN ('liability','equity','revenue') THEN je.amount_cents
    WHEN je.type = 'C' AND a.type IN ('asset','expense') THEN -je.amount_cents
    WHEN je.type = 'D' AND a.type IN ('liability','equity','revenue') THEN -je.amount_cents
    END), 0) AS balance_cents
FROM accounts a
LEFT JOIN journal_entries je ON je.account_id = a.id
GROUP BY a.id, a.code, a.name, a.type, a.currency;

-- WHY compute balance from journal (not store it)?
-- Because balance is derived from the immutable log
-- Any corruption in balance is detectable by recomputing from journal
-- Balance = just a cache of truth; journal = the truth
```

---

## The Transfer Function - The Most Important Code You'll Write

```typescript
// This function moves money between accounts
// It MUST be atomic - all entries recorded or none
async function transfer(params: {
  fromAccountId: string;
  toAccountId:   string;
  amountCents:   number;
  currency:      string;
  description:   string;
  idempotencyKey: string;
}): Promise<Transaction> {
  // Step 1: Check idempotency - already processed?
  const existing = await db.query<Transaction>(
    `SELECT * FROM transactions WHERE idempotency_key = $1`,
    [params.idempotencyKey]
  );
  if (existing.rows[0]) return existing.rows[0];  // Return stored result

  // Step 2: Validate
  if (params.amountCents <= 0) throw new Error('Amount must be positive');

  // Step 3: Execute in a SINGLE ATOMIC TRANSACTION
  return await db.transaction(async (trx) => {
    // Check balance WITHIN the transaction (prevents race conditions)
    const { rows: [from] } = await trx.query(
      `SELECT balance_cents FROM account_balances WHERE id = $1 FOR UPDATE`,
      [params.fromAccountId]
    );
    if (from.balance_cents < params.amountCents) {
      throw new InsufficientFundsError(from.balance_cents, params.amountCents);
    }

    // Create the transaction record
    const txnId = crypto.randomUUID();
    await trx.query(
      `INSERT INTO transactions (id, type, status, description, idempotency_key)
       VALUES ($1, 'transfer', 'pending', $2, $3)`,
      [txnId, params.description, params.idempotencyKey]
    );

    // Record DEBIT (money leaves source account)
    await trx.query(
      `INSERT INTO journal_entries
       (transaction_id, account_id, type, amount_cents, currency, description, created_by)
       VALUES ($1, $2, 'D', $3, $4, $5, 'system')`,
      [txnId, params.fromAccountId, params.amountCents, params.currency, `Transfer out: ${params.description}`]
    );

    // Record CREDIT (money enters destination account)
    await trx.query(
      `INSERT INTO journal_entries
       (transaction_id, account_id, type, amount_cents, currency, description, created_by)
       VALUES ($1, $2, 'C', $3, $4, $5, 'system')`,
      [txnId, params.toAccountId, params.amountCents, params.currency, `Transfer in: ${params.description}`]
    );

    // Update transaction status
    await trx.query(
      `UPDATE transactions SET status = 'completed', completed_at = NOW() WHERE id = $1`,
      [txnId]
    );

    return { id: txnId, status: 'completed' };
  });
  // If ANY step fails -> entire transaction rolled back -> no partial state
}
```

---

## System Accounts - The Hidden Accounts

Real financial systems have internal "system accounts" that represent money in transit or system-held funds:

```sql
-- System accounts (examples)
INSERT INTO accounts (code, name, type, is_system) VALUES
  ('SUSPENSE',     'Suspense Account (money in transit)',    'liability', true),
  ('FEES_REVENUE', 'Fee Revenue',                            'revenue',   true),
  ('STRIPE_SETTLEMENT', 'Stripe Settlement Clearing',       'asset',     true),
  ('UNCOLLECTED', 'Uncollected Payments',                   'asset',     true);

-- When a customer initiates a payment but it hasn't cleared yet:
-- DR: SUSPENSE account (we hold the money)
-- CR: Customer's account (we owe them less / they've paid)
--
-- When the payment clears:
-- DR: STRIPE_SETTLEMENT (actual cash received)
-- CR: SUSPENSE (no longer in transit)
```

---

## The Reconciliation Invariant

```sql
-- Run this EVERY MINUTE as a monitoring check
-- If this returns non-zero, alert IMMEDIATELY

WITH debits AS (
  SELECT transaction_id, SUM(amount_cents) AS total
  FROM journal_entries WHERE type = 'D'
  GROUP BY transaction_id
),
credits AS (
  SELECT transaction_id, SUM(amount_cents) AS total
  FROM journal_entries WHERE type = 'C'
  GROUP BY transaction_id
)
SELECT
  d.transaction_id,
  d.total AS debit_total,
  c.total AS credit_total,
  d.total - c.total AS imbalance
FROM debits d
JOIN credits c ON d.transaction_id = c.transaction_id
WHERE d.total != c.total;

-- MUST return 0 rows. ALWAYS.
-- Any row = data corruption = STOP PROCESSING IMMEDIATELY
```

---

## Real Companies Using Double-Entry Ledger

### Stripe
- Built "Ledger" - their internal double-entry accounting system
- Processes $1T+ annually through double-entry journal entries
- Every PaymentIntent, Charge, Refund, Transfer = journal entries
- 5 billion events per day ingested into the ledger
- Custom internal tool: maps every dollar to two (or more) journal entries

### Monzo
- Cassandra-backed ledger service
- Single service responsible for ALL money movements
- Every transaction = immutable journal entries in Cassandra
- Balance computed at read time from journal history (no stored balance)

### TigerBeetle (open source, built for this)
- Purpose-built database for double-entry accounting
- Handles 1M+ transfers per second
- Enforces: debits = credits at the DB level (not just application)
- Used by: Tigerbeetle startup + growing list of fintechs

### Every Bank in the World
- IBM DB2 + COBOL core banking systems (1960s-present)
- Double-entry has not changed in 500 years
- The accounting logic is unchanged; only the infrastructure evolves

---

## In Clean Architecture Terms

```
Double-Entry Ledger = Domain Layer (the most important domain model)

Account = Entity (aggregate root)
  id, code, name, type, currency

JournalEntry = Entity (immutable value object once created)
  transaction_id, account_id, type, amount_cents, created_at

Transfer = Use Case
  execute(from, to, amount, idempotencyKey): Transaction

ILedgerRepository = Port (interface defined in use case layer)
  recordJournalEntries(debit, credit, txnId): void
  getBalance(accountId): Money

LedgerRepositoryPostgres = Adapter (implements ILedgerRepository)
  All SQL logic here
  Single ACID transaction wrapping debit + credit

Reconciliation = Use Case (scheduled, runs every minute)
  verifyBalanceInvariant(): InvariantViolation[]

The Domain Rule lives in the entity/use case layer:
  SUM(debits) = SUM(credits) for every transaction
  This rule is never moved to the adapter or infrastructure layer.
```
