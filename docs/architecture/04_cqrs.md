# Architecture 04 — CQRS (Command Query Responsibility Segregation)

---

## At a Glance

| | |
|---|---|
| **Type** | Read/Write model separation |
| **Complexity** | Medium |
| **Best for** | High read:write ratio imbalance, complex query needs, audit-heavy systems |
| **Avoid when** | Simple CRUD, small team, no read performance problem |

---

## What Is It?

CQRS separates **Commands** (write operations that change state) from **Queries** (read operations that return data). They use separate models, sometimes separate databases.

**Core insight:** In most systems, reads outnumber writes by 10:1 to 100:1. They have very different optimization needs. Why use the same model for both?

```
Command: PlaceOrder  → changes state → writes to write model DB
Query:   GetOrderById → reads state → reads from read model DB (optimized for fast reads)
```

---

## Diagram Reference
`./diagram.svg`

---

## Levels of CQRS

### Level 1 — Same DB, Separate Models (Simple)
```
                    ┌── CommandHandler → Domain Entities → Write Model (normalized DB)
Application ───────┤
                    └── QueryHandler → Query DTO → Read Model (same DB, different view/table)
```

### Level 2 — Separate Read DB (Full CQRS)
```
                    ┌── CommandHandler → Domain Entities → Write DB (PostgreSQL, normalized)
Application ───────┤                          │
                    │                    [Event / Sync]
                    │                          │
                    └── QueryHandler → Read DB (Elasticsearch, Redis, denormalized views)
```

### Level 3 — CQRS + Event Sourcing (Advanced)
Commands → Events → Event Store → projections → Read Models
(see Pattern 05)

---

## In Clean Architecture Terms

CQRS splits Use Cases into two types:

```typescript
// Command Use Case — WRITES, returns minimal data (just ID or void)
class PlaceOrderCommand {
  constructor(
    public readonly customerId: string,
    public readonly items: OrderItemDto[]
  ) {}
}

class PlaceOrderCommandHandler {
  constructor(
    private orders: IOrderRepository,       // write model interface
    private eventBus: IEventBus
  ) {}

  async execute(cmd: PlaceOrderCommand): Promise<string> {
    const order = Order.create(cmd.customerId, cmd.items); // entity
    await this.orders.save(order);
    await this.eventBus.publish(new OrderPlacedEvent(order));
    return order.id;  // only return the ID
  }
}

// Query Use Case — READS, never changes state, optimized DTO
class GetOrderDetailsQuery {
  constructor(public readonly orderId: string) {}
}

class GetOrderDetailsQueryHandler {
  constructor(private readDb: IOrderReadRepository) {} // read model interface

  async execute(query: GetOrderDetailsQuery): Promise<OrderDetailsDto> {
    // Goes directly to read model — may be a denormalized view, Redis, or Elasticsearch
    return this.readDb.getOrderDetails(query.orderId);
  }
}
```

**Key rule:** Queries NEVER modify state. Commands NEVER return data beyond confirmation.

---

## Read Model Design

The read model is optimized for how data is DISPLAYED, not how it's stored.

```sql
-- Write model: 3NF normalized
orders (id, customer_id, status, created_at)
order_items (id, order_id, product_id, quantity, price)
customers (id, name, email, shipping_address)
products (id, name, sku, description)

-- Read model: denormalized for the "Order Details" page
order_details_view (
  order_id, status, created_at,
  customer_name, customer_email, shipping_address,
  items: [{ product_name, sku, quantity, price, subtotal }],
  total_amount
)
-- This is one query, no joins needed = fast!
```

---

## Resource Consumption

| Resource | Usage | Notes |
|---|---|---|
| **CPU** | Low-Medium | Commands: entity validation; Queries: simple DB lookups |
| **Memory** | Low-Medium | Read model can be cached aggressively (Redis) |
| **Network** | Medium | Sync between write → read DB adds network hops |
| **Disk** | 2x+ | Separate write + read databases; data duplication |
| **Read latency** | Very Low | Read model is purpose-built; often sub-millisecond with Redis |
| **Write latency** | Low | Write model is normalized, simple writes |
| **Consistency** | Eventual | Read model may lag by 10-100ms behind writes |

---

## Benefits

1. **Independent scaling** — scale read replicas ×10; write DB stays ×1
2. **Optimized queries** — read model is shaped exactly for the UI; no joins, no transforms
3. **Clear separation** — can't accidentally write in a read handler; impossible by design
4. **Audit-friendly** — commands are explicit intents; log them all
5. **Multiple read models** — OrderDetailsView for web, OrderSummaryView for mobile, both from same write commands
6. **Caching is easy** — read models are immutable until next command; easy to cache

---

## Problems It Solves Best

| Problem | Why CQRS Wins |
|---|---|
| "Dashboard query JOINs 8 tables and takes 2 seconds" | Build a denormalized read model; one query |
| "The report page slows down order creation (same DB load)" | Separate read DB; writes are not affected |
| "Mobile needs different fields than web" | Two read models from same commands |
| "We need to support full-text search" | Read model = Elasticsearch; write model = PostgreSQL |
| "Who placed this order? What changed? Full audit trail?" | Commands are explicit, named, and logged |

---

## Costs / Tradeoffs

1. **Eventual consistency** — read model may lag; "I just placed an order but it doesn't show yet"
2. **Complexity** — two models, two DBs, synchronization logic
3. **More code** — separate command handlers, query handlers, DTOs for each direction
4. **Sync mechanism** — need to sync write → read DB (events, CDC, polling)
5. **Not worth it for CRUD** — if reads and writes have similar load, CQRS is overhead

---

## Big Tech Examples

### Microsoft (Azure)
- **Architecture:** CQRS is a core Azure architecture pattern (documented in Azure Architecture Center)
- **Use case:** Event-sourced microservices where write = event store, read = materialized views
- **Tools:** Azure Cosmos DB for read model (global distribution), SQL for write model

### Walmart
- **Architecture:** CQRS for e-commerce catalog
- **Write model:** Product master data in normalized DB (SKU, pricing, inventory)
- **Read models:** Elasticsearch for search, CDN-cached JSON for product pages, Redis for inventory count
- **Good at:** Search returns in <50ms while product data updates propagate asynchronously

### Booking.com
- **Architecture:** CQRS for availability and pricing
- **Write model:** Availability updates from hotels (normalized)
- **Read model:** Denormalized availability calendar for search results
- **Good at:** 1M+ queries/day on read model while write model processes hotel updates

### LinkedIn
- **Architecture:** CQRS for activity streams (feed)
- **Write model:** User action (post, like, connect) → normalized event store
- **Read model:** Pre-computed feed per user (Redis sorted sets)
- **Good at:** Reading your feed = one Redis lookup; writing a post = async fan-out

---

## Key Takeaway

> CQRS takes the Clean Architecture principle of "use cases are the app's verbs" and splits those verbs into two kinds: Commands (change state) and Queries (read state). This split allows each side to be optimized independently — writes for consistency and correctness, reads for speed and flexibility.
