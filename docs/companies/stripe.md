# Stripe — Architecture Case Study

> "Our job is to abstract away the complexity of payments so developers never have to think about banking infrastructure again." — Patrick Collison

---

## Company Profile

| | |
|---|---|
| **Founded** | 2010 |
| **Scale** | 500B+ transactions processed, 40+ countries, used by 3M+ businesses |
| **Processing** | ~$1T in payments annually |
| **Engineering** | 8,000+ employees, 4,000+ engineers |
| **Architecture today** | Event Sourcing + CQRS + Idempotency-first design |

---

## What Problem Stripe Solved

> "In 2010, accepting credit cards online required: applying to a merchant bank (6-8 weeks), getting a merchant account, signing a contract, installing a payment gateway, handling PCI DSS compliance, maintaining the SSL certificates, and praying nothing broke. Stripe reduced this to: `npm install stripe` and 7 lines of code."

The engineering problem behind this simplicity is enormous.

---

## The Core Engineering Challenge: Money Must Not Get Lost

**The fundamental constraint of financial systems:**

```
A regular web app: user submits form → if request fails → user retries → harmless

A payments app: user pays $100 → if request fails → retry?
  Case 1: First request failed before charging → retry = charge once ✓
  Case 2: First request charged but response lost → retry = charge TWICE ✗
  Case 3: First request charged, user already has receipt → retry = confuse user ✗

The dual-write problem:
  Action: charge $100
  Steps:
    1. Debit user's card ($100 leaves their account)
    2. Credit merchant's account ($100 arrives in their account)

  If Step 1 succeeds and Step 2 fails:
    User is charged $100
    Merchant receives nothing
    Money is LOST IN THE SYSTEM
    This is a financial liability
```

**Stripe's answer to these problems:**

1. **Idempotency Keys** — solve the retry problem
2. **Event Sourcing** — solve the dual-write problem
3. **Distributed Locking** — prevent concurrent operations on the same object
4. **Reconciliation Jobs** — catch and fix any inconsistencies

---

## Idempotency: The Core Design Principle

**What idempotency means:**

> "An operation is idempotent if running it N times has the same result as running it once. In payments: charging a card twice should be detected and the second charge should be a no-op."

**How Stripe implements it:**

```
Client generates a unique idempotency key:
  key = UUID() → "idem_2XjA8mNpQrL4wBvZ"

API call:
  POST /v1/charges
  Idempotency-Key: idem_2XjA8mNpQrL4wBvZ
  {
    "amount": 10000,  // $100.00 in cents
    "currency": "usd",
    "source": "tok_visa"
  }

Stripe server:
  1. Hash the Idempotency-Key
  2. Check Redis: does key idem_2XjA8mNpQrL4wBvZ exist?
     - Yes → return the stored response (do NOT process again)
     - No  → acquire distributed lock for this key
             → process the charge
             → store result in Redis with key
             → release lock
             → return result

Client network timeout, retries:
  Same Idempotency-Key → Stripe returns stored result
  Card charged EXACTLY once, regardless of how many retries
```

**The implementation in Clean Architecture terms:**

```typescript
class ChargeCardUseCase {
  constructor(
    private idempotencyStore: IIdempotencyStore,  // port
    private paymentGateway: IPaymentGateway,       // port
    private chargeRepo: IChargeRepository          // port
  ) {}

  async execute(req: ChargeRequest): Promise<ChargeResponse> {
    // Check idempotency key
    const existing = await this.idempotencyStore.get(req.idempotencyKey);
    if (existing) return existing;  // Already processed — return cached result

    // Acquire distributed lock
    await this.idempotencyStore.lock(req.idempotencyKey);

    try {
      // Process charge
      const charge = await this.paymentGateway.charge(req.amount, req.paymentMethod);
      await this.chargeRepo.save(charge);

      const response = ChargeMapper.toResponse(charge);
      await this.idempotencyStore.store(req.idempotencyKey, response, TTL_24_HOURS);
      return response;
    } finally {
      await this.idempotencyStore.unlock(req.idempotencyKey);
    }
  }
}
```

---

## Event Sourcing for the Payment Ledger

**Why Stripe uses Event Sourcing:**

```
Traditional DB approach for payments:
  charges table:
    id | amount | currency | status | customer_id | updated_at
    1  | 10000  | usd      | succeeded | cus_123  | 2024-01-15

  Problem: We know the CURRENT state, but not HOW we got there.
  - Was there a failed attempt before it succeeded?
  - Was it refunded and re-charged?
  - What was the authorization code?
  - Regulators need 7 years of history — you can't just show current state.

Event Sourcing approach (Stripe's reality):
  payment_intent_events:
    PaymentIntentCreated     { amount: 10000, currency: 'usd' }
    PaymentMethodAttached    { payment_method: 'pm_card_visa' }
    PaymentIntentConfirmed   { ... }
    ChargeAttempted          { ... }
    ChargeFailed             { decline_code: 'insufficient_funds' }
    ChargeAttempted          { ... }  // second try with different card
    ChargeSucceeded          { charge_id: 'ch_abc', amount_captured: 10000 }
    RefundCreated            { amount: 5000 }  // partial refund
    RefundSucceeded          { ... }

Current state = replay of all these events
Full audit trail: every cent, every action, forever
```

**The PaymentIntent API (Stripe's master achievement):**

```
Old Charges API (pre-2019):
  POST /charges → tries to charge immediately
  Problem: 3D Secure, bank authentication, SCA compliance require
           asynchronous flows that don't fit request/response

PaymentIntents API (event-sourced workflow):
  POST /payment_intents → creates a PaymentIntent
    State: requires_payment_method

  Attach payment method →
    State: requires_confirmation

  Confirm →
    State: requires_action (3D Secure needed)
    → Redirect user to bank for authentication

  User authenticates at bank →
    State: processing

  Bank processes charge →
    State: succeeded OR payment_failed

Each state transition = one event appended to the event log
The PaymentIntent is reconstituted by replaying its events
```

---

## The API Design: 10 Years, Never Breaking Changes

**Stripe's API versioning philosophy:**

```
API launched 2011.
API still supported 2024.
Original API calls from 2011 still work perfectly.

How:
  API version pinned per account: "2014-01-01"
  When you make a call: Stripe runs the 2014 version of the response schema
  Even though internal data structures have changed

The version is a transformation layer (adapter):
  Internal data model evolves freely
  Version adapters transform internal model to the version the client expects

This is Clean Architecture's Interface Adapter pattern at the API level:
  Internal domain objects (Charge, PaymentIntent) = Entities
  Version-specific API response format = DTO shaped by the Adapter
  Client pinned to a version = never forced to upgrade
```

**Why this matters:**

> "A developer in 2011 built their entire checkout on Stripe's original API. They've been running that code for 13 years. They've never needed to update it. Their checkout still works. Stripe's internal systems have changed 10 times. The developer never noticed. This is the Clean Architecture's interface promise kept at planetary scale."

---

## The Reconciliation Job: When Events Lie

**The problem:**

Even with idempotency and event sourcing, distributed systems have edge cases:

```
Scenario: Network partition during charge
  1. Stripe sends charge to Visa
  2. Network partition occurs
  3. Visa processes the charge (charges the card)
  4. Stripe's network connection times out
  5. Stripe doesn't receive confirmation
  6. Stripe marks charge as "unknown"
  7. The user's card IS charged, but Stripe thinks it failed

Result: Customer is charged, Stripe thinks the payment failed
        Merchant ships nothing (order not confirmed)
        Customer has no delivery and no refund
        Money stuck in limbo
```

**The reconciliation job:**

```
Every hour:
  For each charge in state "unknown":
    Query Visa/Mastercard directly via batch API
    If Visa says "yes, we charged it" → update to "succeeded"
    If Visa says "no charge" → update to "failed" → unblock refund

Every day:
  Compare Stripe's ledger with bank's settlement files (CSV dumps)
  If any discrepancy → alert and investigate

Every month:
  Full reconciliation with all payment networks
  Any unclaimed money → held in reserve → returned via regulatory process
```

This is "defensive architecture" — assume events will be lost and build a correction mechanism.

---

## Architecture in Clean Architecture Terms

```
Stripe's Architecture:

PaymentIntent = Entity (aggregate root)
  Contains the business rule: "a payment can only succeed if authorized"
  Reconstituted from event log: PaymentIntentReconstituter.fromEvents([...])

ChargeCardUseCase = Use Case
  Depends on: IPaymentGateway, IEventStore, IIdempotencyStore (all interfaces)
  Publishes domain events: ChargeAttempted, ChargeSucceeded, ChargeFailed

IEventStore → StripeEventStorePostgres = Interface Adapter
  Stores events in PostgreSQL
  Use case never imports PostgreSQL directly

API versioning = Interface Adapter (inbound)
  HTTP request (v1 format) → adapter → internal DTO → use case
  Use case response → adapter → HTTP response (v1 format)
  Different adapters per API version; same use case

Reconciliation Jobs = Use Cases triggered by scheduler
  ReconcileUnknownChargesUseCase.execute()
  Depends on: IPaymentNetworkClient (Visa/Mastercard interface)
  Use case doesn't know if it's querying Visa or Mastercard — just the interface
```

---

## Lessons for Your Architecture

1. **Idempotency is not optional for money** — every payment operation must be idempotent; no exceptions
2. **Event Sourcing is the natural fit for financial systems** — regulators need history; events give you that for free
3. **Design APIs to last decades** — Stripe's version pinning means customers never need to upgrade
4. **Reconciliation catches what events miss** — distributed systems lie; reconcile against ground truth regularly
5. **The PaymentIntent pattern** — when an operation has async steps (3D Secure, bank auth), model it as a state machine driven by events

---

## Sources
- [Stripe's Payments APIs: The First 10 Years — Stripe Dev Blog](https://stripe.dev/blog/payment-api-design)
- [The First 10-Year Evolution of Stripe's Payments API — ByteByteGo](https://blog.bytebytego.com/p/the-first-10-year-evolution-of-stripes)
- [Engineering — Stripe Blog](https://stripe.com/blog/engineering)
- [Stripe System Design Interview Guide](https://www.systemdesignhandbook.com/guides/stripe-system-design-interview/)


---

## Architecture Diagram

![diagram.svg](diagram.svg)

