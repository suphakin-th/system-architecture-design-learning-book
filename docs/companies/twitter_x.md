# Twitter/X — Architecture Case Study

> "The Fail Whale was not a design choice. It was a warning that our architecture couldn't survive our own success." — Twitter Engineering

---

## Company Profile

| | |
|---|---|
| **Founded** | 2006 |
| **Scale** | 250M+ daily active users, 500M+ tweets per day |
| **Peak** | 143,199 tweets per second (Japan World Cup 2014) |
| **Architecture today** | Microservices + Finagle + Manhattan + Redis timeline cache |

---

## The Senior Architect Explains the Core Problem

> "Twitter has one deceptively simple problem: 'I have 50 million followers. I just tweeted. Every follower needs to see it in their timeline. Now.' That's 50 million database writes in milliseconds. Most architectures collapse under this. Twitter's entire history is the story of solving this problem."

---

## Phase 1: The Ruby on Rails Monolith (2006–2011) — The Fail Whale Era

**The original architecture:**

```
Web Client → Ruby on Rails app → MySQL (tweets table, users table, follows table)
```

**Why it was catastrophically bad at scale:**

```sql
-- Getting YOUR timeline in 2008:
SELECT tweets.*
FROM tweets
JOIN follows ON follows.followed_id = tweets.user_id
WHERE follows.follower_id = YOUR_USER_ID
ORDER BY tweets.created_at DESC
LIMIT 20;

-- This query:
-- Joins the tweets table (100M+ rows)
-- With the follows table (1B+ rows)
-- Sorts by timestamp
-- At 250M users all opening Twitter simultaneously → DEAD
```

**The "Fail Whale":** When MySQL couldn't keep up, the entire site went down. Twitter displayed a cartoon whale being held up by birds — the "Fail Whale" became a cultural symbol of a company growing faster than its architecture.

---

## Phase 2: The Fanout Decision — Push vs Pull

**The fundamental architecture choice:**

```
Option A: Fanout on Read (Pull)
  When you open Twitter:
    Query the 500 people you follow
    Get their recent tweets
    Sort and return

  Problem: 500 queries per timeline open × 250M users = billions of queries/second
  Benefit: Writing a tweet is fast (one insert)

Option B: Fanout on Write (Push)
  When someone tweets:
    Get all their followers (could be 50 million for @elonmusk)
    Write the tweet ID to each follower's timeline cache
    Timeline open = one Redis read

  Problem: Lady Gaga tweets → write to 70M follower caches simultaneously
  Benefit: Reading a timeline is a single Redis lookup = fast
```

**Twitter's answer: Hybrid approach**

```
Regular users (< ~10,000 followers):
  Fanout on write: tweet → push tweet_id to all followers' Redis sorted sets
  Timeline read: fetch from Redis (one operation)

"Celebrity" users (@taylorswift13, @elonmusk, etc., > 10,000 followers):
  NO fanout: tweet stored in celebrity tweet table only
  Timeline read: Redis timeline + merge with followed celebrities' recent tweets

Why 10,000 as the threshold?
  Fanout to 10,000 followers = 10,000 Redis writes = ~50ms (acceptable)
  Fanout to 50,000,000 followers = would take minutes; queue backs up
```

---

## Phase 3: The Redis Timeline Cache

**The pre-computed timeline:**

```
Your timeline is pre-computed and stored in Redis as a sorted set:
  Key:   timeline:{user_id}
  Score: tweet timestamp (Unix timestamp for sorting)
  Value: tweet_id

When you open Twitter:
  ZREVRANGE timeline:YOUR_USER_ID 0 19  # get 20 most recent tweets
  → Redis returns 20 tweet IDs in <1ms
  → Parallel fetch of tweet content from Manhattan (key-value store)
  → Returned to you

When your friend tweets:
  ZADD timeline:YOUR_USER_ID {timestamp} {tweet_id}
  → Repeated for all 500 of your friend's followers
  → ZREMRANGEBYSCORE to trim to 800 most recent (memory management)
```

**The Elon Musk problem (2023):**

> "Elon tweets. 150M followers. Fanout service gets backed up. Tweets delayed 10-30 minutes to reach all followers. The system was NOT designed for someone with 150M followers who also owns the company and tweets constantly."

**The actual solution in 2023:**
Twitter (under Elon's ownership) gave @elonmusk's tweets priority processing in the ranking algorithm. His tweets were force-ranked to the top of follower timelines — bypassing the normal fanout queue.

This caused significant negative press but solved the technical problem of Elon seeing his own tweets' metrics be slower than other accounts.

---

## Phase 4: From Ruby to JVM — The Finagle RPC Framework

**Why Twitter rewrote everything in Scala/Java:**

```
Ruby limitations at scale:
  - Global Interpreter Lock (GIL): only one Ruby thread runs at a time
  - No true concurrency for CPU-bound work
  - Garbage collection pauses cause latency spikes

Twitter's answer: Finagle (open-sourced 2011)
  - RPC library for Scala/Java services
  - Built-in: load balancing, circuit breaking, retry, timeout
  - Designed for composing distributed services
  - "Service" in Finagle is a function: Request → Future[Response]
```

**The Twitter service architecture:**

```
User Request → Routing (Finagle) → TweetService (Scala)
                                   UserService (Scala)
                                   TimelineService (Scala)
                                   SearchService (Scala — ElasticSearch)
                                   TrendService (Scala)
```

---

## The Search Architecture

**Problem:** 500M+ tweets per day, all searchable, in real-time.

```
Tweet written → Kafka → EarlyBird (Lucene-based indexer)
EarlyBird: keeps last 7 days of tweets in memory for real-time search
  Index per shard: most recent tweets for ~1/N of users
  Search: fan out to all EarlyBird shards, merge results, return

For historical search (older than 7 days):
  Manhattan → periodic export → Hadoop → offline search index
```

**Why in-memory:** Disk-based search for 500M tweets/day is too slow. EarlyBird keeps the recent index entirely in RAM.

---

## Manhattan — Twitter's Distributed Key-Value Store

**Why Twitter built their own DB:**

```
Tweets require:
  - High write throughput (500M tweets/day)
  - Read throughput (250M users opening timelines)
  - Durability (tweets should never be lost)
  - Low latency (milliseconds for timeline fetch)

Existing options evaluated:
  MySQL: too slow for this write volume
  Cassandra: eventual consistency caused tweet ordering issues
  DynamoDB: AWS vendor lock-in concern; not enough control

Manhattan (2014):
  Custom key-value store built on LevelDB (LSM tree)
  Consistent hashing for sharding
  Strong consistency within a shard
  Multi-datacenter replication for disaster recovery
  Handles: tweets, user data, timelines, direct messages
```

---

## Architecture in Clean Architecture Terms

```
Twitter's Architecture:

Timeline Service = Use Case layer
  GetTimeline use case: reads from ITimelineRepository
  ITimelineRepository → RedisTimelineRepository (pre-computed) OR
                      → RealTimeComputedTimelineRepository (fallback)
  Use case doesn't know which implementation is used

Fanout Service = Use Case with side effects
  PublishTweet use case → calls IFanoutPort → fans out to followers
  IFanoutPort → RedisTimelineFanout (for regular users) OR
              → CelebrityTweetStore (for celebrity users)
  Strategy pattern: fanout strategy selected based on follower count

Manhattan = Infrastructure layer (outbound adapter)
  ITweetRepository → ManhattanTweetRepository
  Use case: storeTweet(tweet) → ITweetRepository.save(tweet)
  Manhattan details (LSM tree, sharding) hidden in adapter

Finagle = Framework & Drivers layer
  The RPC framework; services are composed of Finagle services
  Circuit breaking, load balancing happen in Finagle
  Use case logic never touches Finagle directly
```

---

## Lessons for Your Architecture

1. **The fanout problem is universal** — any "one-to-many notification" system faces it; hybrid push/pull is the standard answer
2. **Pre-computed reads beat real-time reads at scale** — Twitter's Redis cache: one read; MySQL join: system crash
3. **Celebrity users break your assumptions** — design for power-law distributions; 1% of users drive 99% of load
4. **Language/runtime choice matters for concurrency** — Ruby GIL prevented true parallelism; Scala/JVM solved it
5. **Sometimes you build your own DB** — when existing options don't meet your SLAs, build specialized storage

---

## Sources
- [The Architecture Twitter Uses to Deal with 150M Active Users — High Scalability](http://highscalability.com/blog/2013/7/8/the-architecture-twitter-uses-to-deal-with-150m-active-users.html)
- [Twitter's Fanout Strategy at Scale — DEV Community](https://dev.to/gabrielanhaia/twitters-fanout-strategy-at-scale-the-trade-off-most-designs-miss-55oa)
- [Twitter Engineering — Tough Architectural Decisions](https://softwareengineeringwk.substack.com/p/twitter-architecture)


---

## Architecture Diagram

![diagram.svg](diagram.svg)

