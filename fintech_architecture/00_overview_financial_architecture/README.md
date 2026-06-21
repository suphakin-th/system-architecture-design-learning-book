# Financial Systems Architecture — Complete Overview

> "In financial systems, the cost of a bug is not a bad user experience. It is real money lost, real accounts corrupted, real people's savings gone. This changes everything about how you design, test, and operate these systems." — Senior fintech architect

---

## Why Financial Architecture Is Different

| Concern | Normal App | Financial System |
|---|---|---|
| A bug causes | Bad UX, user retries | Money lost, account corrupted |
| Data loss means | Some records missing | Regulatory violation, lawsuit |
| Downtime means | Users annoyed | Transactions fail, money stuck |
| Wrong number | Display error | Customer overcharged/undercharged |
| Missing record | Re-fetch from server | Compliance failure, audit finding |

---

## The Stack of Financial Architecture

```
Layer 1: BUSINESS MODEL
  Double-Entry Bookkeeping
  → Every transaction: debit one account, credit another
  → Sum of all debits = Sum of all credits ALWAYS
  → Mathematical proof of correctness

Layer 2: DATA INTEGRITY
  Immutable Append-Only Ledger
  → Never UPDATE or DELETE financial records
  → Every change is a new INSERT
  → Cryptographic hash chain (tamper-evident)

Layer 3: TRANSACTION SAFETY
  ACID Transactions + Idempotency
  → All-or-nothing: debit + credit together or neither
  → Idempotency keys: charge exactly once even with retries

Layer 4: DISTRIBUTED SAFETY
  Saga Pattern (not 2PC) for cross-service transactions
  → Each step has a compensating action
  → No distributed locks that can deadlock

Layer 5: SECURITY
  HSM → Key management (keys never leave hardware)
  PCI DSS → Compliance framework
  mTLS → Mutual auth between all services
  Zero Trust → Never trust, always verify

Layer 6: AUDIT & COMPLIANCE
  Write-Ahead Log (WAL)
  Append-only transaction log
  Real-time reconciliation
  7-year retention (regulatory)

Layer 7: PAYMENT NETWORK
  ISO 20022 messaging standard
  VisaNet / Banknet / SWIFT
  PromptPay / BAHTNET (Thailand)
  Real-Time Gross Settlement (RTGS)
```

---

## The Fundamental Law: Conservation of Money

```sql
-- This must ALWAYS be true in your ledger:
-- Sum of all debit entries = Sum of all credit entries

SELECT
  SUM(CASE WHEN type = 'debit'  THEN amount ELSE 0 END) AS total_debits,
  SUM(CASE WHEN type = 'credit' THEN amount ELSE 0 END) AS total_credits,
  SUM(CASE WHEN type = 'debit'  THEN amount ELSE 0 END) -
  SUM(CASE WHEN type = 'credit' THEN amount ELSE 0 END) AS imbalance
FROM journal_entries;

-- imbalance must always = 0
-- If it's not zero → data corruption → investigate immediately
-- This is your first monitoring alert
```

---

## The Three Rules That Cannot Be Broken

### Rule 1: Money is Never Created or Destroyed in the System

```
Transfer $100 from Alice to Bob:
  Alice's account: -$100 (debit)
  Bob's account:   +$100 (credit)
  Net change to system: $0

Payment received $100 from Stripe:
  Stripe liability account:  -$100 (debit → we owe Stripe less)
  Customer account:          +$100 (credit → customer has more)
  Net change to system: $0

WRONG approach:
  Alice.balance -= 100;  // where did the $100 go?
  Bob.balance += 100;    // where did the $100 come from?
  → No audit trail of the TRANSFER itself
  → If Alice is debited and system crashes → $100 lost
```

### Rule 2: Every Financial Record Is Immutable

```sql
-- NEVER do this in a financial system:
UPDATE transactions SET amount = 100 WHERE id = 'txn_123';
DELETE FROM transactions WHERE id = 'txn_123';

-- ALWAYS do this:
-- If a transaction is wrong, REVERSE it with a new transaction
INSERT INTO journal_entries (
  transaction_id, type, amount, account_id,
  description, reversed_entry_id
) VALUES (
  'txn_456', 'credit', 100, 'acc_alice',
  'Reversal of txn_123', 'entry_789'
);
-- The wrong entry stays visible. The correction is a new entry.
-- Auditors can trace exactly what happened and when.
```

### Rule 3: Idempotency — Process Each Payment Exactly Once

```
Without idempotency:
  User clicks "Pay" → network timeout → user clicks "Pay" again
  → Two charges processed → user double-charged → angry customer, refund needed

With idempotency:
  User clicks "Pay" → generates idempotency_key = uuid()
  Network timeout → user clicks "Pay" again → SAME idempotency_key
  Second request: "already processed → return stored result"
  → Charged exactly once → correct
```

---

## Financial Architecture Patterns Index

| Pattern | What It Solves | Where Used |
|---|---|---|
| **Double-Entry Ledger** | Conservation of money, audit trail | ALL financial systems |
| **Immutable Audit Log** | Tamper-proof record, compliance | Banking, payments, fintech |
| **Payment Networks** | Routing money between banks | Visa, Mastercard, SWIFT |
| **PromptPay/RTGS** | Real-time interbank settlement | Thailand, UK Faster Payments |
| **HSM + PCI DSS** | Cryptographic key security | Card processing, banking |
| **Zero Trust Network** | No implicit trust in any system | All secure financial systems |
| **Distributed Transactions** | Cross-service money movement | Microservices banking |
| **Company Case Studies** | Stripe, Monzo, Wise real examples | Learning from the best |

---

## Technologies Used by Real Financial Systems

| Category | Technology | Used By |
|---|---|---|
| **Database** | PostgreSQL, Oracle, DB2 | Most banks, Stripe, Monzo |
| **Ledger DB** | TigerBeetle, Amazon QLDB | Stripe, fintech startups |
| **Message Queue** | Kafka, IBM MQ, ActiveMQ | Banking middleware |
| **Cache** | Redis | Rate limiting, session, idempotency |
| **Messaging Standard** | ISO 20022, ISO 8583 | SWIFT, Visa, PromptPay |
| **HSM** | Thales Luna, Futurex, AWS CloudHSM | PCI DSS compliance |
| **Monitoring** | Prometheus, ELK, Splunk | All production systems |
| **Audit** | Immutable S3, Amazon QLDB, PostgreSQL with triggers | Compliance teams |
| **Encryption** | AES-256 at rest, TLS 1.3 in transit | PCI DSS Level 1 |
| **Auth** | OAuth2 + PKCE, mTLS, FIDO2 | All financial APIs |
