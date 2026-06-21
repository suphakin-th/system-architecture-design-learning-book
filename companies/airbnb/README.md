# Airbnb — Architecture Case Study

> "We tried microservices, failed, tried again, and eventually succeeded — but not in the way we expected." — Airbnb Engineering

---

## Company Profile

| | |
|---|---|
| **Founded** | 2008 |
| **Scale** | 150M+ users, 4M+ listings, 220+ countries |
| **Bookings** | 500M+ nights booked total |
| **Architecture today** | SOA + GraphQL (Relay) + Airflow + Hive + Flink |

---

## Phase 1: The Ruby on Rails Monolith (2008–2014)

**The original setup:** Ruby on Rails monolith + PostgreSQL + Nginx on AWS

**The problems that forced change:**

```
By 2014:
  - 5 separate teams, all committing to one Rails app
  - "You broke my build" was said multiple times daily
  - Deploy frequency: once per week (too risky to deploy more)
  - One bad deploy → entire Airbnb down
  - Search, Payments, Messaging, and Listings teams all stepping on each other
```

---

## Phase 2: The Failed Microservices Attempt (2015–2016)

**The honest failure:**

> "In 2015 we announced we would migrate to microservices in 18 months. We missed every deadline. After a year of effort, we called it off. The migration was taking longer than expected, and we were still shipping features in the monolith. We hadn't solved the fundamental problem: circular dependencies."

**What went wrong:**

```
Circular dependency problem:
  PaymentService needed UserService for user verification
  UserService needed PaymentService for billing validation

  Both needed to import each other → circular dependency
  → Can't split into separate services

Root cause: the domain boundaries weren't clear.
If you can't draw a box around one part of your system
that has zero circular dependencies, you can't extract it.
```

**The lesson:**

> "You can't migrate to microservices before you understand your domain boundaries. Domain boundaries first. Service extraction second."

---

## Phase 3: SOA with GraphQL + Relay (2017–present)

**The approach that worked: clear contracts first:**

```
Instead of jumping to separate deployments, Airbnb:
1. Defined the API contracts between domains
2. Enforced those contracts inside the monolith
3. Extracted services only when boundaries were clearly validated

Service extraction order (by value, not by difficulty):
  1. Payments Service (highest risk, clearest boundary, financial compliance)
  2. Messaging Service (clear boundary: just messages between users)
  3. Search Service (compute-intensive, needed Elasticsearch independently)
  4. Reviews Service (clearest domain, lowest coupling)
```

**GraphQL as the BFF layer:**

```
Problem: Airbnb has iOS, Android, and web clients.
  Listing page on web: needs 50 fields (full description, all photos, all reviews)
  Listing on mobile: needs 15 fields (small preview, thumbnail, price)

  REST API: GET /listings/123 → returns all 50 fields always
  Mobile over-fetches: downloads 50 fields, uses 15 → wastes bandwidth/battery

GraphQL solution:
  Mobile client asks for exactly what it needs
  Web client asks for what it needs
  One GraphQL endpoint serves both optimally
```

---

## The Pricing Architecture (How Airbnb Sets Prices)

**The data flow for pricing:**

```
Host sets base price: $100/night

Airbnb's Price Tips (ML suggestion):
  Input: listing characteristics + market supply/demand + seasonality
  Output: "You could earn 20% more by pricing at $120 on weekends"

  ML Pipeline:
    Hive (batch): historical booking data → feature engineering → daily
    Flink (streaming): real-time demand signals → pricing adjustments
    ML model: gradient boosted trees → price recommendation

Dynamic Pricing (Smart Pricing):
  Airbnb adjusts price automatically if host opts in
  Updated: daily (based on Hive batch) + real-time (based on Flink stream)

  Architecture:
    PricingService reads from:
      - Hive (batch features: market rate, seasonality)
      - Redis (real-time demand: how many searches in this area today)
    Returns: adjusted price for this listing on this date
```

---

## The Search Architecture

**Search at Airbnb scale:**

```
User searches: "2 nights in Bangkok, 2 adults, December"
      │
      ▼
Search Query Service
  Parses query → structured search request
      │
      ▼
Availability Service
  Which listings are available Dec 15-17?
  Source: PostgreSQL availability calendar (not search engine)
      │
      ▼
Elasticsearch (listing search)
  Filters: Bangkok area, 2+ guests, available dates
  Returns: candidate listing IDs (10,000 candidates)
      │
      ▼
Ranking Service (ML)
  For each candidate: predict likelihood of booking given this user's history
  Features: price vs user's historical spend, superhost status, photo quality
  Returns: top 100 ranked listings
      │
      ▼
Enrichment Service
  For each of top 100: fetch full listing details from listing service
      │
      ▼
User sees: top 20 results with price, photo, rating
```

---

## Architecture in Clean Architecture Terms

```
Airbnb's Architecture:

Search Use Case = Use Case layer
  SearchListingsUseCase depends on:
    IAvailabilityService (port) → AvailabilityServiceClient (adapter)
    ISearchIndex (port) → ElasticsearchAdapter (adapter)
    IRankingService (port) → MLRankingServiceClient (adapter)

  Use case orchestrates these ports; adapters implement them
  Use case code is framework-agnostic

GraphQL = Interface Adapter (inbound)
  GraphQL resolver = Controller that calls SearchListingsUseCase
  Client specifies field selection in the query
  Resolver returns only the requested fields

Flink Jobs = Specialized Use Cases for pricing data pipeline
  Reads from Kafka (demand events) → computes pricing signals → writes to Redis
  Business logic: "if search volume increased 50% → adjust price signal"
  Infrastructure: Flink handles stream processing; use case is just the logic

Hive = Infrastructure layer (batch data store)
  IPricingFeatureStore → HiveFeatureStore (adapter)
  PricingService reads historical features via IPricingFeatureStore
  Hive details hidden behind interface
```

---

## Lessons for Your Architecture

1. **Domain boundaries must be clear before service extraction** — Airbnb's failed first attempt: boundaries weren't clear
2. **GraphQL solves multi-client data needs** — one endpoint, each client asks for what it needs
3. **Validate boundaries inside the monolith first** — enforce API contracts inside one codebase before splitting deployments
4. **Extract by risk and value, not by difficulty** — Payments first (compliance need), not the easiest service first
5. **Real-time + batch = complete pricing** — Hive for historical patterns, Flink for real-time demand

---

## Sources
- [A Brief History of Airbnb's Architecture — ByteByteGo](https://blog.bytebytego.com/p/a-brief-history-of-airbnbs-architecture)
- [The Great Migration: from Monolith to Service-Oriented — InfoQ](https://www.infoq.com/presentations/airbnb-soa-migration/)
- [Building Services at Airbnb — Airbnb Tech Blog](https://medium.com/airbnb-engineering/building-services-at-airbnb-part-1-c4c1d8fa811b)
- [Airbnb's SOA Migration — TechAhead](https://www.techaheadcorp.com/blog/how-airbnb-migrated-to-microservices/)
