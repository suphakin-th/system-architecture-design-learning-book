# Immutable Audit Log - Tamper-Evident Financial Records

> "The audit log is not a feature. It is the evidence that your system operated correctly. In a court of law, in a regulatory audit, in a fraud investigation - the audit log is all that matters." - Banking compliance officer

---

## At a Glance

| | |
|---|---|
| **What** | Append-only log where every financial event is recorded permanently |
| **Why** | Regulatory compliance (7-year retention), fraud detection, debugging |
| **Key property** | Cannot be modified without detection (cryptographic hash chain) |
| **Used by** | Every bank (required by law), Stripe, Monzo, Amazon QLDB |

---

## Senior Explains to Junior

> "Imagine you're a police investigator. You need to prove that a transaction happened - exactly $9,900 THB was transferred from account A to account B at 14:32:01.234 on January 15, 2024, by user ID 12345, from IP 203.x.x.x. If your database has an UPDATE or DELETE anywhere in that data's history, the defense lawyer will argue the data was tampered with. You can't prove it wasn't.
>
> The immutable audit log solves this: every record is created once and never changed. The only operation allowed is INSERT. And each record contains a cryptographic hash of the previous record - if anyone changes record #500, record #501's hash won't match anymore. Tampering is detectable."

---

## The Three Types of Audit Log

```
Level 1: Simple Audit Log (basic, what most apps do)
  Log who did what, when
  INSERT INTO audit_log (user_id, action, table_name, record_id, timestamp)
  Problem: the audit_log itself can be updated or deleted!

Level 2: Immutable Audit Log (production fintech)
  Append-only: DB trigger prevents UPDATE/DELETE
  Row-Level Security: no user can delete audit rows
  Backup: replicated to immutable storage (S3 + Object Lock)
  Still problem: a DBA with SUPERUSER could delete records

Level 3: Cryptographic Hash Chain (bank-grade)
  Each record contains SHA-256 hash of all its content
  Each record contains the hash of the PREVIOUS record
  Changing any record breaks all subsequent hashes
  Detectable by running hash verification
  Used by: Amazon QLDB, Stripe Ledger, blockchain-adjacent systems
```

---

## Database Schema: Hash-Chained Audit Log

```sql
-- The immutable event log table
CREATE TABLE financial_events (
  id              BIGSERIAL PRIMARY KEY,

  -- Event data
  event_type      VARCHAR(50) NOT NULL,     -- 'payment.created', 'transfer.completed'
  entity_type     VARCHAR(50) NOT NULL,     -- 'transaction', 'account', 'user'
  entity_id       UUID NOT NULL,

  -- The actual event payload (immutable JSON snapshot)
  event_data      JSONB NOT NULL,           -- complete state at the time of the event

  -- Who, when, where
  occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  actor_type      VARCHAR(30) NOT NULL,     -- 'user', 'system', 'api_key'
  actor_id        VARCHAR(100) NOT NULL,    -- user_id or service_name
  client_ip       INET,                     -- originating IP address
  request_id      UUID,                     -- the HTTP request that caused this event

  -- Cryptographic integrity
  entry_hash      CHAR(64) NOT NULL,        -- SHA-256(id + event_type + event_data + occurred_at + prev_hash)
  prev_hash       CHAR(64) NOT NULL         -- hash of the previous row (forms the chain)
);

-- Prevent ANY modification after INSERT
CREATE RULE no_update_financial_events AS
  ON UPDATE TO financial_events DO INSTEAD NOTHING;
CREATE RULE no_delete_financial_events AS
  ON DELETE TO financial_events DO INSTEAD NOTHING;

-- Row-level security: no user role can delete
ALTER TABLE financial_events ENABLE ROW LEVEL SECURITY;
CREATE POLICY audit_insert_only ON financial_events
  FOR INSERT WITH CHECK (true);
CREATE POLICY audit_select_only ON financial_events
  FOR SELECT USING (true);
-- No UPDATE or DELETE policy = those operations silently fail

-- Grant: only INSERT and SELECT, never UPDATE or DELETE
REVOKE ALL ON financial_events FROM application_user;
GRANT INSERT, SELECT ON financial_events TO application_user;
```

---

## Computing and Verifying the Hash Chain

```typescript
import { createHash } from 'crypto';

// Computing the hash for a new event
function computeEventHash(
  id: number,
  eventType: string,
  eventData: object,
  occurredAt: Date,
  prevHash: string
): string {
  const content = JSON.stringify({
    id,
    eventType,
    eventData,
    occurredAt: occurredAt.toISOString(),
    prevHash,
  });
  return createHash('sha256').update(content).digest('hex');
}

// Recording a new financial event
async function recordEvent(params: {
  eventType: string;
  entityType: string;
  entityId: string;
  eventData: object;
  actorId: string;
  actorType: string;
  clientIp?: string;
}): Promise<void> {
  await db.transaction(async (trx) => {
    // Get the last event's hash (the "tip" of the chain)
    const { rows: [last] } = await trx.query(
      `SELECT id, entry_hash FROM financial_events ORDER BY id DESC LIMIT 1`
    );

    const prevHash = last?.entry_hash ?? 'GENESIS'; // first record uses 'GENESIS'
    const now = new Date();

    // Insert the new event (without id, since it's SERIAL - get it after insert)
    const { rows: [inserted] } = await trx.query(
      `INSERT INTO financial_events
       (event_type, entity_type, entity_id, event_data, occurred_at, actor_type, actor_id, entry_hash, prev_hash)
       VALUES ($1, $2, $3, $4, $5, $6, $7, 'computing', $8)
       RETURNING id`,
      [params.eventType, params.entityType, params.entityId,
       JSON.stringify(params.eventData), now, params.actorType, params.actorId, prevHash]
    );

    // Now compute hash with the real id
    const hash = computeEventHash(
      inserted.id, params.eventType, params.eventData, now, prevHash
    );

    // Update the hash (this is the ONLY update allowed - setting the hash we just computed)
    await trx.query(
      `UPDATE financial_events SET entry_hash = $1 WHERE id = $2`,
      [hash, inserted.id]
    );
  });
}

// Verifying the chain integrity (run as a scheduled job)
async function verifyChainIntegrity(fromId: number = 1): Promise<VerificationResult> {
  const events = await db.query(
    `SELECT * FROM financial_events WHERE id >= $1 ORDER BY id ASC`,
    [fromId]
  );

  let prevHash = 'GENESIS';
  const violations: ChainViolation[] = [];

  for (const event of events.rows) {
    const expectedHash = computeEventHash(
      event.id, event.event_type, event.event_data, event.occurred_at, prevHash
    );

    if (event.entry_hash !== expectedHash) {
      violations.push({
        eventId: event.id,
        expected: expectedHash,
        actual: event.entry_hash,
        message: `Chain broken at event ${event.id} - possible tampering!`,
      });
    }

    if (event.prev_hash !== prevHash) {
      violations.push({
        eventId: event.id,
        expected: prevHash,
        actual: event.prev_hash,
        message: `prev_hash mismatch at event ${event.id} - records may have been deleted or reordered!`,
      });
    }

    prevHash = event.entry_hash;
  }

  return {
    eventsVerified: events.rows.length,
    violations,
    isIntact: violations.length === 0,
  };
}
```

---

## Write-Ahead Log (WAL) - PostgreSQL's Built-in Safety

```
PostgreSQL already has its own immutable WAL (Write-Ahead Log):
  Every change to the database is written to the WAL FIRST
  Then applied to the actual data files
  If the server crashes: replay the WAL -> recover all committed changes
  WAL = the source of truth for durability

For financial systems, we use the WAL in two ways:
  1. Durability: PostgreSQL's built-in durability guarantee (automatic)
  2. Change Data Capture (CDC): read the WAL to build audit streams

Streaming WAL via logical replication:
  PostgreSQL logical replication -> Debezium -> Kafka topic: 'financial.events'
  Every INSERT to journal_entries -> event published to Kafka
  Consumers: audit service, analytics, compliance reporting

Benefits:
  Zero additional overhead on the write path (reading WAL, not the DB)
  Real-time audit stream
  Cannot be disabled (WAL is fundamental to PostgreSQL)
```

---

## Regulatory Requirements for Financial Audit Logs

```
PCI DSS (Payment Card Industry Data Security Standard):
  Requirement 10.2: Log all access to cardholder data
  Requirement 10.3: Log 6 attributes per event (user, date/time, action, object, result, originating IP)
  Requirement 10.5: Protect audit logs from modification
  Requirement 10.7: Retain logs for AT LEAST 1 year (3 months immediately accessible)

SOX (Sarbanes-Oxley, public companies):
  5-7 year retention of financial records
  Immutability: logs cannot be altered
  Access control: who can view logs must be logged too

Bank of Thailand (BOT) / SEC Thailand:
  Financial records: 7 years minimum
  Transaction logs: must be retrievable within 24 hours for audit
  Cybersecurity incident logs: 1 year
  Card data: PCI DSS applies (most Thai banks are PCI Level 1)

GDPR (EU General Data Protection Regulation):
  Right to be forgotten: conflicts with immutable audit logs!
  Resolution: pseudonymize PII in audit logs (user_id hash, not name+email)
  Keep financial records (amounts, transaction IDs) - delete PII link
```

---

## Immutable Storage: Beyond the Database

```
Problem: A DBA with SUPERUSER access could delete PostgreSQL data.
Solution: Replicate to storage that TRULY cannot be modified.

Amazon S3 with Object Lock (WORM: Write Once Read Many):
  aws s3api put-object-lock-configuration \
    --bucket financial-audit-logs \
    --object-lock-configuration Mode=COMPLIANCE,Rule=7years

  Once an object is uploaded with COMPLIANCE mode lock:
    - Not even AWS can delete it before expiry
    - Not even the root account can delete it
    - Regulatory-grade immutability

Amazon QLDB (Quantum Ledger Database):
  Purpose-built for financial audit logs
  Built-in cryptographic hash chain (similar to what we built above)
  Immutable by design: no UPDATE or DELETE
  Owned by: Amazon Managed Service - no DBA can tamper
  Used by: Amazon Pay internally

Azure Immutable Blob Storage:
  Same concept as S3 Object Lock
  WORM compliance for financial institutions

Architecture for maximum immutability:
  PostgreSQL (primary source of truth)
 -> WAL -> Debezium -> Kafka
 -> Kafka -> Audit Consumer -> PostgreSQL audit table (with rules preventing delete)
 -> Audit Consumer -> S3 Object Lock (true immutability, 7 years)
 -> Audit Consumer -> QLDB (cryptographic verification)
```

---

## Practical Implementation: What to Log

```typescript
// EVERY financial event needs these fields:
interface FinancialEvent {
  // Identity
  event_id:     string;    // UUID
  event_type:   string;    // 'payment.charged', 'transfer.initiated'

  // Timing (precision matters for dispute resolution)
  occurred_at:  string;    // ISO 8601 with milliseconds: '2024-01-15T14:32:01.234Z'

  // Actor (who caused this)
  actor_type:   string;    // 'customer', 'merchant', 'system', 'admin'
  actor_id:     string;    // user_id or service name
  session_id:   string;    // to correlate events in same session
  client_ip:    string;    // IP address (geolocation, fraud detection)
  user_agent:   string;    // device fingerprint

  // Entity (what was affected)
  entity_type:  string;    // 'transaction', 'account', 'payment_method'
  entity_id:    string;    // the specific record ID

  // Before/After snapshot
  before_state: object;    // what the entity looked like before
  after_state:  object;    // what the entity looks like after

  // Financial specifics
  amount_cents: number;    // always in smallest currency unit (no floats!)
  currency:     string;    // ISO 4217 (THB, USD, SGD)

  // Tracing
  request_id:   string;    // correlate with HTTP logs
  trace_id:     string;    // distributed tracing ID (Jaeger, Zipkin)

  // Integrity
  entry_hash:   string;    // SHA-256 of this event
  prev_hash:    string;    // hash of previous event (chain)
}

// Events to log (non-exhaustive):
const FINANCIAL_EVENTS = [
  'payment.initiated',        // customer starts payment
  'payment.authorized',       // card authorized (funds held)
  'payment.captured',         // funds actually moved
  'payment.failed',           // any failure with reason
  'payment.refunded',         // full or partial refund
  'transfer.initiated',       // money transfer started
  'transfer.completed',       // transfer settled
  'account.balance_checked',  // someone checked a balance
  'account.created',          // new account opened
  'account.frozen',           // account suspended
  'auth.login_success',       // successful login
  'auth.login_failed',        // failed login (fraud signal)
  'admin.override',           // admin bypassed normal flow
  'kyc.submitted',            // identity verification submitted
  'kyc.approved',             // identity verified
];
```

---

## In Clean Architecture Terms

```
Audit Log = Cross-cutting concern spanning all layers

Domain Events (generated by Entities/Use Cases):
  Order.place() -> emits OrderPlacedEvent
  Transfer.execute() -> emits MoneyTransferredEvent
  These events carry all audit data

Event Publisher (Interface Adapter outbound):
  IAuditEventPublisher (port defined in use case layer)
  KafkaAuditEventPublisher (adapter in infrastructure)
  PostgresAuditEventStore (adapter in infrastructure)
  S3AuditArchiver (adapter in infrastructure)

Use Case pattern:
  PlaceOrderUseCase.execute() -> 
    calls domain methods -> 
    domain emits events -> 
    IAuditEventPublisher.publish(events) -> 
    Events stored in immutable log

Key principle: the Use Case never imports Kafka, S3, or PostgreSQL
It only calls IAuditEventPublisher (interface)
Infrastructure adapters handle the actual immutability guarantees
= Clean Architecture protecting business logic from infrastructure
```
