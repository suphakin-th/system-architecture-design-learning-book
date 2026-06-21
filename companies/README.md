# Big Tech Companies — Architecture Case Studies

> How the world's most successful companies solved real engineering problems at scale — and what you can learn from each.

---

## Study Guide

Each company directory contains:
- `README.md` — Full case study (problem → why → solution → lessons)
- `diagram.svg` — Visual architecture diagram (open in browser)

**Reading order for learning:**

| Order | Company | Key Lesson |
|---|---|---|
| 1 | `shopify/` | Modular monolith is a valid choice; NOT all companies need microservices |
| 2 | `amazon/` | The Bezos Mandate: API-first thinking creates the foundation for everything |
| 3 | `netflix/` | Design for failure; circuit breakers; chaos engineering |
| 4 | `twitter_x/` | The fanout problem: pre-computed reads beat real-time joins at scale |
| 5 | `linkedin/` | Kafka was invented to solve a real problem; Kappa beats Lambda |
| 6 | `meta_facebook/` | TAO: when your data is a graph, use a graph store |
| 7 | `airbnb/` | SOA migration requires domain clarity BEFORE service extraction |
| 8 | `uber/` | DOMA: organize microservices by domain, not by technical function |
| 9 | `discord/` | Technology choice matters: Erlang for persistent connections |
| 10 | `whatsapp/` | 50 engineers + right tech beats 5000 engineers + wrong tech |
| 11 | `stripe/` | Idempotency + event sourcing = the foundation of financial systems |
| 12 | `tiktok_bytedance/` | Online ML training: algorithm improves in minutes, not hours |
| 13 | `google/` | Papers > Products: publishing your solutions creates industry |
| 14 | `grab/` | Southeast Asian constraints: multi-country regulation, 2G networks |
| 15 | `doordash/` | Three-sided marketplace; zone-aware routing saves millions in AWS costs |
| 16 | `microsoft/` | Platform leverage: Azure differentiates by integrating with existing investments |
| 17 | `pantip/` | Local context matters: board-based sharding, Thai NLP, spike prediction |

---

## The Universal Pattern

Every company here follows this arc:

```
1. Started with a monolith (LAMP, Rails, Python, Java)
2. Hit a specific bottleneck (scaling, teams, deployment speed)
3. Made an architectural decision based on THAT specific bottleneck
4. Invented or adopted technology to solve THAT problem
5. Published their solution → other companies benefited

The wrong lesson: "Netflix uses microservices, so I should too"
The right lesson: "Netflix had 700 engineers and needed fault isolation → extracted services"
                  "Shopify has 1000 engineers and needs merchant isolation → pods + modular monolith"
```

---

## Quick Reference: What Problem → Which Company Solved It

| Problem | Company | Solution | Pattern |
|---|---|---|---|
| "Monolith deploys break everything" | Amazon | Two-pizza teams, API-first mandate | Microservices |
| "One service crash brings down the site" | Netflix | Circuit breakers + Chaos Engineering | Resilience patterns |
| "Celebrity tweet fans out to 100M followers slowly" | Twitter | Hybrid fanout (push regular, pull celebrity) | CQRS + caching |
| "100 data pipelines, each custom-coded" | LinkedIn | Kafka (hub-and-spoke replaces M×N pipelines) | Event-Driven |
| "Social graph queries take minutes" | Meta | TAO (graph store + 3-tier cache) | Specialized storage |
| "SOA attempt failed — circular dependencies" | Airbnb | Domain boundaries first, GraphQL BFF | Strangler Fig + BFF |
| "2,200 microservices, no clear ownership" | Uber | DOMA (domain-oriented layers) | Domain-Driven Design |
| "5M+ concurrent WebSocket connections" | Discord | Erlang VM (2KB per process, 2M per server) | Actor model |
| "2B users, 50 engineers" | WhatsApp | Erlang + Mnesia + XMPP (battle-tested stack) | Efficiency over novelty |
| "Every payment must be exactly once" | Stripe | Idempotency keys + event sourcing | Financial patterns |
| "Recommendation takes hours to learn new viral content" | TikTok | Online ML (Monolith) — updates in minutes | Real-time ML |
| "Process entire internet for search" | Google | MapReduce + GFS + Bigtable | Distributed computing |
| "10 countries × 10 regulations" | Grab | Strategy Pattern: per-country payment adapters | Adapter pattern |
| "AWS cross-AZ fees at 80M req/sec = millions/hr" | DoorDash | Zone-aware routing via Envoy service mesh | Service Mesh |
| "2M merchants, big merchant slows small merchant" | Shopify | MySQL pods (shop_id shard key) | Database sharding |
| "Thai word search doesn't work with default tokenizer" | Pantip | Thai NLP tokenizer in Elasticsearch | Domain-specific config |

---

## How Every Architecture Maps to Clean Architecture

```
Amazon's Bezos Mandate       = Clean Architecture's Dependency Rule (services = interfaces)
Netflix's Hystrix            = Infrastructure layer (framework, not business logic)
Twitter's FanoutService      = Use Case with side effects + Strategy Pattern
LinkedIn's Kafka             = IEventBus port implemented by KafkaProducer adapter
Meta's TAO                   = IGraphRepository port implemented by TAO adapter
Airbnb's GraphQL             = Interface Adapter (inbound) — BFF as controller layer
Uber's DOMA layers           = Clean Architecture's concentric rings at org scale
Discord's Erlang GenServer   = Entity + Use Case combined in one actor
WhatsApp's Ejabberd          = Framework & Drivers (Erlang runtime; use cases inside)
Stripe's PaymentIntent       = Entity reconstituted from event sourcing
TikTok's Monolith embedding  = Infrastructure adapter for ML model lookups
Google's Borg/Kubernetes     = Framework & Drivers (container runtime; services unaware)
Grab's per-country adapters  = Clean Architecture's Strategy + Adapter patterns
DoorDash's Envoy sidecar     = Framework & Drivers (mesh; services 100% unaware)
Shopify's Packwerk           = Tooling that enforces the Dependency Rule
Pantip's board shard router  = Infrastructure adapter (selects MySQL cluster by board_id)
```

---

## Sources Used in Research

- [A Brief History of Scaling Netflix — ByteByteGo](https://blog.bytebytego.com/p/a-brief-history-of-scaling-netflix)
- [Introducing DOMA — Uber Blog](https://www.uber.com/blog/microservice-architecture/)
- [TAO: The Power of the Graph — Engineering at Meta](https://engineering.fb.com/2013/06/25/core-infra/tao-the-power-of-the-graph/)
- [GraphQL: A Data Query Language — Engineering at Meta](https://engineering.fb.com/2015/09/14/core-infra/graphql-a-data-query-language/)
- [Under Deconstruction: Shopify's Monolith — Shopify Engineering](https://shopify.engineering/shopify-monolith)
- [Running Kafka at Scale — LinkedIn Engineering](https://engineering.linkedin.com/kafka/running-kafka-scale)
- [How Discord Scaled Elixir to 5M Concurrent Users — Discord](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
- [Scaling Trillions of Messages: Discord's Journey from Cassandra to ScyllaDB](https://seifrajhi.github.io/blog/discord-cassandra-to-scylladb/)
- [WhatsApp's Secret Weapon: Erlang — Medium](https://ritik-chopra28.medium.com/whatsapps-secret-weapon-erlang-why-50-engineers-handle-2-billion-users-b19129a01ec9)
- [Stripe's Payments APIs: The First 10 Years — Stripe Dev Blog](https://stripe.dev/blog/payment-api-design)
- [The Great Migration: Airbnb from Monolith to SOA — InfoQ](https://www.infoq.com/presentations/airbnb-soa-migration/)
- [Twitter's Fanout Strategy at Scale — DEV Community](https://dev.to/gabrielanhaia/twitters-fanout-strategy-at-scale-the-trade-off-most-designs-miss-55oa)
- [DoorDash Moves to Service Mesh — ByteByteGo](https://blog.bytebytego.com/p/how-doordash-moved-to-a-service-mesh)
- [Monolith: ByteDance's Recommendation System — Aaron Abraham](https://www.aaronabraham.ca/technical-writing/tiktok-monolith-system)
- [Google SRE Book — Production Environment](https://sre.google/sre-book/production-environment/)
- [Spanner: Google's Globally Distributed Database — Cornell](https://www.cs.cornell.edu/courses/cs5414/2017fa/papers/Spanner.pdf)
- [A Brief History of Airbnb's Architecture — ByteByteGo](https://blog.bytebytego.com/p/a-brief-history-of-airbnbs-architecture)
- [LinkedIn Migrates from Lambda to Kappa — InfoQ](https://www.infoq.com/news/2020/12/linkedin-lambda-architecture/)
