# Architecture 07 — API Gateway

---

## At a Glance

| | |
|---|---|
| **Type** | Single entry-point for all client requests |
| **Complexity** | Low-Medium |
| **Best for** | Microservices with multiple clients, cross-cutting concerns (auth, rate limiting) |
| **Avoid when** | Monolith (overkill), single client, very low traffic |

---

## What Is It?

An API Gateway is a **single entry point** that sits between clients and backend services. It handles cross-cutting concerns so individual services don't have to implement them independently.

Think of it as the **front door** to your microservices ecosystem.

---

## Diagram Reference
`./diagram.svg`

---

## Responsibilities

```
Client Request → API Gateway does:
  1. Authentication & Authorization (verify JWT, OAuth token)
  2. Rate Limiting (max 100 req/sec per client)
  3. SSL Termination (HTTPS → HTTP internally)
  4. Request Routing (GET /orders → Order Service)
  5. Load Balancing (round-robin across Order Service instances)
  6. Request/Response Transformation (v1 API → v2 internal format)
  7. Caching (cache GET /products for 60 seconds)
  8. Logging & Tracing (add correlation ID to every request)
  9. Circuit Breaking (stop routing to unhealthy services)
```

---

## In Clean Architecture Terms

The API Gateway lives entirely in the **Frameworks & Drivers** layer. It is infrastructure, not business logic.

```
[Client] → [API Gateway: Nginx/Kong/AWS ALB] → [Order Service Controller] → [PlaceOrderUseCase]

The API Gateway:
  - Knows about HTTP routing rules
  - Knows about JWT validation
  - Does NOT know about business logic
  - Does NOT know about database schemas
```

Individual service controllers are the **Interface Adapter** (inbound) layer — they receive already-authenticated, already-rate-limited requests from the gateway.

```typescript
// OrderController (Interface Adapter layer)
// By the time this receives a request, the gateway has already:
// - Verified the JWT
// - Rate-limited the client
// - Added the userId to the request headers

class OrderController {
  async placeOrder(req: Request, res: Response) {
    const userId = req.headers['x-user-id']; // injected by gateway after auth
    const result = await this.placeOrderUseCase.execute({
      userId,
      items: req.body.items,
    });
    res.json(result);
  }
}
```

---

## Types of API Gateway

### 1. L7 Reverse Proxy (Simple)
Nginx, Traefik — basic routing and SSL termination.

### 2. API Management Gateway (Full-featured)
Kong, AWS API Gateway, Azure API Management — plugins for auth, rate limiting, caching, analytics.

### 3. GraphQL Gateway
Apollo Federation, Hasura — aggregates multiple services behind one GraphQL schema.

### 4. gRPC Gateway
Envoy, grpc-gateway — transcodes HTTP/JSON to gRPC internally.

---

## Resource Consumption

| Resource | Usage | Notes |
|---|---|---|
| **CPU** | Low-Medium | JWT validation, request transformation |
| **Memory** | Low | Connection pool + routing table |
| **Network** | +1 hop | Every request adds one extra network hop |
| **Latency** | +1-5ms | Auth check + routing overhead |
| **Availability** | Critical | Gateway is SPOF; must be clustered with HA |
| **Ops cost** | Low | Managed options (AWS ALB, Kong Cloud) reduce ops burden |

---

## Benefits

1. **Single entry point** — clients only know one URL; services can move/split freely
2. **DRY cross-cutting concerns** — auth implemented once in gateway, not in 10 services
3. **API versioning** — route `/v1/orders` to old service, `/v2/orders` to new service
4. **Protocol translation** — clients use REST; internally services use gRPC
5. **Rate limiting** — protect services from overload without service-level implementation
6. **Analytics** — all traffic visible in one place; easy dashboards

---

## Problems It Solves Best

| Problem | Why API Gateway Wins |
|---|---|
| "10 microservices each implement JWT auth differently" | Gateway handles auth once, injects userId |
| "Mobile clients can't handle our internal microservice URLs" | One stable URL; internal topology hidden |
| "Product and user data live in different services, mobile needs both" | BFF gateway aggregates them |
| "We're overloaded by one bad API client" | Rate limit that client at the gateway |
| "We need to migrate Order Service v1 to v2 without breaking clients" | Route /v1 to old, /v2 to new simultaneously |

---

## Costs / Tradeoffs

1. **Single point of failure** — gateway must be clustered and HA
2. **Additional latency** — +1 network hop; optimize with local caching
3. **Config complexity** — routing rules, auth plugins, rate limit configs add up
4. **Not a business logic host** — avoid putting business logic in the gateway (anti-pattern)

---

## Big Tech Examples

### Netflix — Zuul
- **Architecture:** Zuul API Gateway fronts all Netflix services
- **Features:** Auth via identity service, dynamic routing, A/B testing at edge, canary deployments
- **Good at:** Traffic shaping — route 5% of traffic to new service version, 95% to old
- **Later:** Moved to Zuul 2 (async, non-blocking) and now Envoy/Service Mesh

### Amazon — AWS API Gateway + ALB
- **Architecture:** AWS API Gateway for REST/HTTP/WebSocket APIs
- **Features:** IAM auth, Lambda integration, usage plans, custom domains
- **Good at:** Serverless APIs — API Gateway → Lambda; no servers to manage
- **Volume:** Handles hundreds of billions of API calls/month for AWS customers

### Uber — Custom API Gateway
- **Architecture:** Custom Go-based gateway called "GAPIC"
- **Features:** Service discovery via Hyperbahn (Uber's internal mesh), auth, rate limiting
- **Good at:** Routes millions of trip requests to the right dispatch service region

### Airbnb — Gatekeeper
- **Architecture:** Internal API Gateway called Gatekeeper
- **Features:** Auth, feature flags, A/B test routing, mobile API aggregation
- **Good at:** Serving mobile apps (iOS/Android) with aggregated responses (BFF style)

### Kong / KrakenD (Open Source)
- **Used by:** Mashape, Salesforce, Nokia, T-Mobile
- **Architecture:** Plugin-based API gateway (Lua plugins on Nginx)
- **Good at:** Custom plugins for any cross-cutting concern imaginable

---

## Key Takeaway

> The API Gateway is infrastructure — it lives in the outermost (Frameworks & Drivers) layer of Clean Architecture. It never contains business logic. Its job is to receive client requests, authenticate, rate-limit, route, and hand off to the correct service's controller. This separation ensures that adding a new gateway plugin never requires changing business logic.
