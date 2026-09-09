# Loyalty Points Ledger — Expiring, Transferable Points Done Right

> "A points balance is not a number. It's a set of promises with expiry dates, and every promise has to be honored in the order it was made." — Loyalty platform engineer

---

## At a Glance

| | |
|---|---|
| **What** | Points earned from spend, each batch carrying its own expiry date and transferable between users |
| **Why a single balance field fails** | Can't tell which points expire next, can't guarantee oldest-first spend, can't reverse a transfer safely |
| **Key property** | Balance is always derived: `SUM(remaining)` over unexpired lots — never a stored counter |
| **Used by** | Every points/rewards program with an expiry policy — airlines, retail, banking cashback, ride-hailing |

---

## Senior Explains to Junior: The Core Concept

> "New engineers reach for `users.points_balance INTEGER` and two functions: `addPoints()` and `deductPoints()`. That works right up until product asks for 'points expire after 12 months' and 'let users gift points to a friend.'
>
> Now you need to know: *which* points are about to expire? If Alice has 500 points and spends 200, which 200 disappear — the ones from March or the ones from August? If she gifts 100 points to Bob, does his gift expire on Bob's clock or the day Alice's original batch would have expired?
>
> The fix is the same one financial ledgers use for money: stop storing a balance. Store every earn event as its own row — a **lot** — with its own expiry, and record every change as an entry in an immutable transaction log. The balance is just a query."

```
Single-balance way (what most beginners code):
  user.points += 100        // earned from a purchase
  user.points -= 30         // redeemed at checkout
  Problem: no idea which 100 expires when, no order to spend in,
  no way to reverse a redemption to the correct source.

Lot-based ledger way:
  INSERT point_lots (user_id, amount, remaining, earned_at, expires_at)
  INSERT point_transactions (lot_id, type, delta, ref_id)
  Balance = SUM(remaining) WHERE expires_at > now()
  Spend always drains the lot with the soonest expires_at first (FIFO).
```

---

## The Database Schema

### Core Tables

```sql
-- Every earn event is its own row, with its own clock
CREATE TABLE point_lots (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id),
  amount      INTEGER NOT NULL CHECK (amount > 0),      -- points originally granted
  remaining   INTEGER NOT NULL CHECK (remaining >= 0),  -- points left to spend
  source      VARCHAR(20) NOT NULL
              CHECK (source IN ('purchase','promo','transfer','manual_adjust')),
  earned_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at  TIMESTAMPTZ NOT NULL,
  status      VARCHAR(10) NOT NULL DEFAULT 'active'
              CHECK (status IN ('active','expired','closed')),
  CHECK (remaining <= amount)
);

CREATE INDEX idx_point_lots_spend_order
  ON point_lots (user_id, expires_at)
  WHERE status = 'active' AND remaining > 0;

-- The immutable log: every earn, redeem, expiry and transfer
CREATE TABLE point_transactions (
  id            BIGSERIAL PRIMARY KEY,
  lot_id        UUID NOT NULL REFERENCES point_lots(id),
  type          VARCHAR(20) NOT NULL
                CHECK (type IN ('earn','redeem','expire','transfer_out','transfer_in','reverse')),
  delta         INTEGER NOT NULL,        -- signed: +earn/+transfer_in, -redeem/-transfer_out
  ref_id        UUID,                    -- links a transfer_out to its matching transfer_in,
                                          -- or an order_id for a redeem
  idempotency_key VARCHAR(128) UNIQUE,   -- one checkout retry, one transaction
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### The Balance View (computed, never stored)

```sql
CREATE VIEW user_point_balances AS
SELECT
  user_id,
  SUM(remaining) AS balance,
  MIN(expires_at) FILTER (WHERE remaining > 0) AS next_expiry
FROM point_lots
WHERE status = 'active' AND expires_at > NOW()
GROUP BY user_id;

-- WHY compute, not store?
-- A stored balance can drift from the ledger silently.
-- A computed balance can never be wrong — it IS the ledger, summarized.
```

---

## The Redeem Function — FIFO by Expiry, Not by Amount

```typescript
// Spends `amount` points from the oldest-expiring lots first.
// Must be atomic: partial consumption across lots or none at all.
async function redeemPoints(params: {
  userId: string;
  amount: number;
  orderId: string;
  idempotencyKey: string;
}): Promise<void> {
  const already = await db.query(
    `SELECT 1 FROM point_transactions WHERE idempotency_key = $1`,
    [params.idempotencyKey]
  );
  if (already.rows[0]) return; // already processed this checkout

  await db.transaction(async (trx) => {
    const lots = await trx.query(
      `SELECT id, remaining FROM point_lots
       WHERE user_id = $1 AND status = 'active' AND remaining > 0
       ORDER BY expires_at ASC
       FOR UPDATE`,
      [params.userId]
    );

    let toSpend = params.amount;
    for (const lot of lots.rows) {
      if (toSpend <= 0) break;
      const take = Math.min(lot.remaining, toSpend);

      await trx.query(
        `UPDATE point_lots SET remaining = remaining - $1 WHERE id = $2`,
        [take, lot.id]
      );
      await trx.query(
        `INSERT INTO point_transactions (lot_id, type, delta, ref_id, idempotency_key)
         VALUES ($1, 'redeem', $2, $3, $4)`,
        [lot.id, -take, params.orderId, params.idempotencyKey]
      );
      toSpend -= take;
    }

    if (toSpend > 0) throw new InsufficientPointsError(params.amount, toSpend);
  });
}
```

---

## Transfer — Moves Points, Never Resets Their Clock

The one rule that matters here: a transferred lot **inherits** the sender's original
`expires_at`. If transfers reset the clock, customers launder points that are about to
expire into a fresh 12-month lot by sending them to a second account and back.

```sql
-- Sender side: consume the sender's lot(s) FIFO, same as a redeem
UPDATE point_lots SET remaining = remaining - 20 WHERE id = 'lot_a3';
INSERT INTO point_transactions (lot_id, type, delta, ref_id)
  VALUES ('lot_a3', 'transfer_out', -20, 'transfer_789');

-- Receiver side: a NEW lot, but expires_at copied from the source lot
INSERT INTO point_lots (user_id, amount, remaining, source, earned_at, expires_at)
  SELECT 'user_bob', 20, 20, 'transfer', NOW(), expires_at
  FROM point_lots WHERE id = 'lot_a3';

INSERT INTO point_transactions (lot_id, type, delta, ref_id)
  VALUES (currval('point_lots_id_seq'), 'transfer_in', 20, 'transfer_789');
-- ref_id ties both legs of the transfer together for audit/reversal
```

---

## Expiry — Lazy Read, Eager Notify

```sql
-- Lazy: correctness needs no cron at all — a lot past its date is worth 0 on read
SELECT SUM(remaining) FROM point_lots
WHERE user_id = $1 AND status = 'active' AND expires_at > NOW();

-- Eager (nightly job): close expired lots and write the formal ledger entry,
-- so reporting and "points expiring soon" notifications have something to read
UPDATE point_lots SET status = 'expired'
WHERE status = 'active' AND expires_at <= NOW() AND remaining > 0
RETURNING id, remaining;
-- for each returned row: INSERT INTO point_transactions (lot_id, type, delta)
--   VALUES (id, 'expire', -remaining)
```

---

## Where Segment and Period Actually Live

A common mistake is to add `segment` or `period` columns to the points tables.
Neither belongs there:

- **Segment** is a property of the *user* (tier, cohort) — it decides the earn rate or
  the lot's expiry length *at the moment a lot is created*, e.g. `tier = 'gold'` → lots
  expire in 24 months instead of 12. It is never stored on the lot itself.
- **Period** is not a stored concept at all — it's a `WHERE earned_at BETWEEN ...` filter
  over the same ledger, used for reporting ("points earned in Q3").

---

## Two Failure Modes That Actually Cost Money

**Redundant issuance (double-crediting).** The `idempotency_key` on `point_transactions`
is the entire fix — a checkout retry, a webhook fired twice, or a duplicate queue job all
resolve to the same key, so the second write is a no-op. Points must be issued from the
confirmed server-side payment/order event exactly once, never from a client-supplied signal.

**Point inflation (currency devaluation).** An unredeemed point is a liability, not a UI
number — track `SUM(remaining)` the way a bank tracks deposits owed. If the earn rate
grows faster than redemption + expiry, that liability balloons the same way printing more
of a currency does. Two guardrails:
- Cap the earn rate per order server-side (never trust a client-supplied point amount).
- Expiry is the deliberate inflation control — lots that go unclaimed roll off the
  liability by design, which is why almost no points program ships with no expiry at all.

A simple health metric: `outstanding_liability / monthly_redemption_volume`. A rising
ratio means points are being issued faster than the business can afford to honor them.

---

## Real Companies Using This Pattern

### Starbucks Rewards
- Stars (points) earned per purchase expire on a rolling 6- or 12-month window depending on tier
- Each earn batch tracked separately so the app can show "X stars expiring this month"

### Grab Rewards
- Points from rides/food orders carry individual expiry dates
- Points can be redeemed across a marketplace of partner merchants — spend order matters for partner settlement

### Airline Miles (all major carriers)
- The oldest textbook case of lot-based expiry — miles from each earning trip historically expired independently
- Modern programs (many airlines now "no-expiry-while-active") still track lot provenance for status-qualifying vs. redeemable distinctions

---

## In Clean Architecture Terms

```
Point Lot Ledger = Domain Layer

PointLot = Entity (aggregate root)
  id, user_id, amount, remaining, earned_at, expires_at, status

PointTransaction = Entity (immutable once created)
  lot_id, type, delta, ref_id, created_at

RedeemPoints = Use Case
  execute(userId, amount, orderId, idempotencyKey): void
  Enforces FIFO-by-expiry — this rule lives here, not in the repository

TransferPoints = Use Case
  execute(fromUserId, toUserId, amount, idempotencyKey): void
  Enforces expiry inheritance — this rule lives here too

IPointLedgerRepository = Port
  lockActiveLots(userId): PointLot[]
  recordTransaction(lotId, type, delta, refId): void

PointLedgerRepositoryPostgres = Adapter
  All SQL and row locking logic here

ExpirePointsJob = Use Case (scheduled, nightly)
  closeExpiredLots(): void

The domain rule never moves to the adapter:
  Spend order = expires_at ascending. Transfer = expiry inherited, not reset.
```
