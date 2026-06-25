# DoorDash - Architecture Case Study

> "DoorDash handles 80 million requests per second at peak. Every request is a potential food delivery. Latency isn't just a performance metric - it's the difference between getting dinner and missing it." - DoorDash Engineering

---

## Company Profile

| | |
|---|---|
| **Founded** | 2013 |
| **Scale** | 37M+ monthly active users, 700K+ merchants, 7M+ Dashers (drivers) |
| **Peak** | 80M+ requests/second on busy Friday nights |
| **Architecture today** | Kubernetes + Envoy Service Mesh + Kafka + CockroachDB |

---

## The Three-Sided Marketplace Problem

> "DoorDash has three actors: consumers (ordering food), merchants (restaurants preparing food), and Dashers (drivers delivering food). Coordinating all three simultaneously, in real-time, across 4,000+ cities is an order-of-magnitude harder than a two-sided marketplace."

```
Order lifecycle (all must happen in coordination):
  1. Consumer places order -> merchant notified (immediately)
  2. Merchant accepts -> DoorDash assigns nearest Dasher
  3. Merchant starts preparing -> estimated prep time communicated
  4. Dasher en route to merchant -> consumer sees driver on map
  5. Merchant completes order -> Dasher picks up
  6. Dasher drives to consumer -> real-time tracking
  7. Delivery -> payment processed -> merchant paid -> Dasher paid

9 state transitions. Each can fail. Each involves 3 parties.
= 3-way Saga pattern across 3 different user types.
```

---

## The Monolith to Microservices Transition

**The monolith problems:**

```
2018: DoorDash is a Python monolith called "Pyxis"
  Problems:
    - Single deployment: a bug in payment code -> all of DoorDash down
    - Scaling: can't scale Dasher matching independently of consumer checkout
    - Team velocity: 200 engineers in one codebase = constant conflicts

  The breaking point: Dasher supply vs consumer demand
    Peak demand: Friday 6pm -> 100x more consumer orders than Monday 2am
    Dasher supply: independent of consumer demand (Dashers choose their hours)

    In the monolith: scaling checkout automatically scales ALL of Pyxis
 -> Paying for 100x server capacity even for Dasher management (doesn't scale)
 -> Need to scale checkout independently from Dasher supply management
```

**The migration:**

```
Priority 1: Extract Dasher dispatch (clearest boundary, highest value)
 -> DispatchService (Go) - matches Dashers to orders

Priority 2: Extract consumer-facing checkout
 -> OrderService (Kotlin) - order lifecycle management

Priority 3: Extract merchant portal
 -> MerchantService (Python initially, then Kotlin)

Priority 4: Extract payments
 -> PaymentService (Java) - Stripe integration, accounting

Each extraction: Strangler Fig pattern
  New service handles new orders
  Old Pyxis handles existing orders during transition
  Migrate traffic gradually (canary: 1% -> 5% -> 20% -> 100%)
```

---

## The Service Mesh: 80M Requests/Second

**The problem at scale:**

```
Each food order involves 15-20 microservice calls:
  OrderService -> MerchantService -> InventoryService
  OrderService -> DispatchService -> DasherLocationService
  OrderService -> PaymentService -> StripeAPI
  OrderService -> NotificationService (push to consumer + Dasher + merchant)

  15 service calls x 80M total requests/second = 1.2 BILLION service calls/second

Problems without service mesh:
  Each service implements its own retry logic (inconsistently)
  One slow service causes threads to pile up, cascading to caller
  Cross-AZ (availability zone) data transfer: expensive ($$$)
  No visibility into which service is slow
```

**The Envoy service mesh solution:**

```
DoorDash uses Envoy proxy as sidecar + Istio control plane

1. Zone-aware routing (cost optimization):
   Service A (pod in us-east-1a) -> Service B (prefer us-east-1a pods)
   Why: Cross-AZ data transfer in AWS costs $0.01/GB
   80M req/sec x average 10KB payload = 800GB/sec
   Cross-AZ 20% of that = 160GB/sec x $0.01 = $1.44M/hour in transfer fees!
   Zone-aware routing: route to same AZ when possible
   Result: "Substantially reduced" (their words) data transfer costs
   AWS even called to ask if DoorDash was having a production incident (the
   cost drop was so dramatic it looked like an outage)

2. Adaptive concurrency limiting:
   Each service has auto-configured concurrency limits
   Based on current response latency (higher latency -> lower concurrency limit)
   Prevents one slow service from consuming all threads on its caller

3. Circuit breaking:
   Merchant API sometimes slow (during rush)
   Circuit breaker: if >50% of requests fail -> stop sending for 30 seconds
   Callers get immediate fallback (cached menu) instead of waiting
```

---

## The Dispatch Algorithm: Matching Orders to Dashers

**The real-time optimization problem:**

```
At any moment:
  4,000+ cities
  Each city: N pending orders, M available Dashers

  Find the optimal assignment:
    Minimize: total delivery time across all orders
    Subject to: Dasher capacity, restaurant prep time, distance

  This is the Vehicle Routing Problem (VRP) - NP-hard in general

  DoorDash's approach:
    Can't solve globally optimal (NP-hard, takes too long)
    Solve approximately: greedy algorithm in <100ms

  Algorithm:
    For each pending order: rank all available Dashers by score
    Score = f(distance, Dasher rating, estimated prep time, batching opportunity)
    Assign order to highest-scoring Dasher
    Run every 30 seconds

  "Batching": one Dasher picks up multiple orders from same restaurant
    Consumer A: Pizza Hut, 2 miles away
    Consumer B: Pizza Hut, 2.5 miles away
 -> One Dasher picks up both -> more efficient -> faster delivery for both
```

---

## CockroachDB for Configuration Management

**The problem:**

```
DoorDash has 700K+ merchants. Each has configuration:
  Delivery radius (miles)
  Operating hours per day
  Menu item availability (some items only at lunch)
  Special promotions (active/inactive)
  Dasher pickup instructions ("Ring bell at back door")

  Problems with old approach:
    Configuration in code repository -> code PR required to change hours
    Operations team: "Restaurant X changed hours for Christmas" -> PR -> review -> deploy -> 1 day
    Restaurant needed: 5 minutes to take effect

  New approach: self-service configuration platform
    Merchants update their own settings in DoorDash Merchant Portal
    Takes effect immediately (< 1 minute)
```

**CockroachDB choice:**

```
Requirements:
  Strong consistency: menu item marked "unavailable" must not appear
  High availability: DoorDash can't have configuration service go down
  Multi-region: configuration needed globally (DoorDash in 4,000+ cities)
  SQL: configuration is relational (menus, hours, items, modifiers)

Options evaluated:
  PostgreSQL: strong consistency, no automatic multi-region
  DynamoDB: multi-region, but eventual consistency only
  Cosmos DB: multi-region, strong consistency available, but Azure-only
  CockroachDB: distributed SQL, multi-region, strong consistency, PostgreSQL wire protocol

Winner: CockroachDB
  Runs CockroachDB across 3 AWS regions (us-east-1, us-west-2, eu-west-1)
  Strong consistency across all regions
  Automatic failover if one region dies
  Merchants can update config -> consistent read anywhere in <200ms
```

---

## Architecture in Clean Architecture Terms

```
DoorDash's Architecture:

OrderUseCase = Use Case layer
  PlaceOrderUseCase depends on:
    IOrderRepository (port) -> CockroachOrderRepo (adapter)
    IDispatchPort (port) -> DispatchServiceGrpcClient (adapter)
    IPaymentPort (port) -> PaymentServiceGrpcClient (adapter)
    INotificationPort (port) -> NotificationServiceClient (adapter)

  Business rule: "Order can only be placed if merchant is open"
 -> Check MerchantConfiguration via IConfigRepository
 -> CockroachConfigRepository (adapter) fetches from CockroachDB

Envoy Sidecar = Framework & Drivers layer
  Zone-aware routing: transparent to OrderUseCase
  Circuit breaking: transparent to all use cases
  Services never import Envoy SDK - it's infrastructure

DispatchAlgorithm = Domain Service (pure logic)
  No database imports
  No Kafka imports
  Input: List<PendingOrder>, List<AvailableDasher>
  Output: List<OrderDasherAssignment>
  Pure function - testable without infrastructure

Kafka = Event bus (IEventBus -> KafkaEventBus adapter)
  OrderPlaced event published by PlaceOrderUseCase
  DispatchService subscribes and runs matching algorithm
```

---

## Lessons for Your Architecture

1. **Three-sided marketplaces need Saga patterns** - coordinating consumer + merchant + driver requires explicit distributed transaction management
2. **Zone-aware routing is a financial decision** - AWS cross-AZ transfer fees at 80M req/sec are measured in millions per hour
3. **Service mesh pays for itself** - adaptive concurrency limiting prevented cascading failures that would have cost more than the mesh's overhead
4. **Self-service configuration reduces ops burden** - merchants updating their own hours vs operations team doing it = 10x faster + ops team freed for real work
5. **NP-hard problems need approximate solutions** - perfect matching is impossible in <100ms; good enough in <10ms beats optimal in 10 seconds

---

## Sources
- [DoorDash Uses Service Mesh and Cell-Based Architecture - InfoQ](https://www.infoq.com/news/2024/01/doordash-service-mesh/)
- [How DoorDash Moved to a Service Mesh to Handle 80M Requests/Second - ByteByteGo](https://blog.bytebytego.com/p/how-doordash-moved-to-a-service-mesh)
- [How DoorDash transitioned from Monolith to Microservices](https://careersatdoordash.com/blog/how-doordash-transitioned-from-a-monolith-to-microservices/)
- [DoorDash Uses CockroachDB for Config Management - InfoQ](https://www.infoq.com/news/2024/02/doordash-config-cockroachdb/)


---

## Architecture Diagram

![diagram.svg](diagram.svg)

