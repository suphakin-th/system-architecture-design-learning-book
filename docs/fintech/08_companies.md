# Fintech Companies - Architecture Case Studies

## Stripe - The Ledger System

**Scale:** $1T+ annually, 5 billion events/day

**Core innovation:** Stripe built "Ledger" - an internal double-entry accounting system that every dollar at Stripe flows through.

```
Stripe Ledger architecture:
  Source: every Charge, PaymentIntent, Transfer, Refund, Fee = events
  Ledger ingests: 5 billion events/day
  Verification: 99.99% of dollar volume verified within 4 hours

  Key design:
    Double-entry: every dollar movement = debit + credit journal entries
    Event-sourced: ledger is rebuilt from the event log
    Reconciliation: automated comparison vs payment networks

  PostgreSQL: primary data store (ACID transactions for journal entries)
  Kafka: event streaming (charge events -> ledger entries)
  Idempotency: every API call has idempotency key -> processed exactly once

Payment API evolution:
  2011: Charges API - simple, one-step charge
  2019: PaymentIntents - event-sourced multi-step flow (3DS, SCA)
  Today: 100B+ API calls/day, 13-year backward compatibility
```

**What Stripe does that most apps don't:**
- Idempotency key on EVERY write operation (no double charges)
- Reconciliation job runs continuously (finds discrepancies within hours)
- Dual-write: every event written to Kafka AND PostgreSQL (consistency)
- Envelope encryption: data key + key encryption key + HSM

---

## Monzo - The Modern Bank (1500 Microservices)

**Scale:** 10M+ customers UK, real-time banking

**Core architecture:** 1500+ microservices, each owning its domain.

```
Monzo's ledger service:
  Single service responsible for ALL money movements at Monzo
  Cassandra-backed (write-heavy, append-only ideal)
  Every debit/credit -> immutable journal entry in Cassandra
  No UPDATE, no DELETE - only INSERT
  Balance = computed from journal at read time

"Modern Banking in 1500 Microservices" (InfoQ talk):
  Every service is small: 1 team owns it
  Cassandra: handles the high write volume of a bank
  Kafka: event bus between all 1500 services

Core banking pattern:
  Customer pays merchant -> PaymentService -> 
    publishes PaymentAuthorized event -> Kafka
    LedgerService subscribes -> records journal entries
    NotificationService subscribes -> sends push notification
    FraudService subscribes -> runs fraud check

  No synchronous calls for these - all async events
  If notification service is down: payment still succeeds (decoupled)
```

**Key lesson from Monzo:** Each microservice has its own database (Cassandra). No shared databases. Boundaries enforced. The Ledger Service is the source of truth - everything else is a projection of ledger events.

---

## Wise (formerly TransferWise) - Cross-Border Money Movement

**Scale:** GBP 10B+ moved monthly, 170+ countries

**Core problem solved:** Banks charged 4-5% for international transfers. Wise charges 0.3-1%.

```
How Wise works (the "clever trick"):
  Traditional SWIFT transfer Alice (UK) -> Bob (Thailand):
    Alice -> UK bank -> Correspondent bank -> Thai bank -> Bob
    Cost: GBP 25 fee + 2-3% exchange rate markup
    Time: 2-3 business days

  Wise "local matching":
    Alice pays GBP 1000 into Wise's UK pool account (local transfer, free)
    Wise uses its existing Thai Baht pool to pay Bob in Thailand
    Net: Alice's GBP pool grows, Wise's THB pool shrinks
    Rebalancing: Wise uses FX market when pools are imbalanced

  No money crosses borders -> no SWIFT fees -> 0.3% fee instead of 4%

Database architecture:
  Multi-currency ledger: tracks pools in 50+ currencies simultaneously
  PostgreSQL: ACID transactions for balance movements
  Event sourcing: every FX conversion = event with exchange rate snapshot

  Balance invariant:
    SUM(customer liabilities in currency X) = Wise's pool in currency X
    Monitored every minute -> alert if imbalanced

Regulatory:
  Money Service Business licenses in 170+ countries
  Each country's funds held separately (ring-fenced)
  Segregated accounts: customer funds never commingled with Wise's operating funds
```

---

## The Common Patterns Across All Three

| Pattern | Stripe | Monzo | Wise |
|---|---|---|---|
| **Ledger** | Custom Ledger system | Cassandra ledger service | Multi-currency PostgreSQL ledger |
| **Accounting** | Double-entry | Double-entry | Double-entry |
| **Immutability** | Append-only events | No UPDATE/DELETE | Event sourcing |
| **Idempotency** | Idempotency keys | Kafka consumer deduplication | Request deduplication |
| **Reconciliation** | Automated, continuous | Daily ledger reconciliation | Real-time pool balance check |
| **Compliance** | PCI Level 1 | FCA regulated | 170+ country licenses |
| **Database** | PostgreSQL + Kafka | Cassandra + Kafka | PostgreSQL |
| **Security** | HSM, tokenization | mTLS, zero trust | HSM, segregated accounts |

---

## In Clean Architecture Terms

All three companies apply Clean Architecture to financial systems:

```
Stripe Ledger:
  Domain: Money, Account, JournalEntry, Transaction (entities)
  Use Cases: RecordCharge, ProcessRefund, ReconcileLedger
  Interfaces: IPaymentNetwork (port), ILedgerStore (port)
  Adapters: VisaAdapter, MastercardAdapter, PostgresLedgerStore

Monzo:
  Domain: Balance, Payment, Account (entities per microservice)
  Use Cases: AuthorizePayment, TransferFunds (one per service)
  Event Bus: IEventBus (port) -> KafkaEventBus (adapter)
  Services: each is an independent Clean Architecture application

Wise:
  Domain: CurrencyPool, FxConversion, CustomerTransfer (entities)
  Use Cases: InitiateTransfer, MatchLocalPayments, RebalancePools
  Adapters: FxMarketAdapter, LocalBankAdapter, RegulatorReportAdapter

The pattern: money and accounting rules in Domain layer
              payment networks and databases in Infrastructure layer
              Clean boundaries = swap infrastructure without changing rules
```
