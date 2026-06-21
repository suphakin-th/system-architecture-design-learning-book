# System Architecture Design — Learning Book

> **"The goal of software architecture is to minimize the human resources required to build and maintain the required system."** — Robert C. Martin

---

## What Is This?

A complete, self-contained learning library for software and system architecture — built from scratch using real research, big-tech engineering blogs, and production battle-tested patterns.

**Written as a senior architect explaining to a junior/mid developer.** Not just "what" a pattern is, but **why it exists**, **what problem it solves**, **what it costs**, and **how real companies use it**.

---

## Who Is This For?

- Junior → Mid developers who want to understand how large systems are built
- Mid → Senior developers preparing for system design interviews
- Anyone studying for **ByteByteGo**, **Designing Data-Intensive Applications**, or **Clean Architecture**
- Thai developers looking for resources in both Thai and English

---

## How to Use This Book

=== "Learning Path (Recommended)"

    ```
    Week 1–2:   Clean Architecture Foundation → Monolith → Layered N-Tier
    Week 3–4:   CQRS → Event-Driven → Event Sourcing
    Week 5–6:   Microservices → Saga → API Gateway → BFF
    Week 7–8:   Database Fundamentals → Indexing → Replication → Transactions
    Week 9–10:  Fintech Architecture (Ledger → Audit → Payments → Security)
    Week 11+:   Company Case Studies (Amazon → Netflix → Google → ...)
    ```

=== "Quick Reference"

    Use the **search bar** (top right) to find any concept instantly.

    Examples to try:
    - `fanout` → Twitter's hybrid fanout solution
    - `idempotency` → Stripe's payment safety
    - `MVCC` → PostgreSQL concurrency control
    - `PromptPay` → Thailand's real-time payment system
    - `สิทธิ์` → Linux permissions in Thai

=== "By Problem"

    | I need to... | Go to |
    |---|---|
    | Understand Clean Architecture from zero | [Foundation](architecture/00_foundation.md) |
    | Know when to use microservices | [Microservices](architecture/02_microservices.md) |
    | Design a payment/financial system | [Fintech Overview](fintech/00_overview.md) |
    | Make my database faster | [Query Optimization](database/28_query_optimization.md) |
    | Pass a security pentest | [Linux Permissions](security/linux_permissions_thai.md) |
    | Learn from Netflix's architecture | [Netflix Case Study](companies/netflix.md) |

---

## Coverage

<div class="grid cards" markdown>

-   :material-layers: **15 Architecture Patterns**

    ---
    From Monolith to Service Mesh, each with SVG diagram, real-world examples, and Clean Architecture mapping.

-   :material-database: **11 Database Topics**

    ---
    Types, indexing (B-Tree vs LSM), replication, ACID/BASE/CAP, MVCC, sharding, ANSI/SPARC 3-layer model.

-   :material-currency-usd: **8 Fintech Patterns**

    ---
    Double-entry ledger, immutable audit logs, Visa/PromptPay networks, HSM/PCI DSS, Zero Trust, idempotency.

-   :material-office-building: **17 Big Tech Companies**

    ---
    Amazon, Google, Meta, Netflix, Uber, Twitter, LinkedIn, Shopify, Stripe, Discord, WhatsApp, TikTok, Airbnb, Grab, DoorDash, Microsoft, Pantip.

-   :material-shield-lock: **Linux Security**

    ---
    Permissions, SUID/SGID, ACL, systemd hardening — written in Thai 🇹🇭 for microservice server setup.

-   :material-school: **Zero to Hero**

    ---
    Every topic explained as a senior architect teaching a junior. No assumed knowledge.

</div>

---

## The One Diagram That Rules Them All

```
┌─────────────────────────────────────────────┐
│         Frameworks & Drivers                │  ← Express, PostgreSQL, Kafka
│  ┌───────────────────────────────────────┐  │
│  │       Interface Adapters              │  │  ← Controllers, Repositories
│  │  ┌─────────────────────────────────┐  │  │
│  │  │         Use Cases               │  │  │  ← Business operations
│  │  │  ┌───────────────────────────┐  │  │  │
│  │  │  │        Entities           │  │  │  │  ← Business rules (pure)
│  │  │  └───────────────────────────┘  │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘

      Dependencies point INWARD only →
      Inner layers know NOTHING about outer layers.
```

**This is Clean Architecture.** Every pattern in this book is a variation on this diagram at a different scale — from a single module to 1,000 microservices.

---

## Sources & Credits

This library synthesizes knowledge from:

- [ByteByteGo](https://bytebytego.com) — Alex Xu's system design resources
- [Netflix Tech Blog](https://netflixtechblog.com) · [Uber Engineering](https://www.uber.com/blog/engineering/) · [Shopify Engineering](https://shopify.engineering)
- [LinkedIn Engineering](https://engineering.linkedin.com) · [Discord Engineering](https://discord.com/blog/engineering) · [Stripe Dev Blog](https://stripe.dev/blog)
- [Designing Data-Intensive Applications](https://dataintensive.net) — Martin Kleppmann
- [Clean Architecture](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164) — Robert C. Martin
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/)
