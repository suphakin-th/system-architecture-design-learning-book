# Architecture 05 — Event Sourcing

---

## At a Glance

| | |
|---|---|
| **Type** | State as a sequence of events |
| **Complexity** | High |
| **Best for** | Audit-critical systems, financial ledgers, complex domain with history |
| **Avoid when** | Simple CRUD, no audit requirement, team unfamiliar with async |

---

## What Is It?

In traditional systems, you store the **current state**: `order.status = 'SHIPPED'`.

In Event Sourcing, you store the **events that led to that state**:
```
OrderCreated → ItemAdded → ItemAdded → OrderPlaced → PaymentCharged → OrderShipped
```

The current state is derived by **replaying** all events from the beginning.

**The event log IS the source of truth. The current state is a derived projection.**

---

## Diagram Reference
`./diagram.svg`

---

## How State Is Built

```typescript
class Order {
  id: string;
  status: OrderStatus = OrderStatus.NEW;
  items: OrderItem[] = [];
  totalAmount: number = 0;

  // Apply an event to evolve state (pure function, no side effects)
  apply(event: DomainEvent): void {
    if (event instanceof OrderCreatedEvent) {
      this.id = event.orderId;
      this.status = OrderStatus.NEW;
    }
    if (event instanceof ItemAddedEvent) {
      this.items.push(event.item);
      this.totalAmount += event.item.price;
    }
    if (event instanceof OrderPlacedEvent) {
      this.status = OrderStatus.PLACED;
    }
    if (event instanceof OrderShippedEvent) {
      this.status = OrderStatus.SHIPPED;
    }
  }

  // Rebuild state from event history
  static reconstitute(events: DomainEvent[]): Order {
    const order = new Order();
    events.forEach(e => order.apply(e));
    return order;
  }
}
```

---

## In Clean Architecture Terms

The **event store** is an outbound adapter (implements `IEventStore` port).
The **entity reconstitution** is pure domain logic.
**Projections** (read models) are built by consuming events — exactly like CQRS read models.

```typescript
// Port (defined in use case layer)
interface IEventStore {
  append(streamId: string, events: DomainEvent[], expectedVersion: number): Promise<void>;
  load(streamId: string): Promise<DomainEvent[]>;
}

// Use Case
class PlaceOrderUseCase {
  constructor(private eventStore: IEventStore, private eventBus: IEventBus) {}

  async execute(orderId: string) {
    // Load entity by replaying events
    const events = await this.eventStore.load(orderId);
    const order = Order.reconstitute(events);

    // Apply business rule
    const newEvent = order.place(); // returns OrderPlacedEvent

    // Append new event (optimistic concurrency via expectedVersion)
    await this.eventStore.append(orderId, [newEvent], events.length);

    // Publish for projections
    await this.eventBus.publish(newEvent);
  }
}
```

---

## Projections (Read Models)

Projections subscribe to events and build optimized read models:

```typescript
class OrderProjection {
  async handle(event: OrderPlacedEvent) {
    await this.readDb.upsert('order_summaries', {
      orderId: event.orderId,
      status: 'PLACED',
      placedAt: event.occurredAt,
    });
  }
}
```

You can **replay all events** to rebuild any projection from scratch — powerful for debugging and new feature rollout.

---

## Resource Consumption

| Resource | Usage | Notes |
|---|---|---|
| **CPU** | Medium | Event replay on load; projection rebuilds on schema change |
| **Memory** | Medium | Event stream loaded into memory for replay; use snapshots for long-lived aggregates |
| **Disk** | High | Events are append-only and never deleted; grows forever |
| **Write latency** | Very Low | Append to event log = extremely fast (sequential write) |
| **Read latency** | Variable | Rebuild state from events (mitigated by snapshots + projections) |
| **Storage growth** | Linear | Every state change = one event; long-lived aggregates need snapshotting |

### Snapshots (Performance Optimization)
For aggregates with 10,000+ events, storing a snapshot avoids full replay:
```
Snapshot at v1000 + events 1001-1050 → rebuilt state
Instead of: events 1-1050 → rebuilt state
```

---

## Benefits

1. **Complete audit trail** — every state change is recorded with WHO, WHAT, WHEN, WHY
2. **Time travel** — rebuild the state as it was at any point in time
3. **No update/delete** — append-only is the simplest possible DB operation
4. **Debug production bugs** — replay exact event sequence that caused the bug
5. **Multiple projections** — rebuild any view of the data from the same events
6. **Natural integration** — events are publishable to other services (EDA built-in)
7. **Optimistic concurrency** — version-based conflict detection without locks

---

## Problems It Solves Best

| Problem | Why Event Sourcing Wins |
|---|---|
| "Regulators require full audit trail of every balance change" | Every event is in the log, immutable |
| "We need to reproduce a bug that happened 3 days ago" | Replay events from that point in time |
| "We need to calculate account balance at end of each month" | Replay events up to month-end |
| "New feature needs historical data we didn't store" | Replay all events with new projection logic |
| "We need to integrate with 5 downstream systems" | Publish every event; each system projects what it needs |

---

## Costs / Tradeoffs

1. **Complexity** — significantly harder to reason about than "just update the row"
2. **Event schema evolution** — events are immutable; changing schema requires versioning strategy
3. **Storage growth** — event log grows forever; need retention policies and snapshots
4. **Performance for long histories** — replay of 10,000 events is slow without snapshots
5. **Querying is hard** — can't do `SELECT * FROM orders WHERE status='placed'`; need projections
6. **Not good for simple CRUD** — massive overhead for create/read/update/delete with no history needed

---

## Big Tech Examples

### Stripe
- **Architecture:** Entire financial ledger is event-sourced
- **Events:** PaymentIntentCreated, ChargeAttempted, ChargeSucceeded, RefundCreated
- **Good at:** Perfect audit trail for every cent; replay events to reconcile accounts; SOX compliance
- **Result:** Can reproduce exact state of any payment at any point in its history

### Shopify
- **Architecture:** Order state machine is event-sourced
- **Events:** OrderCreated, FulfillmentCreated, ItemShipped, OrderClosed
- **Good at:** Merchants can see every state change; Shopify Support can debug any order precisely

### Axon Framework (used by many banks)
- **Architecture:** Java framework specifically for Event Sourcing + CQRS
- **Events:** AccountOpened, MoneyDeposited, MoneyWithdrawn, AccountClosed
- **Companies using it:** ABN AMRO Bank, ING Bank
- **Good at:** Banking ledgers where every transaction must be auditable for 7+ years

### EventStore DB (open source event store)
- **Used by:** Stack Exchange, Walmart, and various fintech companies
- **Architecture:** Purpose-built event store with built-in projection support
- **Good at:** Complex domain modeling where history matters (insurance claims, medical records)

---

## Key Takeaway

> Event Sourcing is the ultimate application of Clean Architecture's principle that "entities contain business rules." The entity doesn't store state — it accumulates events and derives state from them. The event store is just a port+adapter. The result is a complete audit trail and time-travel capability at the cost of significant complexity.
