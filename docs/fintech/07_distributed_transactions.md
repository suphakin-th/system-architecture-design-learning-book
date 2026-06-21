# Distributed Transactions — 2PC vs Saga

> "In a distributed system, the only thing worse than a failed transaction is a half-completed one. Money left in limbo between services." — Senior fintech architect

---

## At a Glance

| | |
|---|---|
| **Problem** | A business operation spans multiple microservices — each with its own DB |
| **Old solution** | Two-Phase Commit (2PC) — works, but blocks; not suitable for microservices |
| **Modern solution** | Saga Pattern — sequence of local transactions with compensating actions |

---

## Why You Can't Use a Regular Transaction Across Services

```
PlaceOrder needs to:
  1. Deduct inventory   → InventoryDB (PostgreSQL)
  2. Charge payment     → PaymentDB   (PostgreSQL)
  3. Create order       → OrderDB     (PostgreSQL)

Normal SQL transaction:
  BEGIN;
    UPDATE inventory ...  ← on InventoryDB
    INSERT INTO payments  ← on PaymentDB (DIFFERENT server!)
    INSERT INTO orders    ← on OrderDB   (DIFFERENT server!)
  COMMIT;

This is IMPOSSIBLE — a SQL transaction cannot span different database servers.
The COMMIT must be atomic, but each DB is independent.
```

---

## Option 1: Two-Phase Commit (2PC)

### How It Works

```
Phase 1 — PREPARE:
  Coordinator → InventoryDB:  "Can you deduct stock? Lock it."
  InventoryDB: locks the row, replies "PREPARED"

  Coordinator → PaymentDB:    "Can you charge $99? Lock it."
  PaymentDB:   holds the auth, replies "PREPARED"

  Coordinator → OrderDB:      "Can you create the order? Lock it."
  OrderDB:     locks, replies "PREPARED"

Phase 2 — COMMIT (all said PREPARED):
  Coordinator → all three: "COMMIT"
  All three commit their local transactions simultaneously.

If any said "ABORT" in Phase 1:
  Coordinator → all three: "ROLLBACK"
  All three roll back.
```

### Why 2PC Is Rarely Used in Microservices

```
Problem 1: Blocking
  All participants hold locks during Phase 1 AND Phase 2.
  If coordinator crashes between phases:
    All participants hold locks FOREVER (deadlock until manual intervention).

Problem 2: Latency
  2 round trips to all N participants before any commit.
  For N=5 services: 10 network hops minimum.
  At 10ms per hop: 100ms minimum added to every transaction.

Problem 3: Not Partition-Tolerant
  If network partition occurs during commit:
    Participants wait indefinitely — entire system stalls.

Real-world verdict:
  2PC is used in: traditional enterprise software (Java EE, IBM MQ XA)
  2PC is NOT used in: modern microservices (Netflix, Uber, Amazon)
  They all use: Saga Pattern instead.
```

---

## Option 2: Saga Pattern (The Modern Standard)

A Saga splits a distributed transaction into a sequence of local transactions. Each step has a **compensating transaction** to undo it if a later step fails.

### Orchestration Saga (Recommended)

```
SagaOrchestrator (a Use Case):
  Step 1: Send "ReserveInventory" command → InventoryService
          Success → proceed   Failure → end (nothing to undo)

  Step 2: Send "ChargePayment" command → PaymentService
          Success → proceed
          Failure → send "ReleaseInventory" (compensate Step 1)

  Step 3: Send "CreateOrder" command → OrderService
          Success → Saga complete ✓
          Failure → send "CancelPayment" + "ReleaseInventory" (compensate both)

Each step:
  - Is a LOCAL ACID transaction (within one service's DB)
  - Has a compensating action (to undo if needed)
  - Is idempotent (safe to retry)
```

### Compensation = Business Undo, Not DB Rollback

```typescript
// Compensating transaction for "charge payment"
async function cancelPayment(orderId: string): Promise<void> {
  // This is a NEW transaction that REVERSES the charge
  // NOT a database ROLLBACK — the original charge is already committed
  const payment = await paymentRepo.findByOrderId(orderId);
  const refund = Refund.create(payment.id, payment.amount, 'saga_compensation');
  await paymentRepo.saveRefund(refund);
  await eventBus.publish(new PaymentCancelledEvent(orderId));
}
// The payment record stays in the DB (immutable audit trail)
// The refund record is a NEW entry (double-entry accounting principle)
```

### Using Temporal/Cadence for Long-Running Sagas

```typescript
// Temporal workflow = Saga Orchestrator with durability
// If server crashes mid-saga → Temporal replays from last checkpoint
import { defineWorkflow } from '@temporalio/workflow';

export const placeOrderWorkflow = defineWorkflow(async (orderId: string) => {
  // Step 1
  await reserveInventory(orderId);

  try {
    // Step 2
    await chargePayment(orderId);
  } catch (error) {
    await releaseInventory(orderId);  // compensate step 1
    throw error;
  }

  try {
    // Step 3
    await createShipment(orderId);
  } catch (error) {
    await cancelPayment(orderId);     // compensate step 2
    await releaseInventory(orderId);  // compensate step 1
    throw error;
  }
});
// If server restarts: Temporal replays the workflow from the checkpoint
// No money gets stuck between services
```

---

## 2PC vs Saga Comparison

| | 2PC | Saga |
|---|---|---|
| **Consistency** | Strong (atomic) | Eventual |
| **Availability** | Low (locks held) | High (no distributed locks) |
| **Latency** | High (2 round trips) | Low (async steps) |
| **Failure handling** | Hard (coordinator crash = stuck) | Explicit compensation |
| **Suitable for** | Single DB, same network | Microservices, cloud |
| **Used by** | Legacy enterprise (JEE, IBM MQ) | Netflix, Uber, Amazon |

---

## Real-World Examples

| Company | Saga Type | Details |
|---|---|---|
| **Uber** | Orchestration | Trip saga: match → price → authorize → start → end → charge → payout |
| **Amazon** | Choreography (SQS/SNS) | Order saga: placed → payment → inventory → shipping |
| **Airbnb** | Orchestration | Booking saga: hold listing → auth payment → confirm → notify host |
| **Klarna** | Orchestration | BNPL saga: credit check → merchant payment → create loan → schedule repayment |

---

## In Clean Architecture Terms

```
SagaOrchestrator = Use Case (spans multiple service ports)

PlaceOrderSaga depends on:
  IInventoryService (port) → InventoryServiceClient (adapter → HTTP/gRPC)
  IPaymentService   (port) → PaymentServiceClient   (adapter → HTTP/gRPC)
  IShippingService  (port) → ShippingServiceClient  (adapter → HTTP/gRPC)
  ISagaRepository   (port) → SagaRepositoryPostgres (adapter → DB for saga state)

The saga state (current step, which compensations ran) is persisted.
If the orchestrator restarts: reload state → continue from last step.

Business rule "inventory must be reserved before payment is charged"
lives in the saga use case — not in the infrastructure adapters.
```
