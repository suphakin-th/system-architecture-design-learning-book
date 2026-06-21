# Grab — Architecture Case Study

> "Building a super app for Southeast Asia is harder than building one for Silicon Valley. Unreliable networks, 600+ million users across 10 countries, 8 languages, and 40+ currencies." — Grab Engineering

---

## Company Profile

| | |
|---|---|
| **Founded** | 2012 (Malaysia), HQ in Singapore |
| **Scale** | 210M+ users, 8 countries (SE Asia) |
| **Services** | GrabRide, GrabFood, GrabMart, GrabPay, GrabFinance |
| **Engineering** | 3,000+ engineers, offices in Singapore, Indonesia, Vietnam, India |
| **Architecture today** | 1000+ microservices on AWS + GCP + Kubernetes + Kafka |

---

## Why Grab's Problem Is Unique

> "The US has 50 states with one currency, one language, one regulatory framework. Southeast Asia has 10 countries with 40+ currencies, 8 major languages, 10 different regulatory frameworks, and infrastructure ranging from Singapore's world-class 5G to rural Indonesia's 2G. Our architecture must handle all of this."

---

## The Super App Architecture Challenge

**What "super app" means technically:**

```
GrabRide: booking, dispatch, GPS tracking, payment
GrabFood: restaurant catalog, ordering, delivery tracking
GrabMart: grocery inventory, express delivery
GrabPay: digital wallet, QR payments, bank transfers
GrabFinance: micro-loans, insurance products

Each service:
  - Different compliance requirements (lending vs ride-hailing vs food delivery)
  - Different real-time requirements (GPS tracking = real-time; insurance = batch)
  - Different data models (route vs order vs loan vs wallet)

BUT: must share:
  - User identity (one Grab account)
  - Payment (one Grab wallet)
  - Rewards (one GrabRewards program)
  - Driver/Delivery partner profile
```

---

## The Real-Time Location Problem

**Grab's hardest technical problem:**

```
GrabRide at peak (Southeast Asia rush hour):
  5M+ active rides simultaneously across 8 countries
  Each ride: driver GPS updates every 2 seconds
  5M drivers × 0.5 updates/sec = 2.5M location updates/second

Architecture for driver location:
  Driver app → TCP connection → Location Gateway (Go)
    → Kafka (locations.updates topic)
      → Flink: compute driver position in H3 hex cells
      → Redis: store driver_id → current H3 cell (TTL: 10 seconds)
      → Cassandra: store location history (for route replay)

Rider requests a GrabCar:
  → RideService: find drivers in same H3 cell as rider
  → Redis lookup: O(1), returns list of driver_ids in that cell
  → Returns 5 nearest drivers in <10ms
```

**The network quality problem (Southeast Asia specific):**

```
Singapore: 5G, WiFi → driver location updates every 2 seconds: fine
Rural Indonesia: 2G, 200ms latency → every-2-second update drops frequently

Solution: adaptive location updates
  Signal quality detected by driver app
  Good signal: update every 2 seconds
  Poor signal: update every 10 seconds (best-effort)
  Connection drops: cache locally, bulk upload when reconnected

  Rider sees: smooth driver movement (interpolated between known positions)
  Reality: driver position estimated between sparse updates
```

---

## GrabPay: Payments in 10 Countries

**The regulatory nightmare:**

```
Country-by-country compliance:
  Singapore: MAS (Monetary Authority of Singapore) — strict fintech licensing
  Indonesia: OJK + BI regulations — separate license from Singapore
  Malaysia: BNM regulations — different from Indonesia
  Vietnam: SBV regulations — foreign payment restrictions
  Philippines: BSP regulations — separate wallet regulations
  Thailand: BOT regulations — KYC requirements differ

Each country requires:
  - Separate legal entity
  - Separate banking partner
  - Different KYC (Know Your Customer) requirements
  - Different transaction limits
  - Different data residency (data must stay in country)
```

**Architecture solution:**

```
Payment Request → PaymentRouter
  PaymentRouter: which country is this transaction in?
  → Routes to country-specific PaymentProcessor

PaymentProcessorSG (Singapore):
  → DBS Bank API → MAS-compliant transaction
  → Data stored in Singapore AWS region

PaymentProcessorID (Indonesia):
  → BCA Bank API → OJK-compliant transaction
  → Data stored in Jakarta AWS region

Shared:
  Common interface: IPaymentProcessor
  PaymentRouter selects implementation based on user's country

This is Strategy Pattern at the architecture level:
  Same use case (ProcessPayment)
  Different adapter (per country's banking/regulatory requirements)
```

---

## The App Size Problem (2024)

**The challenge:**

> "Our app was growing 1% per month in size. In Singapore, users have high-end phones. In rural Indonesia, users have $100 Android phones with 32GB storage and 2G connections. A 150MB app is a significant portion of their phone storage. Downloads over 2G take 40+ minutes."

**The solution:**

```
Problem discovered: 1,500+ modules and 3rd party libraries
Each library includes assets, resources, code that most users never use

Solution approach:
  1. Modular app architecture: load features on demand
     - Core (always loaded): login, home screen, basic navigation
     - GrabFood: downloaded when user first taps GrabFood
     - GrabFinance: downloaded when user opts into financial services

  2. R class optimization:
     Large R classes (Android resource IDs) from 1500+ modules
     Solution: R8 shrinking + dependency cleanup → 25% app size reduction

  3. WebP images (replace PNG/JPG):
     WebP: 30% smaller than PNG, 25% smaller than JPEG
     → Significant reduction in image assets

Result: App size reduced by 25% → meaningful improvement for 2G users
```

---

## Migrating 1000+ Services from Consul to Istio

**The service mesh migration:**

```
Original: Consul (service discovery + health checking)
  Each service registers with Consul
  Other services query Consul to find instances

Problem at 1000+ services:
  Consul query latency adds up across complex call chains
  No mTLS between services (security gap)
  No built-in circuit breaking or retries
  Observability: manual instrumentation in each service

Migration to Istio + Envoy:
  Sidecar proxy (Envoy) injected into each pod automatically
  Service-to-service: always mTLS (zero code changes)
  Circuit breaking: configured in YAML (zero code changes)
  Distributed tracing: automatic (zero code changes)

Challenge: hybrid infrastructure
  Grab uses AWS + GCP + on-premises data centers in some countries
  Istio control plane must span multiple cloud providers
  Solution: federated Istio installation per region + cross-region routing
```

---

## Architecture in Clean Architecture Terms

```
Grab's Architecture:

IPaymentProcessor = Port (interface) defined in PaymentUseCase
  PaymentProcessorSG, PaymentProcessorID, PaymentProcessorTH = Adapters
  PaymentRouter = Factory that selects correct adapter based on country
  Use case never knows which country's banking partner it's using

Location Service = Use Case
  Depends on IDriverLocationStore (port) → Redis adapter
  Depends on IH3IndexService (port) → custom H3 geo-index adapter
  Business rule: "find all drivers within 2km" implemented in use case
  H3 indexing details: hidden in adapter

Kafka = Event bus (IEventBus implemented by KafkaProducer/Consumer adapters)
  RideService publishes RideRequested event
  DriverMatchingService subscribes and processes it
  Services never import Kafka directly

Istio/Envoy = Framework & Drivers layer
  Service-to-service mTLS, retries, circuit breaking
  Services unaware of Istio (Clean Architecture: framework independence)
```

---

## Lessons for Your Architecture

1. **Regulatory compliance shapes architecture** — Grab's per-country payment processors aren't over-engineering; they're legal requirements
2. **Network quality is a first-class concern in developing markets** — adaptive location updates, offline-first, progressive loading
3. **App size is a feature** — for 2G markets, a 25% size reduction increases your addressable user base
4. **Service mesh solves security uniformly** — mTLS across 1000+ services without changing service code
5. **Strategy Pattern scales to country-level** — same interface, different implementation per country/regulation/currency

---

## Sources
- [Engineering — Grab Tech Blog](https://engineering.grab.com/)
- [Driving Southeast Asia Forward with AWS — Grab Engineering](https://engineering.grab.com/driving-southeast-asia-forward-with-aws)
- [Asian superapp Grab shrank its app by a quarter to survive](https://www.theregister.com/2024/03/05/grab_downsized_app/)
- [DoorDash Uses Service Mesh and Cell-Based Architecture — InfoQ](https://www.infoq.com/news/2024/01/doordash-service-mesh/)


---

## Architecture Diagram

![diagram.svg](diagram.svg)

