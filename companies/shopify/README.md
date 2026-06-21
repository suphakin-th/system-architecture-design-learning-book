# Shopify — Architecture Case Study

> "We chose to stay a monolith and optimize it. That was the hardest, most contrarian decision we made." — Shopify Engineering

---

## Company Profile

| | |
|---|---|
| **Founded** | 2006 |
| **Scale** | 2M+ merchants, $200B+ GMV/year, 700M+ buyers |
| **Code** | 2.8M+ lines of Ruby, 500K+ commits, 1000+ developers |
| **Peak** | 30TB of data per minute on Black Friday |
| **Architecture today** | Modular Monolith + MySQL Pods + Packwerk |

---

## The Senior Architect Explains the Key Decision

> "In 2019, when every other company at our scale was racing to microservices, Shopify doubled down on the monolith. Not the spaghetti monolith that breaks everything — the modular monolith with strict boundaries enforced by tooling. This decision is still controversial. Let me explain why it was right."

---

## The Problem: The Tangled Monolith (2015–2018)

**What "Shopify's monolith" looked like before they fixed it:**

```
shopify/
├── app/
│   ├── models/
│   │   ├── order.rb          # business logic AND DB model
│   │   ├── product.rb        # calls shipping.rb directly
│   │   ├── cart.rb           # calls payments.rb directly
│   │   └── checkout.rb       # calls everything
│   ├── controllers/
│   │   └── (fat controllers calling models directly)
│   └── services/
│       └── (no consistent pattern — some have, some don't)
```

**The specific problems:**

1. **Uncontrolled coupling:** The checkout code called shipping, inventory, payment, and tax modules directly. Changing tax calculation broke checkout. The tax team had to coordinate with the checkout team for every change.

2. **Cascading test failures:** A change to `product.rb` would fail 500 tests in `checkout_spec.rb` because Product was imported (even indirectly) by Checkout. "Why did my product change break checkout tests?!"

3. **Slow tests:** 2.8M lines of code means running the full test suite took hours. Developers waited 40+ minutes to know if their change was good.

4. **Black Friday fragility:** During Black Friday, code that calculated shipping rates lived alongside code handling checkouts. A slow shipping rate calculation slowed down checkout for all merchants. No isolation between concerns.

---

## The Solution: Packwerk and Modular Monolith (2019)

**Shopify's key insight:**

> "We don't need network boundaries to have clean boundaries. We need ENFORCED MODULE BOUNDARIES inside our monolith. Microservices enforce this with the network. We'll enforce it with static analysis."

**Packwerk — Shopify's open-source boundary enforcer:**

```ruby
# packages.yml (defines the modules)
# checkout module cannot import from shipping module directly
components/checkout:
  enforce_dependencies: true
  enforce_privacy: true

components/shipping:
  enforce_dependencies: true
  enforce_privacy: true

# If checkout/order.rb tries to import Shipping::Calculator:
#   Packwerk fails the build
#   Error: "checkout cannot depend on shipping — use the API"
```

**The module structure after Packwerk:**

```
shopify/
├── components/
│   ├── checkout/
│   │   ├── app/models/checkout/order.rb     # Internal: can't import shipping directly
│   │   ├── app/public/checkout/api.rb       # Public interface (port)
│   │   └── package.yml                      # Defines this as a bounded module
│   ├── shipping/
│   │   ├── app/models/shipping/calculator.rb
│   │   ├── app/public/shipping/api.rb       # Public interface
│   │   └── package.yml
│   ├── payments/
│   ├── inventory/
│   └── tax/
└── app/                                     # Legacy code (being migrated)
```

**How checkout calls shipping after Packwerk:**

```ruby
# WRONG (before Packwerk enforcement):
class Checkout::Order
  def calculate_shipping
    Shipping::Calculator.calculate(self.items)  # Direct import — BANNED
  end
end

# RIGHT (after Packwerk):
class Checkout::Order
  def calculate_shipping
    Checkout::ShippingPort.calculate(self.items)  # Call through the public API
  end
end

# Checkout::ShippingPort is an adapter that calls Shipping::API
# Shipping can change its internals without affecting Checkout
# This IS the Clean Architecture port/adapter pattern
```

---

## The Database Scaling Problem: MySQL Pods

**The one-DB-per-monolith problem:**

```
Shopify has 2M+ merchants. On a normal day, merchant A and merchant B
don't affect each other. But on Black Friday:

Merchant A (big store): 10,000 orders/minute
Merchant B (small store): 10 orders/minute

On a shared database:
  Merchant A's massive load → long-running queries → locks tables
  → Merchant B's queries wait for locks
  → Merchant B's checkout is slow
  → Merchant B's customers abandon carts
  → Merchant B loses sales they'd have made

  Merchant A's success is costing Merchant B.
```

**Shopify's Pod Architecture:**

```
Pod 1 (MySQL cluster): Merchants #1–500,000
  Primary MySQL + 3 read replicas
  All data for these 500K merchants lives ONLY here

Pod 2 (MySQL cluster): Merchants #500,001–1,000,000
  Primary MySQL + 3 read replicas

Pod 3: Merchants #1,000,001–1,500,000
...
Pod N: Merchants #(N-1)*500K – N*500K

Routing layer: every request → look up merchant → route to correct pod
```

**Why this is brilliant:**

```
Black Friday:
  Big merchant (1M orders/minute) → on Pod 3 → only Pod 3 is stressed
  Small merchant → on Pod 7 → completely unaffected by Pod 3's load

  Pod isolation = merchant isolation = every merchant gets consistent performance

  New merchant joins: assigned to least-loaded pod
  Pod filling up: migrate some merchants to a new empty pod
```

**The shard key: `shop_id`**

```ruby
# Every database query is scoped to the current shop:
Shop.with_current_shop(shop_id) do
  Order.all  # Automatically queries correct pod, correct shard
end

# The ORM (ActiveRecord) + Shopify's custom sharding layer
# routes queries to the correct MySQL pod transparently
```

---

## Black Friday Architecture

**The challenge:** Shopify processes 30TB of data per minute on Black Friday peak.

```
Normal day: baseline load
  Checkout: 10,000 orders/minute across all merchants
  Response time: <100ms

Black Friday peak:
  Checkout: 10,000,000 orders/minute
  Response time: must still be <100ms
  Traffic increase: 1000×

How Shopify handles it:
1. Pre-scaling (days before):
   Database pods pre-scaled to 3× normal capacity
   Redis cluster pre-scaled for session/cart caching
   CDN pre-warmed with merchant storefront assets
   Kubernetes auto-scaling configured with higher upper limits

2. Storefront caching (biggest lever):
   Product pages are cached in CDN and served without hitting app servers
   Only "buy" button flow hits Shopify backend
   99% of Black Friday traffic = reading product pages = CDN cached

3. Queue-based checkout (asynchrony):
   Cart → checkout attempt → enters queue → processed serially per shop
   Prevents N simultaneous checkouts from overwhelming one shop's pod
   User sees: "Your order is being processed" (immediately)
   vs: "503 Service Unavailable" (without queue)
```

---

## Why Shopify Didn't Choose Microservices

**The senior architect's honest assessment:**

> "We looked at what microservices would cost us. We had 1000 developers in 2019. To move to microservices we'd need:
> - Kubernetes expertise (takes 6-12 months to learn)
> - Service mesh (Istio adds 50% ops complexity)
> - Distributed tracing (had none at the time)
> - Per-service CI/CD pipelines (×100 services = ×100 pipelines)
> - Cross-service API versioning (extremely painful in practice)
>
> And what would we gain? Independent deployments, maybe. Independent scaling of specific components, possibly. But we can already scale horizontally — we just add more Shopify app servers. The bottleneck is the database, and we solved that with pods.
>
> The cost of microservices was: 2 years of migration, massive ops overhead, probable bugs from distributed transactions. The benefit: marginal. We chose modular monolith."

---

## Architecture in Clean Architecture Terms

```
Shopify's Modular Monolith = Clean Architecture within one deployment

Packwerk modules = Clean Architecture layers + bounded contexts
  component/checkout/app/public/api.rb = the Port (interface)
  component/checkout/app/models/ = Entities + Use Cases
  component/checkout/app/adapters/ = Interface Adapters

MySQL Pod routing = Infrastructure layer (outbound adapter)
  IOrderRepository → ShopifyActiveRecordRepository
  Repository selects correct pod connection based on current_shop_id
  Use cases never know which pod they're on

Shopify Admin (React + GraphQL) = Framework & Drivers (inbound adapter)
  React component calls GraphQL query
  GraphQL resolver = Controller that calls use case
  Use case response shapes the GraphQL type

Key insight: You CAN have Clean Architecture in a monolith.
The boundaries are enforced by tools (Packwerk) not by the network.
```

---

## Lessons for Your Architecture

1. **Modular monolith beats unstructured microservices** — Shopify serves $200B GMV from one deployable Ruby app
2. **Enforce boundaries with tools, not willpower** — Packwerk makes the architecture real; without it, developers take shortcuts
3. **Database isolation solves tenant isolation** — pods separate merchants more effectively than code separation
4. **Caching is the biggest scaling lever** — 99% of Black Friday traffic is CDN-served product pages
5. **The right architecture depends on your team, not just your scale** — 1000 engineers on Rails > 1000 engineers learning Kubernetes

---

## Sources
- [Under Deconstruction: The State of Shopify's Monolith](https://shopify.engineering/shopify-monolith)
- [Deconstructing the Monolith — Shopify Engineering](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity)
- [Inside Shopify's Modular Monolith — Dr. Milan Milanović](https://newsletter.techworld-with-milan.com/p/inside-shopifys-modular-monolith)
- [How Shopify Handles 30TB of Data Every Minute](https://medium.com/@himanshusingour7/how-shopify-handles-30tb-of-data-every-minute-with-a-monolithic-architecture-cad54df86955)
