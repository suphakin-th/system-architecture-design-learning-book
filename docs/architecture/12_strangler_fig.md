# Architecture 12 - Strangler Fig Pattern

---

## At a Glance

| | |
|---|---|
| **Type** | Incremental migration strategy |
| **Complexity** | Medium |
| **Best for** | Migrating a legacy monolith to microservices or Clean Architecture |
| **Avoid when** | Greenfield project (just build it right from the start) |

---

## What Is It?

Named after the Strangler Fig tree, which grows around an existing tree and gradually replaces it, the Strangler Fig Pattern migrates a legacy system incrementally - **never requiring a big-bang rewrite**.

New functionality is built in the new system. Old functionality is migrated piece by piece. The old system "strangles" until it's replaced entirely.

**Key insight:** You never freeze the old system. Both systems run simultaneously. Traffic is gradually shifted.

---

## Diagram Reference
`./diagram.svg`

---

## Migration Process

```
Phase 1: Facade (Week 1-2)
  All traffic -> Legacy Monolith (unchanged)
  Introduce a facade (API Gateway) in front of the monolith

Phase 2: Extract first module (Month 1-3)
  New "Search Service" built with Clean Architecture
  Gateway routes: GET /search/* -> Search Service
  Gateway routes: everything else -> Legacy Monolith

Phase 3: Extract second module (Month 3-6)
  New "Order Service" built with Clean Architecture
  Gateway routes: /orders/* -> Order Service
  Gateway routes: /search/* -> Search Service
  Gateway routes: everything else -> Legacy Monolith

...repeat...

Phase N: Legacy is empty (Month 12-24)
  All traffic -> New services
  Legacy Monolith is decommissioned
```

---

## In Clean Architecture Terms

The Strangler Fig is a **migration path**, not an architectural pattern per se. But it works beautifully with Clean Architecture because:

1. You build each new service with Clean Architecture from day one
2. The interfaces (ports) you defined in the legacy system's "better parts" become the contracts for new services
3. The API Gateway facade makes switching transparent to clients

```typescript
// Step 1: Facade - just proxies everything to legacy
class LegacyFacadeGateway implements IOrderRepository {
  async findById(id: string): Promise<Order> {
    const res = await axios.get(`http://legacy-monolith/orders/${id}`);
    return OrderMapper.toDomain(res.data); // adapts legacy response to domain model
  }
}

// Step 2: New service built - same interface
class OrderServiceClient implements IOrderRepository {
  async findById(id: string): Promise<Order> {
    const res = await axios.get(`http://order-service/orders/${id}`);
    return OrderMapper.toDomain(res.data);
  }
}

// In composition root - just swap the adapter, use case is unchanged
const orderRepo = featureFlag('use-new-order-service')
  ? new OrderServiceClient()
  : new LegacyFacadeGateway();
```

**The use case code doesn't change at all. Only the adapter swaps.**

---

## Migration Strategies

### By Feature
Extract features one at a time:
```
Month 1: Extract Search
Month 2: Extract User Auth
Month 3: Extract Order Management
```

### By Bounded Context (DDD)
Extract full business domains:
```
Quarter 1: Extract Payments domain
Quarter 2: Extract Catalog domain
Quarter 3: Extract Fulfillment domain
```

### Strangler with Dark Launch
Run new service in shadow mode first:
```
Request -> Legacy (handles it for real)
 -> New Service (handles it silently, compare outputs)
If outputs match -> shift traffic to new service
```

---

## Resource Consumption

| Resource | During Migration | After Migration |
|---|---|---|
| CPU | 1.5x-2x (both systems running) | Normal |
| Memory | 1.5x-2x (both systems in memory) | Normal |
| Network | Extra hop through facade | Normal |
| Ops overhead | Highest (running two systems) | Normal |
| Risk | Low per step (each extraction is small) | None |

---

## Benefits

1. **No big bang rewrite** - most rewrites fail; incremental migration succeeds
2. **Business keeps running** - production never stops; users see no interruption
3. **Each step is reversible** - if new service has bugs, route traffic back to legacy instantly
4. **Team learns incrementally** - team learns new patterns (Clean Architecture) gradually
5. **Risk is contained** - one module at a time; failure scope is tiny
6. **Validate the new architecture** - prove Clean Architecture works before committing 100%

---

## Problems It Solves Best

| Problem | Why Strangler Fig Wins |
| "We have a 10-year-old Rails monolith we need to modernize" | Extract one module per quarter; never freeze the system |
| "Our last big rewrite failed because it took 18 months and features changed" | Incremental; stay synced with feature requests |
| "Management won't approve 6 months of no new features for a rewrite" | Strangler adds features in new system; legacy handles the rest |
| "We need to migrate from PHP to Go but can't stop development" | Route new features to Go services; migrate old PHP features over time |

---

## Costs / Tradeoffs

1. **Running two systems** - CPU/memory/ops for both legacy + new system during migration
2. **Data synchronization** - need to keep legacy DB and new service DB in sync during transition
3. **Feature parity** - must maintain features in legacy while building new system
4. **Long migration duration** - 12-24 months for large monoliths
5. **Team discipline required** - must resist adding features to legacy (pulls back to old patterns)

---

## Big Tech Examples

### Amazon (OBIDOS -> SOA, 2001-2006)
- **Legacy:** OBIDOS monolith in Perl serving the entire Amazon.com
- **Migration:** Gradually extracted services: catalog, pricing, checkout, recommendations
- **Facade:** Internal service boundaries, then API Gateway as external facade
- **Duration:** ~5 years of gradual migration
- **Good at:** Amazon never stopped selling books during the migration

### Shopify (Monolith -> Modular, 2016-present)
- **Legacy:** 1M+ line Rails monolith
- **Migration:** Not full microservices - instead, "modular monolith" (Components)
- **Strategy:** Extract high-value modules into clearly bounded components with interfaces
- **Good at:** Shopify still ships Rails monolith but with clean internal boundaries

### Twitter (Monolith -> SOA, 2011-2015)
- **Legacy:** Ruby on Rails monolith ("Fail Whale" era)
- **Migration:** Strangler Fig - extracted Tweet storage (Finagle services), search, timeline
- **Facade:** Internal routing layer directing traffic to new vs old services
- **Good at:** Timeline service was extracted first (highest scaling pressure)

### Airbnb (Rails -> SOA, 2018-present)
- **Legacy:** Ruby on Rails monolith
- **Migration:** Gradual extraction by domain - Payments extracted first, then Reviews, then Search
- **Good at:** Each extracted service is independently deployable; Rails monolith shrinks over time

### Netflix (Java monolith -> microservices, 2008-2012)
- **Legacy:** DVD-era Java monolith + Oracle DB
- **Migration:** Strangler Fig starting with streaming, then recommendation, then billing
- **Duration:** ~4 years
- **Good at:** Never interrupted streaming service; migrated module by module

---

## Key Takeaway

> Strangler Fig is the responsible path from legacy to Clean Architecture. It treats migration as a product: deliver value at each step, keep the system running, contain risk. The Clean Architecture you build in each extracted service - with its interfaces (ports) - is what makes each extraction safe: the facade implements the same interface as the new service, making the swap transparent.
