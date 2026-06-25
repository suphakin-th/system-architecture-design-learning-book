# Discord - Architecture Case Study

> "We chose Elixir because the Erlang VM was literally built for what Discord is: millions of persistent connections, real-time messaging, fault tolerance." - Discord Engineering

---

## Company Profile

| | |
|---|---|
| **Founded** | 2015 |
| **Scale** | 200M+ registered users, 19M daily active servers |
| **Messages** | Trillions of messages stored |
| **Connections** | 5M+ concurrent WebSocket connections |
| **Architecture today** | Elixir (real-time) + Rust (data services) + ScyllaDB (messages) |

---

## The Problem Discord Was Built to Solve

> "Every gaming session needs voice chat. But every voice chat solution in 2015 was terrible - Skype crashed, TeamSpeak was complicated, Mumble looked like a 2004 website. We needed: always on, always available, no accounts to set up, <50ms voice latency, supports 100 people in one channel."

---

## Phase 1: Why Elixir? (2015)

**The connection problem:**

Discord is fundamentally different from Twitter or Facebook. Users don't open Discord, do something, and close it. Users leave Discord OPEN all day. Your entire friend group is connected simultaneously. This means:

```
Twitter: User opens app -> request -> response -> close
  Connections: burst, then close
  Model: stateless HTTP is fine

Discord: User opens app -> WebSocket -> stays open for 8 hours
  Connections: persistent, stateful, always open
  Model: stateless HTTP is WRONG; need stateful actor model
```

**Why not Node.js?**

> "Node.js runs on a single thread with an event loop. It handles 10K concurrent connections fine. At 100K, garbage collection pauses cause jitter - users hear audio glitches. At 1M, Node.js becomes unstable. We needed something designed for this."

**Why Elixir (Erlang VM):**

```
Erlang was built by Ericsson in 1986 for telephone switching systems.
The requirements then: same as Discord's requirements now.
  - Millions of concurrent connections (phone calls)
  - Must not crash (people can't lose calls)
  - Real-time: <50ms response (voice is unforgiving)
  - Hot code reload: update the system without dropping calls

Erlang/Elixir processes:
  - Lightweight: 2KB per process (vs 1MB per OS thread)
  - 1 server: handles 2M Elixir processes (vs 1K OS threads)
  - Isolated: one crash kills one process, not the server
  - Message-passing: no shared state -> no race conditions
```

**Result:** Discord scaled Elixir to 5 million concurrent users on surprisingly few servers.

---

## Phase 2: The Cassandra Nightmare (2017-2022)

**The initial decision:**
Discord chose Apache Cassandra to store messages. The reasoning was sound:
- Messages are append-only (insert-heavy, rarely updated)
- Cassandra excels at high write throughput
- Cassandra uses consistent hashing internally (auto-sharding)
- No schema migrations needed (flexible schema)

**The problems:**

```
By 2022:
  - 177 Cassandra nodes
  - Trillions of messages
  - Hot partitions causing latency spikes

The hot partition problem:
  Discord servers (communities) are sharded by server_id
  Popular servers (Reddit, gaming communities) = millions of messages
  All messages for r/gaming go to the same Cassandra partition
  That partition = HOTSPOT -> all reads/writes queue up -> latency spikes

p99 read latency: 40-125ms (users can see noticeable lag)
p99 write latency: 5-70ms (inconsistent, spiky)
Engineering time: constant firefighting of Cassandra issues
```

**The senior architect's perspective:**

> "Cassandra saved us at 1 billion messages. At 1 trillion messages, it became our biggest problem. This is normal in architecture: the solution to problem A creates problem B at scale. The question is always: what are the failure modes at 10x current scale? Cassandra's failure mode was hot partitions, and we could see that becoming catastrophic."

---

## Phase 3: Migration to ScyllaDB + Rust (2022-2023)

**Why ScyllaDB:**
- Same Cassandra Query Language (CQL) - no application code changes
- Written in C++ (not Java) - no JVM garbage collection pauses
- Uses the same consistent hashing approach - same data model
- Per-core architecture - one core handles one set of shards (no thread contention)

**The hot partition solution - Bucket Layer:**

```
Old shard key: channel_id
  All messages for one channel -> one partition -> hotspot

New shard key: channel_id + bucket
  bucket = message_id / (10 days worth of messages)
  Messages spread across multiple partitions over time
  Hot channels spread their load across multiple partitions

Before: channel 12345 -> partition 12345 (hot)
After:  channel 12345, bucket 1 -> partition A
        channel 12345, bucket 2 -> partition B
        channel 12345, bucket 3 -> partition C
        Load is distributed across 3 partitions
```

**Why Rust for the data service layer:**

> "Between our Elixir application and ScyllaDB, we built a Rust data service layer. Rust gives us: no garbage collector (no pauses), zero-cost abstractions, memory safety without runtime overhead. For the hot path between our app and DB, we need deterministic performance. Rust delivers that."

**Results:**
- p99 read latency: **15ms** (was 40-125ms -> 4-8x improvement)
- p99 write latency: **5ms** (was 5-70ms -> consistent, not spiky)
- Node count: reduced from 177 Cassandra nodes to fewer ScyllaDB nodes

---

## The Voice Architecture (The Core Product)

**How Discord voice works:**

```
User A (Bangkok) -> Discord Edge Server (Singapore)
  WebSocket: control plane (who's talking, muting, joining)
  UDP: audio data (real-time, loss-tolerant)

Discord Edge Server (Singapore)
  Receives audio from User A
  Forwards to all other users in the channel
  (Selective Forwarding Unit = SFU)

Why UDP not TCP for audio:
  TCP: guarantees delivery, retransmits lost packets
  Audio: a packet from 100ms ago is useless even if retransmitted
  Better to play silence than wait for a retransmit
  UDP: fires and forgets, handles loss at application level
```

**Elixir for voice control plane:**

```elixir
# Each voice channel is an Elixir GenServer (actor)
defmodule VoiceChannel do
  use GenServer

  def join(channel_id, user_id) do
    GenServer.cast(via(channel_id), {:join, user_id})
  end

  def handle_cast({:join, user_id}, state) do
    # Notify all existing users of new joiner
    Enum.each(state.users, fn uid ->
      send_message(uid, {:user_joined, user_id})
    end)
    {:noreply, %{state | users: [user_id | state.users]}}
  end
end
# If this GenServer crashes: supervisor restarts it, users reconnect
# One crash = one channel affected, not the whole server
```

---

## Architecture in Clean Architecture Terms

```
Discord's Architecture:

Elixir GenServer = Entity + Use Case
  VoiceChannel GenServer = the Voice Channel entity + all its use cases
  Each GenServer IS the aggregate root for its channel

ScyllaDB/Rust layer = Interface Adapter (outbound)
  IMessageRepository -> ScyllaDB Rust client
  Use cases call IMessageRepository.save(message)
  ScyllaDB details (shard key, bucket) hidden in the adapter

WebSocket Handler = Interface Adapter (inbound)
  Receives WebSocket events, translates to use case calls
  Sends use case results back as WebSocket messages

Elixir Supervisor = Framework & Drivers layer
  Manages GenServer lifecycle
  Restarts crashed processes automatically
  Use cases never interact with the supervisor directly
```

---

## Lessons for Your Architecture

1. **Match technology to the problem** - Erlang/Elixir exists for exactly Discord's problem (persistent connections, actor model)
2. **Cassandra isn't always the answer** - it saved Discord at 1B messages, hurt them at 1T messages
3. **Design the shard key around access patterns** - the bucket layer solved Cassandra's hotspot problem
4. **Language choice matters for specific problems** - Rust's predictability in the hot data path vs Elixir's concurrency for the control plane
5. **Supervisors = architectural resilience** - Elixir supervisors let you isolate failures to one actor

---

## Sources
- [How Discord Scaled Elixir to 5,000,000 Concurrent Users](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
- [Scaling Trillions of Messages: Discord's Journey from Cassandra to ScyllaDB](https://seifrajhi.github.io/blog/discord-cassandra-to-scylladb/)
- [How Discord solved the Hot Partition Problem](https://engineeringatscale.substack.com/p/how-discord-solved-hot-partition-problem)
- [Discord Engineers Add Distributed Tracing to Elixir's Actor Model](https://www.infoq.com/news/2026/03/discord-elixir-actor-tracing/)


---

## Architecture Diagram

![diagram.svg](diagram.svg)

