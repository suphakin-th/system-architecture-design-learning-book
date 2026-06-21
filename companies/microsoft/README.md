# Microsoft — Architecture Case Study

> "Azure was built because Microsoft realized: if you don't eat your own dog food at cloud scale, someone else will eat your lunch." — Microsoft Engineering culture

---

## Company Profile

| | |
|---|---|
| **Founded** | 1975 |
| **Scale** | Azure: 60+ regions, 200+ services; Teams: 320M+ users; GitHub: 100M+ developers |
| **Revenue** | $200B+ annually, cloud is the majority |
| **Engineering** | 200,000+ employees, 70,000+ engineers |
| **Architecture today** | Azure cloud-native + Service Fabric + Cosmos DB + Event Hub + gRPC |

---

## Phase 1: The Windows/Office Monolith Era (1975–2008)

**The original Microsoft architecture:**
Desktop software. Monolithic. No internet connectivity required. The "ship it on a CD" model.

**The crisis:** Amazon launched AWS in 2006. Google Apps launched in 2006. Microsoft's cash cow (Office licenses) was threatened by cloud-based alternatives. The entire business model was at risk.

**The Steve Ballmer problem:** Microsoft missed mobile (iPhone 2007) and early cloud. Ballmer's focus on Windows/Office monopoly prevented innovation. The entire organization was structured around desktop software.

---

## Phase 2: Azure — The Platform Bet (2008–2014)

**The decision:** Build a cloud platform competitive with AWS, from scratch, in a company that had never run cloud services.

**The architecture challenge:**

> "Microsoft had to build the equivalent of Amazon's entire AWS from scratch — but faster. We had one advantage: Windows Server, SQL Server, .NET, and Active Directory were already inside every enterprise. We made Azure deeply integrated with all of them. That was our differentiation: 'Azure works with your existing Microsoft investment.'"

**Azure Service Fabric (2014):**

Microsoft needed an orchestrator for their own services (Azure SQL, Azure Storage, etc.) before Kubernetes existed. They built Service Fabric:

```
Service Fabric vs Kubernetes:
  Kubernetes: container orchestration (stateless preferred)
  Service Fabric: stateful service orchestration (actors with persistent state)

  Azure's use case: Azure SQL Database has state (your database!)
    → Service Fabric actor model: each SQL database instance = one actor
    → Actor knows its state (which rows, which transactions)
    → Service Fabric handles placement, failover, scaling

  Microsoft's pattern: Reliable Actors = Erlang processes in .NET
    Stateful, isolated, addressable, supervised
    Orleans (open-source) = Service Fabric actors for .NET developers
```

---

## The Teams Architecture: 300M Users During COVID

**The problem:**

Teams launched in 2017. Microsoft Teams had 32 million daily users in March 2020. By April 2020 (COVID lockdowns): **75 million daily users — 2.3× in 3 weeks.**

```
The scaling challenge:
  March 2020: 32M daily users
  April 6, 2020: 75M daily users
  Growth: 43M new users in 3 weeks

  Each user: 5-10 open WebSocket connections
    (presence, chat, notifications, meetings)

  75M users × 7 connections = 525M persistent connections

  Mitigation:
    Azure Kubernetes Service: auto-scale horizontally
    Azure Event Hub (Kafka-compatible): process chat events at scale
    Azure Cosmos DB: globally distributed presence data
    Azure CDN: static assets cached at edge (reduce origin load)
```

**The architecture that survived:**

```
Teams message flow:
  User sends message → Teams WebSocket API Gateway
    → Azure Event Hub (Kafka-compatible stream)
      → Message Processing Service (Azure Functions)
        → Azure Cosmos DB (message storage, global consistency)
          → Notification Service → push to all participants' connections

Why Cosmos DB for Teams:
  Messages are global (Tokyo user in meeting with Paris user)
  Cosmos DB: multi-region writes, reads always from nearest replica
  Consistency level: Session (your own messages appear immediately)
  Other users' messages: bounded staleness (within 100ms)
```

---

## Azure Cosmos DB: The Multi-Model Database

**The problem Cosmos DB solved:**

> "Enterprises use MongoDB, Cassandra, Redis, Gremlin (graph), and Table storage. Each requires different SDK, different operations team, different expertise. Cosmos DB: one service, one operations team, supports all APIs."

```
Cosmos DB multi-model:
  MongoDB API: document store (JSON)
  Cassandra API: wide-column store (CQL)
  Gremlin API: graph database
  Table API: Azure Table Storage (key-value)
  SQL/Core API: document with SQL query support

All backed by the same Cosmos DB infrastructure:
  Global distribution: replicate to 60+ Azure regions
  Multiple consistency levels: Strong → Bounded Staleness → Session → Eventual
  Auto-sharding: horizontal scaling is invisible to the application
```

**In Clean Architecture terms:**

```typescript
// Teams Message Service — use case
class SendMessageUseCase {
  constructor(
    private messageRepo: IMessageRepository,  // port
    private eventBus: IEventBus,              // port
    private presenceService: IPresenceService // port
  ) {}

  async execute(req: SendMessageRequest) {
    const message = Message.create(req.channelId, req.text, req.senderId);
    await this.messageRepo.save(message);  // → Cosmos DB adapter
    await this.eventBus.publish(new MessageSentEvent(message));  // → Event Hub adapter
    return { messageId: message.id };
  }
}

// Adapters in Infrastructure layer
class CosmosMessageRepository implements IMessageRepository { /* Cosmos SDK */ }
class EventHubEventBus implements IEventBus { /* Event Hub SDK */ }
```

---

## GitHub Architecture (Post-Acquisition, 2018)

**GitHub at acquisition:**
- 85M+ repositories
- 30M+ developers
- Ruby on Rails monolith ("Ghe" — GitHub Enterprise)
- MySQL for core data
- Elasticsearch for code search

**Post-acquisition architecture evolution:**

```
The code search problem:
  GitHub had 200M+ repositories × average 100K lines = 20 quadrillion lines
  Finding all uses of a function across all open source code:
    Old: grep through all repos serially → impossible at scale
    New: Blackbird (GitHub's code search engine, 2022)

Blackbird architecture:
  Trigram index: every 3-character sequence in every file indexed
  Custom inverted index: stored in memory for speed
  Distributed across 100s of machines
  Query: "HttpClient usage" → trigrams → candidate files → grep → results
  Result: search 500M+ code files in <1 second
```

**Actions and the CI/CD architecture:**

```
GitHub Actions: runs CI/CD for 100M+ repositories
  Each job: container on ephemeral runner
  Runners: Azure VMs that spin up on demand

  Scale: 1M+ concurrent CI/CD jobs during peak
  (Every developer pushes code → CI runs → 1M simultaneous builds)

  Architecture:
    Push to GitHub → webhook → Actions queue (Azure Service Bus)
    → Runner picker (finds available runner) → runner assigned
    → Container started with workflow YAML
    → Logs streamed back to GitHub (WebSocket)
    → Status updated in GitHub commit
```

---

## Key Technologies

| Problem | Solution | Why |
|---|---|---|
| Global data for Teams | Cosmos DB | Multi-region, tunable consistency |
| Event streaming | Azure Event Hub | Kafka-compatible, fully managed |
| Container orchestration | AKS (Kubernetes) | Standard, integrates with Azure |
| Stateful services | Service Fabric / Orleans | Actors with persistent state |
| CDN | Azure CDN / Akamai | 60+ regions, low-latency edge |
| Code search | Blackbird (custom) | 20+ quadrillion lines can't use Elasticsearch |
| Identity | Azure Active Directory | SSO for enterprise (240M+ orgs) |

---

## Lessons for Your Architecture

1. **Existing customer base is an architectural constraint** — Azure's deepest differentiation: works with existing Microsoft investments (Active Directory, SQL Server, .NET)
2. **COVID-scale events are planning problems** — Teams survived 2.3× growth in 3 weeks because of elastic cloud architecture designed for it
3. **Multi-model databases reduce operational complexity** — Cosmos DB's multiple APIs mean one ops team instead of five
4. **Code search at GitHub scale requires custom solutions** — Blackbird replaced Elasticsearch because no off-the-shelf tool handles 20 quadrillion lines

---

## Sources
- [Cloud Design Patterns — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/)
- [Reinventing hybrid cloud integration at Microsoft](https://www.microsoft.com/insidetrack/blog/reinventing-hybrid-cloud-integration-at-microsoft-from-months-to-one-day/)
- [Design patterns for microservices — Microsoft Azure Blog](https://azure.microsoft.com/en-us/blog/design-patterns-for-microservices/)
