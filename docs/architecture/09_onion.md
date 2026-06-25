# Architecture 09 - Onion Architecture

---

## At a Glance

| | |
|---|---|
| **Type** | Layered dependency inversion (same as Clean Arch / Hexagonal) |
| **Complexity** | Medium |
| **Best for** | Domain-heavy enterprise apps, DDD (Domain-Driven Design) |
| **Relationship** | Same concept as Clean Architecture and Hexagonal - DDD-flavored naming |

---

## What Is It?

Onion Architecture (Jeffrey Palermo, 2008) organizes code into concentric layers like an onion. The innermost layers are the most stable; the outermost layers are the most volatile.

**Key difference from Clean Architecture naming:**
- Clean Arch: Entities -> Use Cases -> Adapters -> Frameworks
- Onion: Domain Model -> Domain Services -> Application Services -> Infrastructure

The dependency rule is identical: **dependencies point inward only.**

---

## Diagram Reference
`./diagram.svg`

---

## The Four Onion Layers

### Layer 1 - Domain Model (Innermost)
Pure domain objects: entities, value objects, domain events, aggregates.
No dependencies on anything.

```typescript
// Pure domain, no imports from outer layers
class Money {
  constructor(public readonly amount: number, public readonly currency: string) {}
  add(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Currency mismatch');
    return new Money(this.amount + other.amount, this.currency);
  }
}

class Order {
  private _items: OrderItem[] = [];
  constructor(public readonly id: OrderId, private readonly customerId: CustomerId) {}
  addItem(item: OrderItem) { /* business rule */ }
  place() { /* business rule */ }
  total(): Money { /* calculation */ }
}
```

### Layer 2 - Domain Services
Business logic that doesn't naturally belong in one entity.
Depends only on Domain Model.

```typescript
class PricingService {
  applyVolumeDiscount(order: Order, catalogue: ProductCatalogue): Money {
    // Logic spanning Order and ProductCatalogue - belongs in a domain service
    const baseTotal = order.total();
    const discount = catalogue.getVolumeDiscount(order.itemCount());
    return baseTotal.multiply(1 - discount);
  }
}
```

### Layer 3 - Application Services
Orchestrates domain model and domain services to fulfil use cases.
Defines interfaces (ports) for infrastructure.
Corresponds to Use Cases in Clean Architecture.

```typescript
interface IOrderRepository { /* port defined HERE */ }
interface IPaymentGateway  { /* port defined HERE */ }

class PlaceOrderApplicationService {
  constructor(
    private orders: IOrderRepository,
    private payments: IPaymentGateway,
    private pricing: PricingService
  ) {}

  async placeOrder(cmd: PlaceOrderCommand) {
    const order = await this.orders.findById(cmd.orderId);
    const finalPrice = this.pricing.applyVolumeDiscount(order, cmd.catalogue);
    await this.payments.charge(finalPrice, cmd.paymentMethod);
    order.place();
    await this.orders.save(order);
  }
}
```

### Layer 4 - Infrastructure (Outermost)
Implements all interfaces (ports) from layer 3.
Contains: repositories, HTTP controllers, DB adapters, external API clients.

```typescript
class OrderRepositoryPostgres implements IOrderRepository { /* ... */ }
class StripePaymentGateway implements IPaymentGateway { /* ... */ }
class OrderController { /* HTTP adapter */ }
```

---

## Onion vs Clean Architecture vs Hexagonal

| Concept | Onion | Clean Architecture | Hexagonal |
|---|---|---|---|
| Core | Domain Model | Entities | Application Core |
| Business logic | Domain Services + App Services | Use Cases | Application Core |
| Interface definitions | Application Services layer | Use Cases layer | Ports |
| Implementations | Infrastructure | Interface Adapters + Frameworks | Adapters |
| Dependency rule | Points inward | Points inward | All external through ports |

**All three are the same architecture. Onion just has stronger DDD vocabulary.**

---

## Domain-Driven Design (DDD) Alignment

Onion Architecture is the natural home for DDD concepts:

| DDD Concept | Onion Layer |
|---|---|
| Entity | Domain Model |
| Value Object | Domain Model |
| Aggregate | Domain Model |
| Domain Event | Domain Model |
| Domain Service | Domain Services |
| Repository (interface) | Application Services |
| Application Service | Application Services |
| Repository (impl) | Infrastructure |
| Anti-Corruption Layer | Infrastructure |

---

## Resource Consumption

Same as Hexagonal / Clean Architecture - purely organizational:

| Resource | Impact |
|---|---|
| Runtime CPU/Memory | Zero overhead |
| Test speed | Dramatically faster (domain model tests: <1ms each) |
| Developer productivity | Higher long-term (clear location for every concept) |

---

## Benefits

1. **DDD-aligned** - onion layers map directly to DDD tactical patterns
2. **Maximum testability** - domain model layer testable with zero dependencies
3. **Business logic isolated** - framework changes never touch domain model
4. **Explicit domain services** - cross-entity logic has a clear home
5. **Rich domain model** - encourages putting behavior IN domain objects, not anemic models

---

## Problems It Solves Best

| Problem | How Onion Helps |
|---|---|
| "Our domain is complex (insurance, banking, healthcare)" | Rich domain model + domain services express it clearly |
| "Business rules are scattered across controllers and DB queries" | All business logic in Domain Model + Domain Services |
| "Domain experts can't understand our code" | DDD ubiquitous language maps to domain model class names |
| "We're building a long-lived enterprise system (10+ years)" | Stable inner layers survive framework/DB changes |

---

## Costs / Tradeoffs

1. **Overhead for simple apps** - CRUD apps don't benefit; onion adds structure without value
2. **DDD learning curve** - team must understand aggregates, value objects, bounded contexts
3. **More files** - domain model, domain services, application services, infrastructure all separate
4. **Anti-patterns can slip in** - "anemic domain model" (entities with no behavior) defeats the purpose

---

## Big Tech Examples

### Domain-Driven Design Practitioners (Finance / Insurance)

**ABN AMRO Bank (Netherlands)**
- **Architecture:** Onion Architecture + CQRS + Event Sourcing (Axon Framework)
- **Domain:** Banking - Account, Transaction, Loan, Customer aggregates
- **Good at:** Banking regulations change frequently; infrastructure (cloud provider, DB) can be swapped without touching domain logic

**ING Bank (Global)**
- **Architecture:** DDD + Onion layers per bounded context (Payments, Mortgages, Insurance)
- **Domain Services:** `InterestCalculationService`, `RiskAssessmentService`
- **Good at:** Complex domain logic (mortgage eligibility rules) lives in Domain Services, tested independently

**Axon Framework (Open Source)**
- **Tech:** Java framework built specifically for Onion + CQRS + Event Sourcing
- **Used by:** Banks, insurance companies, healthcare systems
- **Good at:** Provides the scaffolding so teams can focus on domain model

**Medical Records Systems (Epic, Cerner)**
- **Architecture:** Onion with rich domain model: `Patient`, `Encounter`, `Prescription` aggregates
- **Domain Services:** `DrugInteractionService`, `LabResultInterpretationService`
- **Good at:** Healthcare business rules (drug dosing limits) are complex and must be isolated from EHR platform details

---

## Key Takeaway

> Onion Architecture is Clean Architecture with DDD vocabulary. If your domain is complex - banking, insurance, healthcare, e-commerce with complex pricing - use Onion Architecture to give every DDD concept a clear, stable home. The "richer" the domain, the more value the onion layers provide.
