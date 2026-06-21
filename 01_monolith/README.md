# Architecture 01 — Monolith (Modular Monolith)

---

## At a Glance

| | |
|---|---|
| **Type** | Single deployable unit |
| **Complexity** | Low |
| **Best for** | Early-stage products, small teams, unclear domain |
| **Avoid when** | Teams >10, components need independent scaling |

---

## What Is It?

A monolith is a **single process** containing all application functionality. One build, one deploy, one running process. When structured well (modular monolith), it applies Clean Architecture WITHIN that single process — clean layers, clear module boundaries, dependency inversion.

**There is no "bad monolith" by default. A poorly structured monolith (Big Ball of Mud) is bad. A well-structured modular monolith is excellent.**

---

## Diagram Reference
`./diagram.svg`

---

## Structure (Clean Architecture Applied)

```
my-app/
├── modules/
│   ├── users/
│   │   ├── domain/          ← Entities (User, Profile, Role)
│   │   ├── application/     ← Use Cases (RegisterUser, Login)
│   │   ├── adapters/        ← Controller, UserRepositoryPostgres
│   │   └── infrastructure/  ← Express routes, pg client wiring
│   ├── orders/
│   │   ├── domain/          ← Entities (Order, OrderItem)
│   │   ├── application/     ← Use Cases (PlaceOrder, CancelOrder)
│   │   ├── adapters/        ← Controller, OrderRepositoryPostgres
│   │   └── infrastructure/
│   └── products/
│       ├── domain/
│       ├── application/
│       ├── adapters/
│       └── infrastructure/
├── shared/                  ← Shared value objects (Money, Email)
└── main.ts                  ← Composition root — wires everything
```

---

## How Clean Architecture Fits

Each module has its own full Clean Architecture stack. The module boundary replaces the network boundary. Cross-module calls are just function calls in the same process:

```typescript
// Order Use Case calls User Use Case via interface (not direct import of User's DB)
class PlaceOrderUseCase {
  constructor(
    private userService: IUserService,   // interface — could be UserService in same process OR a remote HTTP call
    private orderRepo: IOrderRepository
  ) {}
}
```

This means migrating to microservices later is just changing the adapter:
- `IUserService` implemented by `InProcessUserService` → monolith
- `IUserService` implemented by `HttpUserServiceClient` → microservices

**The use case doesn't change at all.**

---

## Resource Consumption

| Resource | Usage | Notes |
|---|---|---|
| **CPU** | Low overhead | No serialization between modules; in-process calls |
| **Memory** | Single process heap | Shared memory space — no duplicate data copies |
| **Network I/O** | Minimal | Only external APIs and DB; no inter-service HTTP |
| **Disk** | Single artifact | One Docker image, one build artifact |
| **Ops cost** | Very low | One service to monitor, one to deploy, one log stream |
| **Dev machine** | Simple | `npm start` — one command runs everything |

---

## Benefits

1. **Simple deployment** — `docker build` + `docker run`, one container
2. **Low latency** — cross-module calls are in-process function calls (~nanoseconds)
3. **Free ACID transactions** — single DB = transactions span all modules trivially
4. **Simple debugging** — one log stream, one stack trace, one profiler
5. **Easy refactoring** — IDE finds all usages of a renamed function across all modules
6. **Fast test feedback** — no service orchestration; integration tests run against one process
7. **Low operational cost** — no Kubernetes, no service mesh, no distributed tracing needed

---

## Problems It Solves Best

| Problem | Why Monolith Wins |
|---|---|
| "We need to ship an MVP fast" | Lowest friction to get running |
| "Our domain is still unclear" | Refactoring a monolith is 10x easier than splitting services |
| "We have 3 developers" | No orchestration overhead |
| "We need transactions across orders + inventory" | One DB = free atomic transactions |
| "We want to hire juniors" | One codebase, linear reasoning, simpler to learn |

---

## Costs / Tradeoffs

1. **All-or-nothing scaling** — can't scale just the search module; must scale the whole app
2. **Deployment coupling** — one module's bug requires full redeployment
3. **Technology lock-in** — all modules must use the same language + framework version
4. **Risk of tangling** — without discipline, modules import each other directly → Big Ball of Mud
5. **Memory pressure** — all modules share one heap; a memory leak anywhere affects everything

---

## Real-World Examples

| Company | Tech | Notes |
|---|---|---|
| **Shopify** | Ruby on Rails | ~2M lines, serves $200B+ GMV/year |
| **GitHub** | Ruby on Rails | Monolith for most of its life |
| **Stack Overflow** | .NET | Serves millions of requests/day on very few servers |
| **Basecamp** | Rails | Deliberately chose to stay monolith (see "The Majestic Monolith") |
| **Etsy** | PHP | Stayed monolith, optimized it, outperformed microservice competitors |

---

## Migration Path to Microservices (When Ready)

Use the **Strangler Fig Pattern** (see Pattern 12):
1. Identify the module with the most scaling pressure (e.g., Search)
2. Extract it as a standalone service
3. Add API Gateway to route its traffic
4. The interface you already defined (`ISearchService`) becomes the network contract
5. Repeat for next module — no big bang rewrite ever needed

**The Clean Architecture inside the monolith made this migration easy: interfaces were already defined.**

---

## Key Takeaway

> A well-structured modular monolith is the correct default starting point. Apply Clean Architecture rigorously. When you have a concrete, measurable reason to extract a service (scaling, team independence, tech diversity), the Clean Architecture you built makes that extraction straightforward.
