# Architecture 03 — Event-Driven Architecture (EDA)

---

## At a Glance

| | |
|---|---|
| **Type** | Async communication via events |
| **Complexity** | Medium-High |
| **Best for** | Decoupled workflows, real-time reactions, audit trails |
| **Avoid when** | Simple CRUD, strong consistency required, team unfamiliar with async |

---

## What Is It?

Event-Driven Architecture (EDA) is a style where components communicate by **producing and consuming events** rather than making direct calls to each other.

An **event** is a record of something that happened: `OrderPlaced`, `PaymentCharged`, `UserRegistered`. It's immutable — it happened, it can't be un-happened.

**Producers** emit events. **Consumers** react to events. Neither knows about the other.

---

## Diagram Reference
`./diagram.svg`

---

## Three EDA Patterns

### 1. Event Notification
Fire-and-forget. Publisher emits event; subscriber does something.
```
Order Service: "OrderPlaced" →→→ Notification Service: sends email
```

### 2. Event-Carried State Transfer
Event contains ALL the data the consumer needs (no need to query back).
```
OrderPlaced { orderId, userId, items[], totalAmount, shippingAddress }
→ Inventory Service: deducts stock (no need to call Order Service back)
```

### 3. Event Sourcing (see Pattern 05)
Events are the source of truth; state is derived by replaying them.

---

## Structure

```
[Order Service]
    │  publishes
    ▼
[Message Broker] — Kafka / RabbitMQ / AWS SNS+SQS
    │
    ├──► [Payment Service]       subscribes to OrderPlaced → charges card
    │        │ publishes PaymentCharged
    │        ▼
    │    [Order Service]         subscribes to PaymentCharged → marks paid
    │
    ├──► [Inventory Service]     subscribes to OrderPlaced → reserves stock
    │
    ├──► [Notification Service]  subscribes to OrderPlaced → sends email
    │
    └──► [Analytics Service]     subscribes to all → writes to data warehouse
```

---

## In Clean Architecture Terms

The **message bus is an outbound port/adapter**. The use case depends on an interface, not on Kafka directly.

```typescript
// Use Case Layer — defines the port
interface IEventBus {
  publish(event: DomainEvent): Promise<void>;
}

// Use Case — depends on interface
class PlaceOrderUseCase {
  constructor(
    private orders: IOrderRepository,
    private eventBus: IEventBus        // ← interface, not Kafka
  ) {}

  async execute(req: PlaceOrderRequest) {
    const order = await this.orders.findById(req.orderId);
    order.place();
    await this.orders.save(order);
    await this.eventBus.publish(new OrderPlacedEvent(order)); // ← event crosses boundary
  }
}

// Infrastructure Layer — Kafka adapter
class KafkaEventBus implements IEventBus {
  async publish(event: DomainEvent) {
    await this.producer.send({
      topic: event.type,
      messages: [{ value: JSON.stringify(event) }]
    });
  }
}
```

**The use case doesn't know about Kafka. Swap Kafka for RabbitMQ — only the adapter changes.**

---

## Event Schema Design

Events should be:
- **Immutable** — past tense name, never updated
- **Self-contained** — carry all data consumers need
- **Versioned** — include schema version for backward compatibility

```typescript
interface OrderPlacedEvent {
  type: 'OrderPlaced';
  version: '1.0';
  occurredAt: string;          // ISO 8601
  orderId: string;
  customerId: string;
  items: { productId: string; quantity: number; price: number }[];
  totalAmount: number;
  currency: 'THB' | 'USD';
}
```

---

## Resource Consumption

| Resource | Usage | Notes |
|---|---|---|
| **CPU** | Medium | Event serialization/deserialization overhead |
| **Memory** | Medium | Consumer groups hold message offsets in memory |
| **Network** | Medium | Events flow through broker; not direct service-to-service |
| **Disk** | High (Kafka) | Kafka persists events to disk by default (log retention) |
| **Latency** | Higher than sync | Async by nature; not suitable for "I need the answer NOW" |
| **Ops cost** | Medium-High | Kafka cluster management adds complexity |

---

## Benefits

1. **Loose coupling** — Order Service doesn't know Notification Service exists
2. **Scalability** — consumers scale independently based on their queue depth
3. **Resilience** — if Notification Service is down, events queue up and process when it recovers
4. **Audit trail** — event log IS the audit log; see every state change
5. **New consumers are free** — add Analytics Service without changing Order Service
6. **Temporal decoupling** — producer and consumer don't need to run at the same time

---

## Problems It Solves Best

| Problem | Why EDA Wins |
|---|---|
| "When an order is placed, 5 different things must happen" | Fan-out to 5 consumers; Order Svc knows none of them |
| "Notification Service goes down and we lose emails" | Events queue in broker; consumed when service recovers |
| "We need a full audit trail of every state change" | Event log is the audit trail |
| "Adding a new downstream workflow must not touch the producer" | New consumer subscribes; producer unchanged |
| "Real-time dashboards showing sales as they happen" | Analytics subscribes to all events; pushes to dashboard |

---

## Costs / Tradeoffs

1. **Eventual consistency** — consumer may be 100ms behind producer; no strong consistency
2. **Hard to debug** — trace a bug across 5 async consumers; need distributed tracing
3. **Event ordering** — Kafka guarantees order per partition; across partitions, no guarantee
4. **Idempotency required** — consumers must handle duplicate events (at-least-once delivery)
5. **Schema evolution** — changing event schema breaks consumers; need versioning strategy
6. **No immediate feedback** — can't easily tell the user "your email failed to send"

---

## Big Tech Examples

### LinkedIn
- **Architecture:** Kafka was invented at LinkedIn (2010)
- **Events:** 7 trillion messages/day; activity streams, metrics, log aggregation
- **Use case:** User clicks "Like" → ActivityEvent → feed update, notification, analytics
- **Good at:** Planet-scale fan-out; one event → hundreds of consumers

### Uber
- **Architecture:** Kafka for real-time trip events
- **Events:** TripStarted, DriverLocationUpdated (every 4 seconds per driver!), TripEnded, PaymentCharged
- **Volume:** Billions of location events/day
- **Good at:** Real-time surge pricing reacts to TripStarted/Ended events

### Airbnb
- **Architecture:** Kafka + Flink for stream processing
- **Events:** BookingCreated, MessageSent, ReviewPosted
- **Good at:** Fraud detection subscribes to BookingCreated events, runs ML model, flags suspicious patterns

### Twitter (X)
- **Architecture:** EventBus (internal Kafka wrapper)
- **Events:** TweetPosted → fan-out to followers' timelines, trending algorithm, search index
- **Challenge:** Celebrity tweet fan-out to 10M followers = massive write amplification
- **Solution:** Lazy fan-out for high-follower accounts (pull instead of push)

### AWS
- **Architecture:** EventBridge, SNS, SQS — the backbone of AWS event-driven workflows
- **Pattern:** Lambda reacts to S3 upload events, DynamoDB stream events, SQS messages
- **Good at:** Serverless workflows where each step reacts to the previous step's event

---

## Key Takeaway

> Event-Driven Architecture is the Clean Architecture "IEventBus port" made real at scale. The use case publishes an event through an interface; the broker fans it out to any number of independent consumers. This gives radical decoupling and resilience, at the cost of eventual consistency and operational complexity.
