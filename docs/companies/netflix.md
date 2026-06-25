# Netflix - Architecture Case Study

> "Netflix's greatest achievement isn't streaming video - it's building a system where any part can fail and users never notice." - Engineering observation

---

## Company Profile

| | |
|---|---|
| **Founded** | 1997 (DVD mail), 2007 (streaming) |
| **Scale** | 260M+ subscribers, 190+ countries |
| **Peak load** | 65M+ concurrent streams, 15% of global internet traffic |
| **Engineering** | 700+ microservices, 1000+ engineers |
| **Architecture today** | Microservices + Service Mesh + CQRS + Event-Driven |

---

## The Senior Architect Tells the Story

> "Netflix is the definitive case study for why you need to plan for failure at the architecture level. Not 'log errors nicely' level - 'assume every single dependency WILL fail' level. Their architecture decisions come from one core belief: your system is always partially broken, and users should never see it."

---

## Phase 1: The Monolith (1997-2008)

**The DVD-era monolith:**
- Java monolith called the "NCCP" (Netflix Content Delivery Platform)
- Oracle database for all data
- Hosted in a single data center in Hillsboro, Oregon

**The breaking point (August 2008):**
> A database corruption event in their single data center caused Netflix to be completely unavailable for 3 days. DVDs couldn't ship. Customers couldn't be served. Revenue stopped.

**The decision:**
> "We will never have a single point of failure again. We will move to AWS and microservices. We will assume we're always broken."

This 2008 outage changed software architecture forever.

---

## Phase 2: Migration to AWS + Microservices (2008-2016)

**The 7-year migration:**
- Gradually extracted features from the NCCP
- Each extracted feature became a microservice on AWS
- Used the Strangler Fig pattern - NCCP shrank as services were extracted
- By 2016: fully on AWS, 700+ microservices, NCCP decommissioned

**Key services extracted:**
```
User Service -> account management, authentication
Viewing History -> what you've watched
Recommendation Engine -> what to watch next (ML models)
Search Service -> find content
Streaming Delivery -> actual video bytes (Open Connect CDN)
Billing Service -> payment processing
Content Service -> metadata (title, description, ratings)
```

---

## The Chaos Engineering Revolution

**The problem with "test for success":**
> "We'd test that everything worked. But in production, things fail in ways you never imagined. A network partition between the recommendation service and the streaming service - our test suite never covered that. Chaos Monkey was born."

**Chaos Monkey (2011):**
Netflix built a tool that randomly kills production servers. Every day. On purpose.

```
Chaos Monkey -> randomly terminates EC2 instances
Chaos Gorilla -> kills entire Availability Zones
Chaos Kong -> kills entire AWS Regions

If your system can survive random component death,
it's truly resilient - not just theoretically resilient.
```

**Result:** Netflix's architecture is designed around "what happens when THIS breaks" not "what happens when this works."

**Junior asks:** "Why would you break your own production servers?"

**Senior answers:** "Because they WILL break. The only question is: does it happen during a chaos test at 2pm on Tuesday when engineers are watching, or does it happen at 2am on New Year's Eve when 65 million people are watching the countdown? Choose your failure."

---

## The Architecture That Makes Failure Invisible

### Fallbacks at Every Level

```typescript
// Recommendation Service with fallbacks
class GetRecommendationsUseCase {
  async execute(userId: string): Promise<Movie[]> {
    try {
      // Level 1: Personalized ML recommendations
      return await this.mlRecommendationService.getPersonalized(userId);
    } catch {
      try {
        // Level 2: Pre-computed popular movies for this user's region
        return await this.cache.getRegionalTopMovies(userId);
      } catch {
        // Level 3: Globally popular movies (static list, always available)
        return this.staticFallback.getTopMovies();
      }
    }
  }
}
// User sees slightly less personalized recommendations,
// but they ALWAYS see something. Netflix never shows an error.
```

**This is the Bulkhead Pattern** - like compartments in a ship, isolate failures so they don't sink the whole vessel.

### Circuit Breaker Pattern (Hystrix)

Netflix invented Hystrix - the most widely used circuit breaker library:

```
State: CLOSED (normal)
  All requests flow through
  If failure rate > 50% in 10 seconds -> OPEN

State: OPEN (tripped)
  All requests immediately return fallback
  After 5 seconds -> HALF-OPEN

State: HALF-OPEN (testing recovery)
  One request allowed through
  If success -> CLOSED
  If failure -> OPEN again
```

> "Without circuit breakers, one slow downstream service blocks all threads waiting for its response. Eventually ALL your threads are waiting, your service runs out of threads, and YOU become the slow service for your upstreams. The failure cascades. With circuit breakers, you detect the problem and stop calling the broken service immediately."

---

## CQRS for Content Delivery

**The problem:** Netflix catalog has 15,000+ titles. Reading catalog data (search, browse) is 1000x more frequent than writing it (adding new titles).

**Solution: CQRS**

```
Write side (Content Team uploads new movie):
  AdminService -> ContentMetadataService -> PostgreSQL (normalized)
 -> Triggers event: ContentPublished

Read side (Users searching/browsing):
  ContentPublished event -> fan out to:
    ElasticsearchProjection -> search index (fast full-text search)
    RedisProjection -> most-viewed cache
    S3Projection -> CDN-ready JSON files per region
    PersonalizationProjection -> ML training data

User searches for "action movies":
 -> Elasticsearch (pre-built search index)
 -> Returns in 50ms
 -> No join to PostgreSQL needed
```

---

## Open Connect CDN - The Real Secret

**The problem no one talks about:** 15% of global internet traffic. Moving video bytes from Netflix data centers to your TV through the public internet = unacceptably high latency and cost.

**The solution:** Netflix built their own CDN with 18,000+ servers in 1,000+ locations worldwide.

```
You press Play on a movie:
  Netflix API -> determines nearest Open Connect Appliance (OCA)
  Your ISP (True, AIS, DTAC in Thailand) -> has a Netflix OCA in their network
  Video bytes come from the OCA in your ISP's building
  Round trip: <5ms instead of 200ms+ to California
```

**Business logic:** Netflix partners with ISPs to place their hardware INSIDE ISP networks for free. ISPs benefit (less transit bandwidth needed). Netflix benefits (lower CDN cost, better quality).

---

## The 2024 Live Streaming Problem

**Mike Tyson vs Jake Paul (65M concurrent viewers, Nov 2024):**

Netflix's VOD architecture was NOT designed for live streaming. Problems:

1. **Manifest management:** VOD has a pre-generated playlist. Live has a constantly updating manifest that 65M clients are polling simultaneously.
2. **CDN caching:** VOD segments are cacheable. Live segments are unique per second - CDN can't help.
3. **Real-time encoding:** Must encode 4K -> 8 quality levels in <2 seconds.

**Resolution:** The microservices architecture meant live streaming issues didn't affect VOD. Subscribers watching normal content experienced no degradation. Only live viewers were affected.

> "This is fault isolation in practice. Our microservices architecture protected 260M VOD subscribers from a live streaming capacity issue. In the monolith era, the whole Netflix would have gone down."

---

## Architecture in Clean Architecture Terms

```
Netflix Architecture = Clean Architecture at massive scale

Each microservice:
  domain/       User, Movie, WatchSession, Recommendation (entities)
  application/  GetRecommendations, StreamVideo, UpdateWatchHistory (use cases)
  adapters/     REST controllers + Cassandra/DynamoDB repos + Kafka publishers
  infra/        Hystrix circuit breaker, Ribbon load balancer, Eureka discovery

Hystrix = infrastructure layer - use cases never call Hystrix directly
Eureka (service discovery) = framework layer - services don't know about it
Ribbon (load balancing) = framework layer - transparent to business logic
```

---

## Key Technologies and Why

| Problem | Solution | Why |
|---|---|---|
| Service discovery | Eureka | Services register themselves; clients discover via registry |
| Load balancing | Ribbon | Client-side LB; knows which instances are healthy |
| Circuit breaking | Hystrix | Prevents cascading failures; most important Netflix invention |
| API Gateway | Zuul | Auth, routing, rate limiting; now moving to Envoy |
| Messaging | Kafka | Watching history, viewing events, analytics |
| Storage: viewing | Apache Cassandra | Always available; no single point of failure; multi-region |
| Storage: metadata | MySQL on RDS | ACID transactions for content metadata |
| Search | Elasticsearch | Full-text search across 15,000+ titles |
| ML models | Spark + SageMaker | Train recommendation models on viewing history |
| Video delivery | Open Connect (own CDN) | 18,000 servers in 1,000+ locations |

---

## Lessons for Your Architecture

1. **Design for failure first** - assume every dependency will fail; build fallbacks
2. **Circuit breakers are mandatory** - not optional - in microservices
3. **Read/write separation (CQRS)** - 1000:1 read/write ratio = separate models
4. **Chaos testing in production** - theoretical resilience != actual resilience
5. **Graceful degradation** - less personalized > no content; always show something
6. **Infrastructure is a product** - Netflix open-sourced Hystrix, Eureka, Ribbon because they built them as reusable tools, not hacks

---

## Sources
- [A Brief History of Scaling Netflix - ByteByteGo](https://blog.bytebytego.com/p/a-brief-history-of-scaling-netflix)
- [Netflix Microservices Engineering Blog](https://netflixtechblog.com/)
- [Netflix Architecture Case Study - Clustox](https://www.clustox.com/blog/netflix-case-study/)
- [How Discord Scaled Elixir to 5M Concurrent Users](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)


---

## Architecture Diagram

![diagram.svg](diagram.svg)

