# Architecture 02 — Microservices

---

## At a Glance

| | |
|---|---|
| **Type** | Distributed independent services |
| **Complexity** | High |
| **Best for** | Large teams, clear domain boundaries, independent scaling needs |
| **Avoid when** | Team < 10, domain still unclear, no Kubernetes expertise |

---

## What Is It?

Microservices decompose a system into small, independently deployable services. Each service:
- **Owns its own database** — no shared DB, no DB joins across services
- **Runs in its own process** — crashes independently, scales independently
- **Communicates via network** — REST, gRPC, or async events
- **Is developed by one team** — Conway's Law applied by design

**Each microservice IS its own complete Clean Architecture application.**

---

## Diagram Reference
`./diagram.svg`

---

## Structure

```
Client
  │
  ▼
API Gateway  ← auth, rate-limit, route, load-balance
  │
  ├──► User Service        ──► PostgreSQL (users DB)
  │    ├── domain/         User, Profile entities
  │    ├── application/    RegisterUser, Login use cases
  │    ├── adapters/       REST controller, UserRepoPostgres
  │    └── infrastructure/ Express, pg driver
  │
  ├──► Order Service       ──► PostgreSQL (orders DB)
  ├──► Product Service     ──► MongoDB (products DB)
  ├──► Payment Service     ──► PostgreSQL (payments DB)
  └──► Notification Svc   ──► Redis (queued jobs)

Services communicate via:
  ─── REST/gRPC    (synchronous, immediate answer needed)
  ··· Events/MQ    (async, fire-and-forget or eventual consistency)
```

---

## In Clean Architecture Terms

The **network boundary replaces the module boundary**. The interface (port) between services becomes a network contract (API spec / event schema).

```typescript
// In Monolith: IUserService is in-process
class PlaceOrderUseCase {
  constructor(private userService: IUserService) {}
}

// In Microservices: IUserService is implemented by an HTTP client
class HttpUserServiceClient implements IUserService {
  async getUserById(id: string): Promise<User> {
    const res = await axios.get(`http://user-service/users/${id}`);
    return UserMapper.toDomain(res.data);
  }
}

// ← The PlaceOrderUseCase code didn't change at all!
```

**This is why Clean Architecture + interfaces make the monolith → microservices migration mechanical.**

---

## Communication Patterns

### Synchronous (REST / gRPC)
```
Order Service ──HTTP GET──► User Service
```
- Use when: immediate answer required to continue the request
- Risk: if User Service is down, Order Service call fails

### Asynchronous (Events / Message Queue)
```
Order Service ──publishes──► [OrderPlaced event]
Payment Service ◄─subscribes─┘
Email Service ◄─subscribes─┘
```
- Use when: eventual consistency is acceptable; decoupling desired
- Tools: Kafka, RabbitMQ, AWS SQS

---

## Resource Consumption

| Resource | Usage | Notes |
|---|---|---|
| **CPU** | Medium-High | Each service has its own runtime overhead |
| **Memory** | High | N services × minimum heap = significant total RAM |
| **Network** | High | Every cross-service call = network round-trip (1–10ms) |
| **Disk** | Medium | N Docker images, N build artifacts |
| **Ops cost** | Very High | Kubernetes, service mesh, distributed tracing, central logs, secrets management |
| **Dev machine** | Complex | docker-compose with 5+ services to run locally |

---

## Benefits

1. **Independent scaling** — scale Payment Service ×10, keep User Service ×1
2. **Fault isolation** — Product Service crash doesn't bring down orders
3. **Team autonomy** — 3 teams deploy on their own schedules, own CI/CD pipelines
4. **Technology diversity** — ML service in Python, API in Go, frontend BFF in Node.js
5. **Smaller codebases** — each team owns an understandable repo
6. **Targeted deployments** — fix a bug in Payment Service without touching anything else

---

## Problems It Solves Best

| Problem | Why Microservices Win |
|---|---|
| "Search needs 10× more CPU than checkout" | Scale each service independently |
| "3 teams deploy to the same monolith, blocking each other" | Each team has their own service |
| "The ML recommendation engine needs Python, the API needs Go" | Different language per service |
| "One service crash takes down the whole site" | Fault isolation via circuit breakers |
| "We need to comply with GDPR — user data must be isolated" | User service owns all PII |

---

## Costs / Tradeoffs

1. **Network latency** — in-process call = nanoseconds; HTTP call = 1–10ms
2. **Distributed transactions** — no free ACID; must use Saga or eventual consistency
3. **Operational complexity** — Kubernetes, Istio, Jaeger, ELK, Vault, etc.
4. **Testing is hard** — integration tests need service orchestration
5. **Data duplication** — denormalized data stored in multiple services
6. **Debugging is hard** — trace spans across 5 services to find the bug

---

## Big Tech Examples

### Amazon
- **Architecture:** 1000+ microservices; each team owns their services end-to-end
- **Rule:** "Two-pizza team" — if a team can't be fed by two pizzas, it's too big
- **Communication:** Mix of REST (synchronous) and SNS/SQS (async events)
- **Good at:** Independent scaling of checkout, search, recommendations, payments
- **Story:** Amazon's original 2002 Bezos mandate: all teams must expose data via service interfaces; no direct DB access

### Netflix
- **Architecture:** 700+ microservices
- **Services:** Content delivery (Open Connect CDN), streaming, recommendations, billing, search
- **Tools they built:** Eureka (service discovery), Hystrix (circuit breaker), Zuul (API Gateway), Ribbon (load balancer)
- **Good at:** Chaos Engineering — deliberately kill services to test fault tolerance
- **Key insight:** Each service has its own Cassandra cluster; no shared state

### Uber
- **Architecture:** Started as monolith (dispatch system in Python), migrated to microservices 2014-2016
- **Services:** Pricing, dispatch, maps, payments, driver profiles, trip history, notifications
- **Communication:** Protocol Buffers + gRPC for synchronous; Kafka for async
- **Good at:** Surge pricing algorithm runs in isolation; can be updated without touching dispatch
- **Lesson learned:** Too many microservices → moved to "domain-oriented microservices" (DOMA)

### Airbnb
- **Architecture:** Started as Rails monolith, migrated to SOA (service-oriented architecture)
- **Services:** Payments (Chronos), Search, Bookings, Messaging, Reviews
- **Lesson:** Service explosion → introduced "Minerva" (data consistency framework)
- **Good at:** Search service uses Elasticsearch and scales independently from booking flow

---

## Key Takeaway

> Microservices take Clean Architecture's "dependency rule" and make it a physical boundary: the network. This gives massive organizational benefits (team autonomy, independent scaling) but requires significant operational investment. Don't start here — earn it.
