# Architecture 14 - Service Mesh

---

## At a Glance

| | |
|---|---|
| **Type** | Infrastructure layer for service-to-service communication |
| **Complexity** | High (operational) |
| **Best for** | Large microservices deployments needing observability, security, and resilience |
| **Avoid when** | Monolith, small number of services (<5), team without Kubernetes expertise |

---

## What Is It?

A Service Mesh is a **dedicated infrastructure layer** that handles service-to-service communication in a microservices architecture. Instead of each service implementing retry logic, mTLS, circuit breaking, and observability independently, the mesh handles all of this via **sidecar proxies** injected alongside each service.

**The service doesn't know the mesh exists. Clean Architecture achieved at the infrastructure level.**

---

## Diagram Reference
`./diagram.svg`

---

## How It Works

Every service pod gets a **sidecar proxy** (e.g., Envoy). All traffic flows through these proxies:

Traffic from the Order Service flows through a local sidecar, across the network, and into the Payment Service's sidecar before reaching the service:

```mermaid
flowchart LR
    A["Order Service"] --> B["Envoy Sidecar"]
    B --> C["Network"]
    C --> D["Envoy Sidecar"]
    D --> E["Payment Service"]
```

The Order Service just makes an HTTP call to `http://payment-service:3004/charge`. The sidecar proxy handles everything else:

- mTLS encryption (both sides authenticated)
- Retry on 503 (3 times, exponential backoff)
- Circuit breaking (stop sending if 50% errors)
- Timeout (fail fast after 2 seconds)
- Metrics (request count, latency, error rate)
- Distributed tracing (add and propagate trace headers)
- Load balancing (choose healthiest instance)

---

## Components

### Data Plane (Sidecar Proxies)
- **Envoy Proxy** - the most widely used; handles all traffic
- Injected automatically by the mesh control plane
- Intercepts ALL inbound + outbound traffic from the service

### Control Plane
- **Istio** - most feature-rich; uses Envoy as data plane
- **Linkerd** - lightweight alternative; Rust-based
- **Consul Connect** - HashiCorp's mesh; integrates with Consul service discovery

---

## In Clean Architecture Terms

The service mesh is **entirely in the Frameworks & Drivers layer** - it's infrastructure. Your use cases, entities, and interface adapters know NOTHING about it.

```typescript
// Order Service - PlaceOrderUseCase calls Payment Service via IPaymentGateway port
class PlaceOrderUseCase {
  constructor(private payments: IPaymentGateway) {}

  async execute(req: PlaceOrderRequest) {
    // The use case calls the payment gateway through the interface.
    // The HTTP call goes through the Envoy sidecar.
    // The sidecar handles mTLS, retries, circuit breaking.
    // The use case has NO IDEA any of this is happening.
    await this.payments.charge(req.amount, req.card);
  }
}

// HttpPaymentGateway (adapter) makes a simple HTTP call
class HttpPaymentGateway implements IPaymentGateway {
  async charge(amount: Money, card: PaymentCard) {
    const res = await axios.post('http://payment-service/charge', { amount, card });
    return PaymentMapper.toDomain(res.data);
  }
}
// The sidecar intercepts this call and adds: mTLS + retry + circuit breaker + metrics
// The HttpPaymentGateway doesn't need any of this code!
```

**This is the ultimate expression of Clean Architecture's "framework independence" principle.**

---

## Features the Mesh Provides for Free

### Traffic Management
- Load balancing (round-robin, least-connections, consistent hash)
- Traffic splitting (canary: 5% to new version, 95% to old)
- Circuit breaking (stop routing to unhealthy instances)
- Retry policies (retry 3 times on 503, with backoff)
- Timeout enforcement (fail after 2 seconds)

### Security
- **mTLS** - mutual TLS between all services; every service authenticates
- **Authorization policies** - "Order Service can call Payment Service; cannot call User Service"
- **Certificate rotation** - automatic, no human involvement

### Observability
- **Metrics** - request count, error rate, latency per service pair (Prometheus)
- **Distributed tracing** - full trace across service calls (Jaeger, Zipkin)
- **Access logs** - who called what, when, with what response

---

## Resource Consumption

| Resource | Usage | Notes |
|---|---|---|
| **CPU** | +5-15% per sidecar | Envoy proxy has CPU overhead per request |
| **Memory** | +50-100MB per pod (sidecar) | Each sidecar = extra container in the pod |
| **Network latency** | +1-2ms per hop | Loopback call to sidecar then out to network |
| **Ops complexity** | Very High | Istio alone is a significant operational investment |
| **Storage** | Low (metrics) | Prometheus stores metrics; Jaeger stores traces |
| **Team required** | Platform/SRE team | Dedicated team to manage the mesh |

---

## Benefits

1. **Zero-code resilience** - retries, circuit breaking, timeouts configured in YAML, not code
2. **Automatic mTLS** - all service-to-service traffic encrypted and authenticated without code
3. **Full observability** - golden signals (latency, traffic, errors, saturation) for every service pair
4. **Traffic splitting** - canary deployments and A/B tests without code changes
5. **Policy enforcement** - "Search Service must NOT call Payments Service" enforced at infrastructure level
6. **Uniform across languages** - same sidecar works for Node.js, Python, Go, Java services

---

## Problems It Solves Best

| Problem | Why Service Mesh Wins |
|---|---|
| "10 teams, 50 services - each implements retries differently" | Mesh provides uniform retry policy for all |
| "We need mTLS between all services but can't change 50 service codebases" | Mesh injects mTLS at sidecar level; zero code change |
| "How do I know which service is the bottleneck?" | Mesh traces every request; Jaeger shows you exactly |
| "We want to deploy new version to 5% of traffic" | Istio VirtualService: weight: 5 for new, weight: 95 for old |
| "GDPR: Service A must never send data to Service B" | Istio AuthorizationPolicy: DENY A -> B at network level |

---

## Costs / Tradeoffs

1. **Complexity** - Istio alone has 20+ CRDs; steep learning curve
2. **Resource overhead** - every pod needs a sidecar (+100MB RAM, +5-10% CPU)
3. **Debugging harder** - mTLS issues, sidecar config errors add a new failure layer
4. **Requires Kubernetes** - most meshes are Kubernetes-native
5. **Not needed at small scale** - 3 services don't need a mesh; overkill

---

## Big Tech Examples

### Lyft (Envoy creator, 2016)
- **Architecture:** Envoy was created at Lyft to solve their microservices communication chaos
- **Problem:** 100+ services, each team implementing its own retry/timeout/circuit breaker logic in different ways
- **Solution:** One sidecar proxy (Envoy) for all services; one team manages it
- **Good at:** Uniform resilience across all services; observability dashboard for every service pair

### Google (Istio, Traffic Director)
- **Architecture:** Istio control plane + Envoy data plane (Google developed Istio)
- **Use case:** Internal Google services use similar proxy-based mesh (Stubby -> gRPC)
- **Scale:** Handles trillions of RPCs/day internally
- **Good at:** Automatic certificate rotation; mTLS enforced at cluster level

### Uber (uForwarder + Envoy)
- **Architecture:** Envoy-based mesh managing 4000+ services
- **Good at:** Traffic shifting during deployments - 1% canary, then 5%, 20%, 100% - automated
- **Observability:** Full request traces across 4000 services via Jaeger

### Pinterest
- **Architecture:** Envoy service mesh across their Python/Java/Go microservices
- **Problem:** Needed mTLS without re-writing 50 services
- **Solution:** Istio + Envoy; zero code changes in services
- **Good at:** Authorization policies: "Feed service cannot query Payment service" enforced at network level

### Airbnb
- **Architecture:** Envoy sidecar + custom control plane ("Aperture")
- **Good at:** Circuit breaking for their third-party integrations (Twilio, Braintree, etc.); one trip takes down only that integration, not other services

---

## Senior Advice

> "Add a service mesh when you have 10+ services and a dedicated platform team. It's not something a product team should manage. Think of it as 'WiFi for your microservices' - you don't implement 802.11 in every application; you let the infrastructure layer handle it. The beauty is that your services (and Clean Architecture within them) remain completely unchanged. The mesh is invisible to your use cases, which is exactly right."

---

## Key Takeaway

> The Service Mesh is the purest expression of Clean Architecture's "separation of concerns" at the infrastructure level. Your services don't implement resilience, observability, or security - the mesh does. Your use cases remain clean. This is Clean Architecture applied not just to code organization, but to the entire runtime infrastructure.
