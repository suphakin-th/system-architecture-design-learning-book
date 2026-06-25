# TODO - Curriculum Roadmap

Topics a senior developer should know that are not yet taught as their own page.
From the gap analysis on 2026-06-26 (Tier 1 is done; these are what remain).

## Tier 2 - deepens existing strengths

- [ ] **Message queues / streaming deep-dive** - Kafka internals, delivery
  guarantees (at-least-once / at-most-once / exactly-once), partitions and
  consumer groups, backpressure. Referenced across many case studies but never
  taught on its own page.
- [ ] **Distributed-systems theory** - consensus (Raft / Paxos), leader election,
  logical / vector clocks, CRDTs, distributed locks. The depth behind the
  existing Saga and Replication pages.
- [ ] **Domain-Driven Design (DDD)** - bounded contexts, aggregates, ubiquitous
  language, anti-corruption layer. The missing companion to Clean Architecture.
- [ ] **Containers & Kubernetes** - Docker, pods / services / ingress, the
  control loop. The existing Service Mesh page already assumes this.
- [ ] **Scalability & capacity estimation** - vertical vs horizontal, load
  balancing algorithms, stateless design, back-of-envelope math. Interview staple.

## Tier 3 - engineering craft (slightly different altitude)

- [ ] **Testing strategy** - test pyramid, unit / integration / e2e, TDD,
  contract testing, property-based testing.
- [ ] **CI/CD & deployment strategies** - pipelines, blue-green, canary, feature
  flags, rollback, GitOps, infrastructure as code.
- [ ] **Concurrency & parallelism** - threads, async / event loops, locks, race
  conditions, deadlocks, the actor model.

## Done (Tier 1, 2026-06-26)

- [x] Caching & CDN
- [x] Observability (logs / metrics / traces)
- [x] Application Security (authn / authz / OWASP)
- [x] API Design & Protocols
- [x] Resilience Patterns
