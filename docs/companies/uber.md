# Uber — Architecture Case Study

> "We went from a monolith to microservices, then realized we had too many microservices, then invented DOMA. Architecture is iterative." — Uber Engineering

---

## Company Profile

| | |
|---|---|
| **Founded** | 2009 |
| **Scale** | 131M monthly active users, 6M+ drivers, 70+ countries |
| **Real-time** | 15 million driver location updates per second |
| **Architecture today** | DOMA (Domain-Oriented Microservices) + Kafka + Cadence |

---

## The Senior Architect Tells the Story

> "Uber's architecture journey is the most honest one in the industry. They publicly admitted: 'we over-corrected from monolith to microservices, created chaos, then invented DOMA to organize the chaos.' That honesty and the solutions they built are worth studying deeply."

---

## Phase 1: The Monolith (2009–2014)

**The original monolith: written in Python**

```
UberApp (iOS/Android)
      │
      ▼
Uber Python Monolith
  ├── Driver matching logic
  ├── Pricing (surge pricing algorithm)
  ├── Trip management
  ├── Payment processing
  ├── Push notifications
  └── Maps & routing
      │
      ▼
PostgreSQL (one database)
```

**Why Python?** Fast to prototype, startup speed, small team.

**Problems that forced them to change:**

1. **The dispatch bottleneck:**
   The core of Uber is matching drivers to riders. In Python, the matching algorithm ran 4,000 matches per second. At peak in San Francisco, they needed 20,000 per second. Python's GIL (Global Interpreter Lock) prevented true parallelism.

2. **Deployment risk:**
   Changing the pricing algorithm required deploying the entire Python monolith. If the deploy had a bug, ALL of Uber went down — not just pricing.

3. **Database lock contention:**
   Driver location updates hit the PostgreSQL driver table 15M times/second. Row-level locks + MVCC overhead brought Postgres to its knees.

---

## Phase 2: Microservices (2014–2018)

**Extracted core services:**

```
Mobile API
      │
      ▼
API Gateway (custom Go-based)
      │
      ├──► Dispatch Service (Go)      — match drivers to riders
      ├──► Pricing Service (Go)       — surge pricing calculation
      ├──► Trip Service (Python/Go)   — trip lifecycle management
      ├──► Payment Service (Java)     — Stripe/Braintree integration
      ├──► Notification Service (Go)  — push notifications
      ├──► Maps Service (Go)          — routing, ETAs
      └──► Driver Location Service (Go) — real-time GPS updates
```

**The driver location problem — solved with geospatial indexing:**

```
Problem: 6M drivers, each updating location every 4 seconds
  = 1.5M location updates per second just for location storage
  = Can't use regular SQL index (too many writes)

Solution: H3 geospatial indexing (hexagonal hierarchical)
  Earth divided into hexagonal cells at different resolutions
  Level 9 hex cell ≈ 0.1 km² (neighborhood-sized)

  Driver update: store driver_id → H3 cell mapping
  Rider request: find all drivers in my H3 cell + adjacent cells

  Instead of: "Find all drivers within 2km radius" (slow polygon query)
  Use: "Find all drivers in hex cells 12345, 12346, 12347..." (fast lookup)
```

**The surge pricing architecture:**

```
Traditional approach (would not work at scale):
  Every ride request → query all drivers in area → calculate supply/demand → price

Uber's approach:
  Kafka stream of ride requests and driver locations
  Flink processes the stream continuously
  Pre-compute surge multiplier per H3 hex cell every 30 seconds
  Store surge map in Redis (expires after 60 seconds)

  Ride request: lookup hex cell → read surge multiplier from Redis
  = one Redis read = sub-millisecond pricing
  = surge pricing is a read, not a calculation
```

---

## Phase 3: The Microservices Problem (2018)

**What went wrong with 1,000+ microservices:**

> "Every team had their own microservice. Building a 'assign driver to trip' feature required changing 8 services owned by 6 different teams. We needed 4 cross-team design reviews, 8 PRs, 8 deployments. What used to be a 1-day change became a 3-week coordination effort."

**The specific problems:**

1. **No clear ownership:** A bug in the dispatch flow touched code owned by 5 teams. Who debugs it?
2. **Circular dependencies:** Service A calls B calls C calls A → deadlock, impossible to reason about
3. **"Networked monolith":** Services that appeared independent had to deploy together because of tight coupling via API contracts

---

## Phase 4: DOMA — Domain-Oriented Microservice Architecture (2020)

**The insight:**

> "Microservices became our Big Ball of Mud — just distributed. DOMA says: group services by business domain, not by technical function. Define strict ownership. Create explicit layers. Enforce boundaries."

**DOMA Structure:**

```
Layer 1: Infrastructure (used by everyone)
  ├── Logging Service
  ├── Monitoring Service
  ├── Auth Service
  └── Config Service

Layer 2: Business Services (Uber-wide, used across products)
  ├── Payments Domain
  │   ├── PaymentMethodService
  │   ├── ChargeService
  │   └── RefundService
  ├── Identity Domain
  │   ├── UserService
  │   └── DriverService
  └── Maps Domain
      ├── RoutingService
      ├── GeofenceService
      └── LocationService

Layer 3: Product Services (specific to one Uber product)
  ├── UberRides Domain
  │   ├── DispatchService
  │   ├── TripService
  │   └── PricingService
  └── UberEats Domain
      ├── OrderService
      ├── RestaurantService
      └── DeliveryService

Rule: Layer N can call Layer N-1 and lower, but NEVER higher
Rule: Same layer services should NOT call each other (go through domain gateway)
```

**DOMA Gateway Pattern:**

Each domain exposes a single Gateway API that all external callers use:

```
External caller → RidesGateway → internal dispatch to DispatchService/TripService
                                   (callers don't know which internal service handles it)

This is the Facade design pattern at the domain level.
```

---

## Cadence — Workflow Orchestration for the Saga Pattern

**Problem:** A trip involves: matching driver → pricing → payment auth → trip start → trip end → payment charge → driver payout. Any step can fail. The saga pattern requires compensating transactions.

**Cadence (now Temporal.io):**

```go
// Cadence workflow = Saga Orchestrator
func TripWorkflow(ctx workflow.Context, tripId string) error {
  // Step 1: Match driver
  var driver DriverInfo
  err := workflow.ExecuteActivity(ctx, matchDriver, tripId).Get(ctx, &driver)
  if err != nil {
    return err // no driver found, trip cancelled
  }

  // Step 2: Authorize payment
  err = workflow.ExecuteActivity(ctx, authorizePayment, tripId, driver.EstimatedFare)
  if err != nil {
    // Compensate: release driver
    workflow.ExecuteActivity(ctx, releaseDriver, driver.Id)
    return err
  }

  // Step 3: Run trip (waits for trip completion — could be hours)
  err = workflow.ExecuteActivity(ctx, runTrip, tripId).Get(ctx, nil)

  // Step 4: Capture payment
  workflow.ExecuteActivity(ctx, capturePayment, tripId)

  return nil
}
// If the server crashes mid-trip, Cadence replays the workflow from the last checkpoint
// The driver and rider never know a server restarted
```

**Why Cadence/Temporal was invented at Uber:**
> "Long-running workflows (a trip is 20 minutes; waiting for human approval is days) can't run in-process — the server restarts. Cadence persists workflow state as events, replays them on restart. It's event sourcing for workflow orchestration."

---

## Real-Time Data Architecture

```
Driver Location Update (every 4 seconds per driver)
  │
  ▼
Kafka (kafka.location.updates topic)
  │
  ├──► Flink (stream processing) → surge pricing calculation → Redis
  ├──► Cassandra (location history for trip replay/audit)
  └──► H3 GeoIndex Service → dispatch lookup

Rider requests a trip:
  → DispatchService reads H3 index from Redis (nearest drivers)
  → Sends trip offer to N nearest drivers via WebSocket
  → First driver to accept → TripService creates trip
  → Saga workflow starts
```

---

## Architecture in Clean Architecture Terms

```
DOMA = Clean Architecture at organizational scale

Domain Gateway = Interface Adapter (inbound for the domain)
  Callers interact with the domain through the gateway
  Internal domain structure is hidden

Internal Domain Services = Use Cases
  DispatchService.matchDriver() = a use case
  Depends on IDriverRepository, IGeofenceService (interfaces)

Infrastructure Services = Framework & Drivers layer
  Kafka publishing, Redis caching, Cassandra repos = adapters
  Business logic never imports Kafka directly

DOMA Layer Rule = Clean Architecture's Dependency Rule
  Layer N can only depend on Layer N-1 and lower
  = same as "inner layers never import outer layers"
```

---

## Lessons for Your Architecture

1. **Microservices is a people problem** — 1000 services × 1000 engineers works; 1000 services × 100 engineers = chaos
2. **Domain boundaries must match team boundaries** — Conway's Law is real; your architecture mirrors your org chart
3. **Event streaming solves real-time scale** — 15M location updates/second requires Kafka, not REST calls
4. **Geospatial indexing is a specialized problem** — H3 hexagons beat lat/lon polygon queries at Uber's scale
5. **Long-running workflows need Saga/Temporal** — a trip is 20 minutes; you can't hold a DB transaction that long

---

## Sources
- [Introducing Domain-Oriented Microservice Architecture — Uber Blog](https://www.uber.com/blog/microservice-architecture/)
- [Service-Oriented Architecture: Scaling the Uber Engineering Codebase](https://www.uber.com/blog/service-oriented-architecture/)
- [Uber System Design Deep Dive — Grokking](https://grokkingthesystemdesign.com/guides/uber-system-design/)


---

## Architecture Diagram

![diagram.svg](diagram.svg)

