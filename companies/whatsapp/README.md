# WhatsApp — Architecture Case Study

> "50 engineers. 2 billion users. The most efficient engineering team in the history of technology." — Industry observation

---

## Company Profile

| | |
|---|---|
| **Founded** | 2009 |
| **Scale** | 2 billion monthly active users, 190 countries |
| **Messages** | 100 billion messages per day |
| **Engineering** | ~50 engineers at $19B acquisition (2014); ~500 today |
| **Architecture** | Erlang + FreeBSD + XMPP + sharded storage |

---

## The Problem WhatsApp Was Solving

> "Jan Koum grew up in Ukraine without a phone. He immigrated to California, got his first iPhone, and thought: why is SMS so expensive? We can build a messaging app on top of data networks. The engineering challenge: make it work for 2 billion people with 50 engineers."

---

## The Core Engineering Insight

**Why 50 engineers for 2 billion users?**

Most apps scale engineers linearly with users. WhatsApp didn't — because their technology choices were optimal for the problem.

**The connection problem:**

```
The hardest problem in messaging: persistent connections.

Email: client polls the server every few minutes
  "Do I have new messages? No. Do I have new messages? No."
  Simple but creates massive polling load

WhatsApp's approach: push model via persistent TCP connections
  Client connects → TCP socket stays open
  Server pushes messages to the client immediately
  No polling needed — server knows exactly where the client is

The challenge:
  2 billion users × 1 persistent TCP connection each
  = 2 billion simultaneous open TCP sockets
  = most systems crash long before this
```

**Why Erlang solved this:**

```
Traditional web server (Java, Node.js, etc.):
  1 thread or 1 event loop per server
  Java thread: ~1MB stack, ~10K threads max per server
  10K threads × 1 connection each = 10K concurrent connections max

Erlang BEAM VM:
  Erlang process: ~2KB of memory
  100,000 Erlang processes per server is normal
  1,000,000 Erlang processes per server is possible
  Each Erlang process handles one user's connection
  = millions of concurrent connections per server

WhatsApp used FreeBSD (not Linux) because:
  FreeBSD handles more concurrent connections per CPU
  Better memory management for long-lived connections
  Each server handled 2 million concurrent TCP connections
```

---

## The Architecture

### Message Delivery Flow

```
User A (Bangkok, iPhone)
  │ TCP connection (persistent)
  ▼
WhatsApp Ejabberd Server (XMPP)
  │ Is User B online? → Check presence database
  │ Yes → Push message directly via User B's connection
  │ No  → Store in offline queue (Mnesia)
  │
  │ Message delivered → send delivery receipt to User A
  │ User B opens app → receives message + marks as read
  │ Read receipt → sent back to User A
  │
  │ Store in message log (for multi-device support)
```

### XMPP Protocol

WhatsApp built on XMPP (Extensible Messaging and Presence Protocol) — the open standard for real-time messaging:

```xml
<message from="user_a@s.whatsapp.net" to="user_b@s.whatsapp.net">
  <body>Hello!</body>
  <x xmlns="urn:xmpp:receipts">
    <request/>  <!-- please send delivery receipt -->
  </x>
</message>
```

**Why XMPP:**
- Purpose-built for presence (online/offline) + messaging
- Handles delivery receipts natively
- Battle-tested at Jabber scale since 2000
- Erlang's Ejabberd is the most performant XMPP server ever built

### Ejabberd: The Core Server

```erlang
% Erlang process per user connection (conceptual)
-module(user_session).

handle_message(FromUser, ToUser, Message) ->
  case presence:is_online(ToUser) of
    true  ->
      % Push to their active TCP connection immediately
      tcp_session:send(ToUser, Message),
      ack(FromUser, delivered);
    false ->
      % Queue for when they come online
      offline_queue:store(ToUser, Message),
      ack(FromUser, queued)
  end.
```

**Erlang's supervision trees handle failures:**
- One user's session crashes → only that Erlang process dies
- Supervisor restarts it → user reconnects automatically
- Other 1,999,999 sessions on the same server unaffected

---

## The Sharding Strategy

**Problem:** Message history for 2 billion users doesn't fit on one machine.

**WhatsApp's solution:**

```
Shard by phone number (hashed):
  hash(+66812345678) % N = shard 7
  All messages to/from this number → shard 7
  All phone number data: same phone number = same shard

Geographic sharding layer:
  EU numbers → EU shards (GDPR compliance)
  US numbers → US shards
  APAC numbers → APAC shards
  India numbers → India shards (India has own data localization laws)

Each shard:
  Primary + 2 replicas
  Primary handles writes
  Replicas handle reads
  Mnesia (Erlang's built-in distributed DB) for hot data
  MySQL for message history (cold storage)
```

**Why Mnesia for hot data:**
- Mnesia is Erlang's native distributed database
- Lives entirely in RAM (with disk backup)
- ACID transactions within Erlang nodes
- Designed to work with Erlang's process model
- "Hot data" = messages in transit, presence, session state

---

## End-to-End Encryption (2016)

**The problem:**

WhatsApp servers see every message they route. Government requests (NSA revelations, Snowden 2013) showed this was dangerous.

**The solution: Signal Protocol**

WhatsApp partnered with Open Whisper Systems to implement the Signal Protocol:

```
User A and User B each have:
  - Identity Key (permanent, used to verify identity)
  - Signed Pre-Key (medium-term, rotated monthly)
  - One-Time Pre-Keys (ephemeral, one per session)

When A wants to message B:
  A fetches B's public keys from WhatsApp server
  A + B derive a shared secret using Diffie-Hellman
  Messages encrypted with this secret
  WhatsApp servers see: ciphertext only
  Not even WhatsApp can read your messages
```

**Architecture implication:**

WhatsApp's servers now became message relay nodes — they forward encrypted bytes from A to B. They have no idea what's inside. This changed the entire architecture from "message store" to "encrypted relay."

---

## Why 50 Engineers Was Enough

**The math:**

```
2 billion users / 50 engineers = 40 million users per engineer

How is this possible?

1. No content moderation at scale (E2E encryption = can't moderate)
2. No recommendation algorithm (no feed to curate)
3. No ads (no ad tech complexity)
4. XMPP + Erlang = battle-tested, minimal custom code
5. Operational stability: Erlang systems run for years without restart

Their tech stack did the heavy lifting:
  Erlang → concurrency + fault tolerance
  FreeBSD → connection efficiency
  XMPP → messaging protocol (no need to invent)
  Mnesia → fast in-memory DB (no need to invent)
  Signal Protocol → E2E encryption (no need to invent)

Everything is building on proven shoulders. Very little custom work.
```

---

## Architecture in Clean Architecture Terms

```
WhatsApp's Architecture:

Ejabberd GenServer = Entity + Use Case
  Each user session = one Erlang process = the aggregate
  Message delivery logic = the use case inside that process

Mnesia/MySQL = Interface Adapters (outbound repositories)
  IMessageStore → Mnesia (hot messages) + MySQL (history)
  IPresenceStore → Mnesia (real-time online status)
  Use cases never interact with storage directly

XMPP Protocol Handler = Interface Adapter (inbound)
  Receives XMPP stanzas → translates to use case calls
  Returns results as XMPP stanzas

Erlang Supervisor Tree = Framework & Drivers
  Manages process lifecycle, restarts crashed sessions
  Use cases never touch the supervisor
```

---

## Lessons for Your Architecture

1. **Use boring, proven technology** — XMPP + Erlang existed; WhatsApp didn't invent them
2. **Actor model for persistent connections** — for anything where state must survive across multiple requests, actors beat REST
3. **Shard by the thing you most commonly query by** — phone number is the shard key because every WhatsApp operation starts with a phone number
4. **50 engineers beat 5,000 with the right technology** — technology choice multiplies engineering productivity
5. **E2E encryption changes your architecture** — you become a relay, not a store; embrace the simplicity

---

## Sources
- [WhatsApp's Secret Weapon: Erlang — Medium](https://ritik-chopra28.medium.com/whatsapps-secret-weapon-erlang-why-50-engineers-handle-2-billion-users-b19129a01ec9)
- [How WhatsApp Scaled to Billions of Users with Just 50 Engineers](https://singhajit.com/whatsapp-scaling-secrets/)
- [WhatsApp Erlang Architecture — ScaleWithChintan](https://scalewithchintan.com/blog/whatsapp-erlang-architecture-2-billion-users)
- [Ericsson to WhatsApp: The Story of Erlang](https://thechipletter.substack.com/p/ericsson-to-whatsapp-the-story-of)
