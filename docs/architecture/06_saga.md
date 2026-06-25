# Architecture 06 - Saga Pattern

---

## At a Glance

| | |
|---|---|
| **Type** | Distributed transaction management |
| **Complexity** | High |
| **Best for** | Long-running transactions across multiple microservices |
| **Avoid when** | Single-service transactions (use local ACID), monolith (use DB transactions) |

---

## What Is It?

When an operation spans multiple microservices (each with their own DB), a traditional ACID transaction is impossible. The Saga pattern manages **distributed transactions** as a sequence of local transactions, with **compensating transactions** to undo steps if something fails.

A Saga is a **use case** that spans service boundaries.

**Two styles:**
1. **Choreography** - each service reacts to events; no central coordinator
2. **Orchestration** - a dedicated Saga Orchestrator tells each service what to do

---

## Diagram Reference
`./diagram.svg`

---

## Example: Place Order Saga

### Choreography (Event-driven)
```
Order Svc: OrderCreated event published
 -> Payment Svc: PaymentReserved event published
 -> Inventory Svc: StockReserved event published
 -> Shipping Svc: ShipmentCreated event published
 -> Order Svc: OrderConfirmed event published [OK]

If any step fails -> publish compensating events:
  StockReservationFailed -> Payment Svc: PaymentCancelled -> Order Svc: OrderCancelled
```

### Orchestration (Command-driven)
```
SagaOrchestrator:
  1. Send Reserve Payment command -> await PaymentReserved or PaymentFailed
  2. If OK: Send Reserve Stock command -> await StockReserved or StockFailed
  3. If OK: Send Create Shipment command -> await ShipmentCreated or ShipmentFailed
  4. If any fail: send compensating commands in reverse order
```

---

## In Clean Architecture Terms

The Saga Orchestrator is a **Use Case** that depends on interfaces (ports) for each service:

```typescript
// Saga Use Case
class PlaceOrderSaga {
  constructor(
    private payment: IPaymentServicePort,
    private inventory: IInventoryServicePort,
    private shipping: IShippingServicePort,
    private orderRepo: IOrderRepository,
    private sagaRepo: ISagaStateRepository
  ) {}

  async execute(orderId: string) {
    const saga = await this.sagaRepo.load(orderId) ?? new SagaState(orderId);

    try {
      saga.step = 'RESERVING_PAYMENT';
      await this.payment.reserve(orderId);          // via interface/port

      saga.step = 'RESERVING_STOCK';
      await this.inventory.reserve(orderId);        // via interface/port

      saga.step = 'CREATING_SHIPMENT';
      await this.shipping.createShipment(orderId);  // via interface/port

      saga.step = 'COMPLETED';
    } catch (error) {
      await this.compensate(saga, error);           // undo in reverse order
    }

    await this.sagaRepo.save(saga);
  }

  private async compensate(saga: SagaState, error: Error) {
    if (saga.step >= 'RESERVING_STOCK')
      await this.inventory.releaseReservation(saga.orderId);
    if (saga.step >= 'RESERVING_PAYMENT')
      await this.payment.cancelReservation(saga.orderId);
    await this.orderRepo.markFailed(saga.orderId, error.message);
  }
}
```

---

## Resource Consumption

| Resource | Usage | Notes |
|---|---|---|
| **CPU** | Medium | Orchestrator + each service step |
| **Memory** | Low | Saga state is persisted, not held in memory across steps |
| **Network** | High | Each step = at least one network call + one event publish |
| **Latency** | High | N sequential network calls; can be seconds for complex sagas |
| **Storage** | Low-Medium | Saga state table (one row per in-flight transaction) |
| **Complexity** | Very High | Compensating logic + idempotency + failure recovery = lots of code |

---

## Benefits

1. **Distributed transaction management** without two-phase commit (2PC) locks
2. **Each service stays autonomous** - no shared lock across service boundaries
3. **Long-running transactions** - saga can span hours (e.g., hotel booking + flight + payment)
4. **Visibility** - saga state shows exactly where a transaction is stuck
5. **Recovery** - can resume a failed saga from its last known-good step
6. **Compensating actions** - clean rollback without DB-level rollback

---

## Problems It Solves Best

| Problem | Why Saga Wins |
|---|---|
| "Booking a flight requires reserving seat + charging card + notifying airline - any can fail" | Saga with compensation undoes each step |
| "Order requires payment + inventory + shipping - all three must succeed" | Orchestration saga coordinates all three |
| "Payment succeeded but inventory failed - we must refund" | Compensating transaction: cancelPayment() |
| "Transaction takes 10 minutes (waiting for human approval)" | Long-running saga stores state between steps |

---

## Costs / Tradeoffs

1. **No atomicity** - steps succeed at different times; data is temporarily inconsistent
2. **Complex compensating logic** - every step needs a corresponding undo
3. **Idempotency required** - steps may be retried; must be safe to run twice
4. **Saga state management** - need a DB table or event store for saga state
5. **Debugging is hard** - where in the saga did it fail? Need distributed tracing
6. **Testing is very hard** - need to simulate partial failures across services

---

## Big Tech Examples

### Uber (Trip Booking Saga)
- **Architecture:** Orchestration-based saga for trip lifecycle
- **Steps:** Match driver -> Calculate price -> Reserve trip -> Process payment -> Start trip
- **Compensation:** If payment fails -> release driver reservation -> cancel trip
- **Good at:** Handles driver cancellation mid-saga gracefully with compensation

### Amazon (Order Fulfillment Saga)
- **Architecture:** Choreography-based saga via SQS/SNS
- **Steps:** OrderPlaced -> PaymentCharged -> InventoryReserved -> ShipmentCreated
- **Compensation:** If stock unavailable -> PaymentRefunded -> OrderCancelled
- **Good at:** Handles millions of sagas/day with individual step retries

### Airbnb (Booking Saga)
- **Architecture:** Orchestration saga for accommodation booking
- **Steps:** Hold listing -> Authorize payment -> Confirm booking -> Notify host
- **Compensation:** If host declines -> release hold -> full refund
- **Good at:** Multi-day hold periods between authorization and confirmation

### Klarna (Buy Now Pay Later Saga)
- **Architecture:** Saga for BNPL transaction
- **Steps:** Credit check -> Merchant payment -> Create loan -> Set repayment schedule
- **Compensation:** If credit check fails -> decline -> notify merchant
- **Good at:** Complex financial flow spanning 4 separate microservices atomically

---

## Choreography vs Orchestration

| | Choreography | Orchestration |
|---|---|---|
| **Coordinator** | None - services react to events | Dedicated saga orchestrator |
| **Coupling** | Services know about events, not each other | Services know only their commands |
| **Visibility** | Hard - must trace events across services | Easy - orchestrator has full picture |
| **Best for** | Simple sagas with 3-4 steps | Complex sagas with many branches |
| **Risk** | Cyclic dependencies between events | Single point of failure (orchestrator) |

---

## Key Takeaway

> The Saga pattern is a Use Case that spans service boundaries. In Clean Architecture terms: the orchestrator use case depends on interfaces (ports) for each service; compensating transactions are the "undo" business rules. This replaces ACID transactions with explicit, recoverable distributed coordination.
