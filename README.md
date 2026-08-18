# System Architecture Design Library
### Clean Architecture — Zero to Hero

> A complete learning library: 15 architecture patterns, each with an SVG diagram, detailed guide, resource analysis, real-world big-tech examples, and grounding in Clean Architecture principles.

---

## How to Use This Library

1. **Start here:** `00_clean_architecture_foundation/` — the lens for understanding EVERYTHING else
2. **Follow the learning path below** — each level builds on the previous
3. **Each directory:** `README.md` (full explanation) + `diagram.svg` (open in browser)
4. **Open SVG files** in any browser (Chrome, Firefox, Edge) — drag and drop the `.svg` file

---

## Learning Path

### Level 0 — The Foundation (Day 1-7)
**Goal:** Understand what Clean Architecture is and why it exists

| Step | Directory | What You'll Learn |
|---|---|---|
| 1 | `00_clean_architecture_foundation/` | The dependency rule, 4 layers, SOLID, ports & adapters |
| 2 | `10_layered_ntier/` | Where most devs start — and why it breaks down |

**Outcome:** You can explain the dependency rule and draw the concentric circle diagram.

---

### Level 1 — Monolith First (Day 8-14)
**Goal:** Build a well-structured application that applies Clean Architecture

| Step | Directory | What You'll Learn |
|---|---|---|
| 3 | `01_monolith/` | Modular monolith structure, when to stay, when to leave |
| 4 | `08_hexagonal/` | Same as Clean Arch but with ports/adapters vocabulary |
| 5 | `09_onion/` | DDD-aligned naming for domain-heavy apps |

**Outcome:** You can build a monolith with clean layers, interfaces everywhere, and unit tests that don't need a DB.

---

### Level 2 — Splitting the Domain (Day 15-30)
**Goal:** Understand how to separate read from write and handle complex commands

| Step | Directory | What You'll Learn |
|---|---|---|
| 6 | `04_cqrs/` | Commands (write) vs Queries (read) — two separate models |
| 7 | `05_event_sourcing/` | State = event history — audit trail + time travel |
| 8 | `03_event_driven/` | Events as the primary communication mechanism |

**Outcome:** You can design a system where reads and writes are independently optimized.

---

### Level 3 — Going Distributed (Day 31-60)
**Goal:** Understand how to break a monolith into services responsibly

| Step | Directory | What You'll Learn |
|---|---|---|
| 9 | `02_microservices/` | Each service = its own Clean Architecture app |
| 10 | `06_saga/` | Distributed transactions — when ACID crosses service boundaries |
| 11 | `07_api_gateway/` | Single entry point — auth, routing, rate limiting |
| 12 | `11_bff/` | Per-client backends for mobile/web/TV optimization |

**Outcome:** You can design a microservices system with proper boundary management and distributed transaction handling.

---

### Level 4 — Infrastructure Patterns (Day 61-90)
**Goal:** Understand the ops-level patterns that support distributed systems

| Step | Directory | What You'll Learn |
|---|---|---|
| 13 | `13_serverless/` | Functions as code — zero ops, auto-scale, pay per use |
| 14 | `14_service_mesh/` | Infrastructure layer for resilience and observability |
| 15 | `12_strangler_fig/` | How to migrate legacy to modern architecture without a rewrite |

**Outcome:** You can design and migrate real-world systems with production-grade operational concerns.

---

### Supplementary — AI Infra Hardware

Not part of the software architecture learning path above, but a related decision
that comes up constantly when running AI workloads on your own machine rather than
someone else's cloud.

| Directory | What You'll Learn |
|---|---|
| `34_ai_infra_hardware/` | NPU vs GPU vs CPU for local AI inference — why NPU generations aren't interchangeable, why "has an NPU" doesn't mean "can run your model," and a decision framework for picking the right compute target |

---

## Pattern Reference Table

| # | Pattern | Problem It Solves | Resource Cost | Complexity |
|---|---|---|---|---|
| 00 | **Clean Architecture Foundation** | Testability, maintainability, tech independence | None (organizational) | Medium |
| 01 | **Monolith** | Fast delivery, team simplicity, ACID transactions | Low | Low |
| 02 | **Microservices** | Independent scaling, team autonomy, fault isolation | High | High |
| 03 | **Event-Driven** | Decoupling, resilience, real-time reactions | Medium | Medium-High |
| 04 | **CQRS** | Read/write imbalance, complex queries, audit | Medium | Medium |
| 05 | **Event Sourcing** | Audit trail, time travel, history | High | High |
| 06 | **Saga** | Distributed transactions, long-running workflows | Medium | High |
| 07 | **API Gateway** | Single entry point, cross-cutting concerns | Low | Low-Medium |
| 08 | **Hexagonal** | Framework independence, testability | None | Medium |
| 09 | **Onion** | Complex domain (DDD), enterprise systems | None | Medium |
| 10 | **Layered (N-Tier)** | Simple CRUD, beginner teams, MVC frameworks | Very Low | Low |
| 11 | **BFF** | Multiple client types with different data needs | Medium | Medium |
| 12 | **Strangler Fig** | Legacy migration without big-bang rewrite | 2x during migration | Medium |
| 13 | **Serverless** | Variable traffic, event-driven, zero ops | Pay per use | Low-Medium |
| 14 | **Service Mesh** | Uniform resilience + observability at scale | High (sidecar overhead) | High |

---

## Quick Decision Guide: Which Pattern for My Problem?

```
START HERE: How big is your team?

1-5 devs → Modular Monolith [01] + Clean Architecture [00]
6-20 devs → Modular Monolith or consider extracting 1-2 services
20+ devs → Microservices [02] (if domain is clear)

───────────────────────────────────────────────────────────────

SCALING: What needs to scale independently?
Search/recommendations → Extract as a service [02]
Nothing yet → Stay in monolith [01]

───────────────────────────────────────────────────────────────

READ/WRITE RATIO: More reads than writes?
10:1 or higher → CQRS [04] (separate read model)
Complex audit trail → CQRS [04] + Event Sourcing [05]

───────────────────────────────────────────────────────────────

COMMUNICATION: How should services talk?
Immediate answer needed → REST/gRPC (synchronous)
Eventually consistent OK → Event-Driven [03]
Transaction spans services → Saga [06]

───────────────────────────────────────────────────────────────

CLIENTS: How many client types?
One → Single API is fine
Multiple (mobile/web/TV) → BFF [11]

───────────────────────────────────────────────────────────────

TRAFFIC: What pattern?
Spiky/event-driven workloads → Serverless [13]
Steady, latency-critical → Containers/services

───────────────────────────────────────────────────────────────

LEGACY: Have a monolith to migrate?
Yes → Strangler Fig [12] (never big-bang rewrite)

───────────────────────────────────────────────────────────────

INFRASTRUCTURE: Many services, need uniform ops?
10+ services → API Gateway [07]
50+ services → Service Mesh [14]
```

---

## Big Tech Architecture Map

| Company | Pattern Used | Why |
|---|---|---|
| **Amazon** | Microservices + Event-Driven + Saga | 1000+ services; two-pizza teams; SNS/SQS for async |
| **Netflix** | Microservices + Service Mesh (Envoy) + CQRS | 700+ services; Chaos Engineering; Cassandra per service |
| **Uber** | Event-Driven (Kafka) + Saga + BFF | Real-time trip events; driver location at scale |
| **Airbnb** | BFF + Strangler Fig + Event-Driven | Gatekeeper BFF; Rails → SOA migration |
| **LinkedIn** | Event-Driven (invented Kafka) + CQRS | 7T messages/day; feed = pre-computed CQRS read model |
| **Stripe** | Event Sourcing + CQRS | Financial ledger must be auditable; payment = event stream |
| **Shopify** | Modular Monolith + Strangler Fig | 2M LOC Rails monolith; extract only when needed |
| **Stack Overflow** | Layered Monolith | Proves monolith can scale with proper optimization |
| **Twitter/X** | Event-Driven + CQRS | Tweet fan-out = massive write; feed = read-optimized view |
| **Google** | Service Mesh (built Istio/Envoy) + gRPC | Internal: billions of RPCs/day; mesh = standard practice |
| **Spotify** | Hexagonal + Microservices | Python/Java services; swap recommendation engine freely |

---

## The Most Important Diagram

All 14 patterns are implementations of one underlying principle:

```
CLEAN ARCHITECTURE:
  Inner layers (business rules) must never depend on outer layers (frameworks, DBs).
  All dependencies point inward.
  Boundaries are crossed using interfaces (ports) and implementations (adapters).

MONOLITH:       boundaries are module boundaries (in-process calls)
MICROSERVICES:  boundaries are network boundaries (HTTP/gRPC)
EVENT-DRIVEN:   boundaries are event schema contracts
HEXAGONAL:      boundaries are ports (interfaces) and adapters (implementations)
CQRS:           boundary between write model and read model
EVENT SOURCING: boundary between events (source of truth) and projections (views)
SAGA:           boundary between distributed transaction steps
API GATEWAY:    boundary between external clients and internal services
BFF:            boundary between client-specific API and backend services
SERVICE MESH:   boundary between application logic and network infrastructure
SERVERLESS:     boundary between function (business logic) and runtime (cloud)
STRANGLER FIG:  boundary between old system and new system (progressive replacement)
```

**Master the boundary. Master software architecture.**

---

## File Structure

```
clean_architechture_design/
├── README.md                              ← You are here (master index)
├── 00_clean_architecture_foundation/
│   ├── README.md                          ← Zero-to-hero guide
│   └── diagram.svg                        ← Concentric circle diagram
├── 01_monolith/
│   ├── README.md
│   └── diagram.svg
├── 02_microservices/
│   ├── README.md
│   └── diagram.svg
├── 03_event_driven/
│   ├── README.md
│   └── diagram.svg
├── 04_cqrs/
│   ├── README.md
│   └── diagram.svg
├── 05_event_sourcing/
│   ├── README.md
│   └── diagram.svg
├── 06_saga/
│   ├── README.md
│   └── diagram.svg
├── 07_api_gateway/
│   ├── README.md
│   └── diagram.svg
├── 08_hexagonal/
│   ├── README.md
│   └── diagram.svg
├── 09_onion/
│   ├── README.md
│   └── diagram.svg
├── 10_layered_ntier/
│   ├── README.md
│   └── diagram.svg
├── 11_bff/
│   ├── README.md
│   └── diagram.svg
├── 12_strangler_fig/
│   ├── README.md
│   └── diagram.svg
├── 13_serverless/
│   ├── README.md
│   └── diagram.svg
├── 14_service_mesh/
│   ├── README.md
│   └── diagram.svg
└── 34_ai_infra_hardware/       ← supplementary, not part of the architecture ladder
    ├── README.md
    └── diagram.svg
```
