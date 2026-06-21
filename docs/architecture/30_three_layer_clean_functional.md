# 3-Layer Model + Clean Code + Functional Programming

> "Clean Architecture tells you WHERE code lives. Clean Code tells you HOW code is written. Functional Programming tells you WHAT style of code is best for each layer. Together they create code that is correct, readable, testable, and maintainable." — Senior architect

---

## At a Glance

| | |
|---|---|
| **3-Layer Model** | Presentation → Business Logic → Data Access |
| **Clean Architecture** | Entities → Use Cases → Adapters → Frameworks |
| **Clean Code** | Principles for writing readable, maintainable code |
| **Functional Programming** | Pure functions, immutability, function composition |
| **How they relate** | FP style applied to Clean Architecture layers = maximum quality |

---

## The Three Layers — Simplified Mental Model

Before Clean Architecture (4 rings), every developer learns 3 layers:

```
┌─────────────────────────────────┐
│  Presentation Layer             │  ← "What the user sees"
│  (UI, Controllers, API routes)  │
├─────────────────────────────────┤
│  Business Logic Layer           │  ← "What the app does"
│  (Services, Use Cases, Rules)   │
├─────────────────────────────────┤
│  Data Access Layer              │  ← "Where data lives"
│  (Repositories, ORM, Queries)   │
└─────────────────────────────────┘
```

**The rule:** Each layer only talks to the layer directly below it.
- Presentation calls Business Logic
- Business Logic calls Data Access
- Data Access calls the Database

**Clean Architecture adds precision:**
- Presentation Layer → Interface Adapters + Frameworks & Drivers
- Business Logic → Use Cases + Entities
- Data Access → Interface Adapters (Repositories) + Frameworks & Drivers (DB driver)

**The key improvement:** Dependency Inversion — Business Logic defines interfaces, Data Access implements them.

---

## Senior Explains the 3-Layer to a Junior

> "Think of a restaurant:
> - **Presentation Layer** = the waiter. Takes your order, brings your food. Knows how to talk to customers. Doesn't cook.
> - **Business Logic Layer** = the chef. Knows HOW to make the food. The recipe is the business rule. The chef doesn't care if you're a customer or just walked into the kitchen.
> - **Data Access Layer** = the pantry/storage. Stores and retrieves ingredients. Doesn't care what dish is being cooked.
>
> The problem with bad code: the waiter sometimes cooks (business logic in controllers), and the chef sometimes talks to customers (database queries in service methods that also return HTTP responses). Clean Code means every layer does exactly its job — nothing more."

---

## The 3 Layers in Code (TypeScript)

### Presentation Layer (Controller)

```typescript
// ✅ CLEAN: Controller only translates HTTP ↔ Use Case
// No business logic. No database access.
@Controller('/orders')
class OrderController {
  constructor(private placeOrder: PlaceOrderUseCase) {}

  @Post('/')
  async create(@Body() dto: PlaceOrderDto, @Request() req: Request) {
    // 1. Parse HTTP request into use case input
    const request: PlaceOrderRequest = {
      customerId: req.user.id,
      items: dto.items.map(item => ({
        productId: item.productId,
        quantity: item.quantity,
      })),
    };

    // 2. Call use case (no business logic here!)
    const result = await this.placeOrder.execute(request);

    // 3. Format use case output as HTTP response
    return {
      orderId: result.orderId,
      total: result.total.format(),
      status: 'created',
    };
  }
}

// ❌ BAD: Business logic in controller
@Post('/')
async create(@Body() dto: any) {
  if (dto.items.length === 0) throw new Error('Empty order'); // ← business rule in controller!
  const order = await this.db.query('INSERT INTO orders...'); // ← DB access in controller!
  await this.stripe.charges.create({ amount: total }); // ← payment logic in controller!
  return order;
}
```

### Business Logic Layer (Use Case)

```typescript
// ✅ CLEAN: Use Case contains ONLY business rules
// No HTTP. No SQL. Depends on interfaces, not implementations.
class PlaceOrderUseCase {
  constructor(
    private readonly orders: IOrderRepository,    // interface ← not Postgres
    private readonly inventory: IInventoryService, // interface ← not SQL
    private readonly payments: IPaymentGateway,   // interface ← not Stripe
    private readonly events: IEventBus,           // interface ← not Kafka
  ) {}

  async execute(request: PlaceOrderRequest): Promise<PlaceOrderResponse> {
    // Business rule 1: validate order
    if (request.items.length === 0) {
      throw new DomainError('ORDER_EMPTY', 'Cannot place an empty order');
    }

    // Business rule 2: check inventory
    for (const item of request.items) {
      const available = await this.inventory.getAvailable(item.productId);
      if (available < item.quantity) {
        throw new DomainError('INSUFFICIENT_STOCK', `Product ${item.productId} has only ${available} units`);
      }
    }

    // Business rule 3: create order entity (domain logic in entity)
    const order = Order.create(request.customerId, request.items);

    // Business rule 4: process payment
    const payment = await this.payments.charge(order.total(), request.paymentMethod);

    // Persist
    await this.orders.save(order);
    await this.events.publish(new OrderPlacedEvent(order));

    return { orderId: order.id, total: order.total() };
  }
}
```

### Data Access Layer (Repository)

```typescript
// ✅ CLEAN: Repository implements the interface, handles all DB concerns
class OrderRepositoryPostgres implements IOrderRepository {
  constructor(private db: DatabasePool) {}

  async save(order: Order): Promise<void> {
    await this.db.transaction(async (trx) => {
      await trx.query(`
        INSERT INTO orders (id, customer_id, status, created_at)
        VALUES ($1, $2, $3, $4)
        ON CONFLICT (id) DO UPDATE SET status = $3
      `, [order.id, order.customerId, order.status, order.createdAt]);

      for (const item of order.items) {
        await trx.query(`
          INSERT INTO order_items (order_id, product_id, quantity, unit_price_cents)
          VALUES ($1, $2, $3, $4)
        `, [order.id, item.productId, item.quantity, item.unitPrice.toCents()]);
      }
    });
  }

  async findById(id: string): Promise<Order | null> {
    const rows = await this.db.query(`
      SELECT o.*, oi.product_id, oi.quantity, oi.unit_price_cents
      FROM orders o
      LEFT JOIN order_items oi ON oi.order_id = o.id
      WHERE o.id = $1
    `, [id]);

    if (rows.length === 0) return null;
    return OrderMapper.toDomain(rows);  // DB rows → Domain entity
  }
}
```

---

## Clean Code Principles Applied to Each Layer

### 1. Meaningful Names (No Abbreviations)

```typescript
// ❌ BAD: cryptic abbreviations
const usrSvc = new UserSvc();
const ord = await ordRepo.fndById(req.params.id);
const amt = calcTotal(ord.itms);

// ✅ GOOD: reads like English
const userService = new UserService();
const order = await orderRepository.findById(request.params.orderId);
const totalAmount = calculateOrderTotal(order.items);

// ✅ GOOD: boolean names should sound like questions
const isActive = user.active;
const hasPermission = user.canEdit(resource);
const isEmpty = cart.items.length === 0;
```

### 2. Single Responsibility at Every Level

```typescript
// ❌ BAD: function does multiple things
async function processOrder(orderId: string, userId: string) {
  const order = await db.query('SELECT * FROM orders WHERE id = $1', [orderId]);
  if (order.user_id !== userId) throw new Error('Forbidden');  // auth check
  if (order.items.length === 0) throw new Error('Empty');      // validation
  const total = order.items.reduce((s, i) => s + i.price, 0); // calculation
  await stripe.charge(total);                                   // payment
  await db.query('UPDATE orders SET status = paid WHERE id = $1', [orderId]); // persistence
  await sendEmail(userId, `Your order ${orderId} is confirmed`);  // notification
  // This function has 5 responsibilities = 5 reasons to change
}

// ✅ GOOD: each function does ONE thing
class PlaceOrderUseCase {
  async execute(req: PlaceOrderRequest) {
    this.authorizeUser(req);           // delegates to auth
    const order = this.buildOrder(req); // delegates to entity
    await this.chargePayment(order);   // delegates to payment gateway
    await this.persist(order);         // delegates to repository
    await this.notify(order);          // delegates to notifier
  }
}
```

### 3. Small Functions (Do One Thing Well)

```typescript
// ❌ BAD: 80-line function with nested conditionals
function validateAndProcessPayment(order, user, card, currency) {
  if (order !== null) {
    if (order.items.length > 0) {
      if (user.isActive) {
        if (card.isValid()) {
          if (currency === 'USD' || currency === 'THB') {
            // 50 more lines...
          }
        }
      }
    }
  }
}

// ✅ GOOD: small, flat, guard-clause style
function processPayment(order: Order, user: User, card: PaymentCard): void {
  assertOrderNotEmpty(order);
  assertUserActive(user);
  assertCardValid(card);
  assertCurrencySupported(order.currency);

  this.paymentGateway.charge(order.total(), card);
}

function assertOrderNotEmpty(order: Order): void {
  if (order.items.length === 0) throw new DomainError('ORDER_EMPTY');
}
// Each assertion is one function, one test, one reason to change
```

### 4. No Magic Numbers or Strings

```typescript
// ❌ BAD: magic numbers
if (user.trialDays > 14) { /* ... */ }
await redis.setex(key, 3600, value);
if (order.items.length > 100) { /* ... */ }

// ✅ GOOD: named constants
const TRIAL_PERIOD_DAYS = 14;
const PRODUCT_CACHE_TTL_SECONDS = 60 * 60;  // 1 hour
const MAX_ITEMS_PER_ORDER = 100;

if (user.trialDays > TRIAL_PERIOD_DAYS) { /* ... */ }
await redis.setex(key, PRODUCT_CACHE_TTL_SECONDS, value);
if (order.items.length > MAX_ITEMS_PER_ORDER) { /* ... */ }
```

### 5. Error Handling: Be Explicit

```typescript
// ❌ BAD: generic errors that lose context
throw new Error('Something went wrong');
return null;  // instead of throwing

// ✅ GOOD: domain-specific errors with context
class DomainError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly context?: Record<string, unknown>
  ) {
    super(message);
    this.name = 'DomainError';
  }
}

// In use case:
if (available < requested) {
  throw new DomainError(
    'INSUFFICIENT_STOCK',
    `Only ${available} units of ${productId} available, ${requested} requested`,
    { productId, available, requested }
  );
}
// The controller catches DomainError → maps to 422 Unprocessable Entity
// The use case never knows what HTTP status code to use
```

---

## Functional Programming in Clean Architecture

### Why FP Fits Clean Architecture

**Entities should be pure functions:**
```typescript
// ✅ FP: Entity methods are pure — same input → same output, no side effects
class Order {
  // Pure: doesn't modify state, returns new value
  calculateTotal(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.subtotal()),
      Money.ZERO
    );
  }

  // Pure: business rule as a pure function
  canBeCancelled(): boolean {
    return this.status === OrderStatus.PENDING || this.status === OrderStatus.PLACED;
  }

  // Mutation returns NEW instance (immutability)
  withStatus(newStatus: OrderStatus): Order {
    return new Order({ ...this, status: newStatus });
  }
}
```

### Immutability in Practice

```typescript
// ❌ Mutable (hard to reason about, causes bugs)
class Cart {
  items: CartItem[] = [];

  addItem(item: CartItem) {
    this.items.push(item);   // mutates in place — what was items before?
    this.recalculateTotal(); // side effect
  }
}

// ✅ Immutable (easy to reason about, thread-safe, testable)
class Cart {
  constructor(
    private readonly items: ReadonlyArray<CartItem>,
    private readonly total: Money,
  ) {}

  addItem(item: CartItem): Cart {
    // Returns NEW Cart — original unchanged
    const newItems = [...this.items, item];
    const newTotal = this.total.add(item.subtotal());
    return new Cart(newItems, newTotal);
  }
}

// Usage:
const emptyCart = new Cart([], Money.ZERO);
const cartWithApple = emptyCart.addItem(apple);   // emptyCart unchanged!
const cartWithBoth = cartWithApple.addItem(book); // cartWithApple unchanged!
```

### Pure Functions for Business Rules

```typescript
// All business rules as pure functions — easy to test, impossible to have side effects
// Lives in Domain / Entity layer

// Pure function: same inputs → same output, no external state
const calculateOrderTotal = (items: OrderItem[]): Money =>
  items.reduce((total, item) => total.add(item.subtotal()), Money.ZERO);

const applyDiscount = (total: Money, discount: Discount): Money =>
  total.multiply(1 - discount.percentage / 100);

const isEligibleForFreeShipping = (total: Money, threshold: Money): boolean =>
  total.greaterThanOrEqual(threshold);

// Test: pure functions are trivially testable
describe('calculateOrderTotal', () => {
  it('returns zero for empty order', () => {
    expect(calculateOrderTotal([])).toEqual(Money.ZERO);
  });

  it('sums all items', () => {
    const items = [
      makeItem({ price: 100, qty: 2 }),  // 200
      makeItem({ price: 50, qty: 1 }),   // 50
    ];
    expect(calculateOrderTotal(items)).toEqual(Money.of(250));
  });
});
// No database needed. No HTTP server. Runs in <1ms.
```

### Function Composition — Building Pipelines

```typescript
// Compose small pure functions into larger workflows
// Each step transforms the data

const pipe = <T>(...fns: Array<(x: T) => T>) =>
  (x: T): T => fns.reduce((acc, fn) => fn(acc), x);

// Each transformation is a pure function
const normalizeEmail = (user: User): User => ({
  ...user,
  email: user.email.toLowerCase().trim(),
});

const setDefaultRole = (user: User): User => ({
  ...user,
  role: user.role ?? UserRole.VIEWER,
});

const enrichWithTimestamps = (user: User): User => ({
  ...user,
  createdAt: user.createdAt ?? new Date(),
  updatedAt: new Date(),
});

// Compose into a pipeline (reads left to right)
const prepareUserForSave = pipe(
  normalizeEmail,
  setDefaultRole,
  enrichWithTimestamps,
);

// Usage:
const savedUser = prepareUserForSave(rawUser);
// Clean, testable, each step independently verifiable
```

### Option/Result Types — Eliminating null Checks

```typescript
// ❌ BAD: null everywhere — causes NullPointerException
const user = await userRepo.findById(id); // might be null
const address = user.address; // ERROR if user is null!
const city = address.city;    // ERROR if address is null!

// ✅ GOOD: Option type — null is explicit
type Option<T> = { kind: 'some', value: T } | { kind: 'none' };

class UserRepository {
  async findById(id: string): Promise<Option<User>> {
    const row = await this.db.query('SELECT * FROM users WHERE id = $1', [id]);
    if (!row) return { kind: 'none' };
    return { kind: 'some', value: UserMapper.toDomain(row) };
  }
}

// In use case — must handle both cases explicitly:
const userOption = await userRepo.findById(id);
if (userOption.kind === 'none') {
  throw new DomainError('USER_NOT_FOUND', `User ${id} not found`);
}
const user = userOption.value; // TypeScript knows this is User, not User | null
```

### Result Type — Making Errors Explicit

```typescript
// ❌ BAD: exceptions for expected failures
async function chargeCard(amount: Money): Promise<void> {
  const result = await stripe.charge(amount);
  if (result.status === 'declined') throw new Error('Card declined'); // exception for control flow
}

// ✅ GOOD: Result type — success or failure is explicit in the type
type Result<T, E> =
  | { success: true; value: T }
  | { success: false; error: E };

async function chargeCard(amount: Money): Promise<Result<Payment, PaymentError>> {
  const result = await stripe.charge(amount);
  if (result.status === 'declined') {
    return { success: false, error: { code: 'CARD_DECLINED', message: result.message } };
  }
  return { success: true, value: Payment.from(result) };
}

// Caller must handle both cases:
const chargeResult = await this.payments.charge(order.total());
if (!chargeResult.success) {
  if (chargeResult.error.code === 'CARD_DECLINED') {
    throw new DomainError('PAYMENT_DECLINED', chargeResult.error.message);
  }
}
const payment = chargeResult.value;
```

---

## Putting It All Together — A Complete Feature

```
User Story: "As a customer, I want to place an order with items in my cart."

Layer            Component                FP Style
──────────────── ──────────────────────── ────────────────────────────────────
Presentation     OrderController          Thin adapter (translates HTTP ↔ DTO)
                 PlaceOrderDto            Immutable data transfer object
                 ↓
Use Case         PlaceOrderUseCase        Orchestrates domain + ports
                                          No side effects in business logic
                 ↓
Domain/Entity    Order.create()           Pure function (returns new Order)
                 Order.calculateTotal()   Pure function (same input → same output)
                 Money.add()              Immutable value object
                 ↓
Interface        IOrderRepository         Port (interface) defined here
Adapters         IPaymentGateway          Port (interface) defined here
                 OrderMapper.toDomain()   Pure function (DB row → Domain entity)
                 ↓
Infrastructure   OrderRepositoryPostgres  Implements IOrderRepository
                 StripePaymentGateway     Implements IPaymentGateway
                 KafkaEventBus            Implements IEventBus
```

---

## Testing Strategy per Layer (with FP)

| Layer | Test type | Speed | What's tested |
|---|---|---|---|
| **Pure functions** (entity) | Unit test | ⚡ <1ms | Business rules (pure functions, no mocks needed) |
| **Use Cases** | Unit test + mocks | ⚡ <10ms | Orchestration (mock all ports) |
| **Adapters** | Integration test | 🐢 ~200ms | DB queries, HTTP calls |
| **E2E** | System test | 🐌 ~2s | Full user journey |

```typescript
// Pure entity test — zero infrastructure, zero mocks
describe('Order', () => {
  it('calculates total correctly', () => {
    const order = Order.create('user-1', [
      OrderItem.of('product-1', 2, Money.of(100)),  // 200
      OrderItem.of('product-2', 1, Money.of(50)),   // 50
    ]);
    expect(order.total()).toEqual(Money.of(250));
  });

  it('cannot be placed when empty', () => {
    const emptyOrder = Order.create('user-1', []);
    expect(() => emptyOrder.place()).toThrow('ORDER_EMPTY');
  });
});

// Use case test — mock all ports, test orchestration
describe('PlaceOrderUseCase', () => {
  it('charges payment and saves order', async () => {
    const mockOrders = { save: jest.fn(), findById: jest.fn() };
    const mockPayments = { charge: jest.fn().mockResolvedValue({ id: 'pay-1' }) };
    const useCase = new PlaceOrderUseCase(mockOrders, mockPayments, mockEvents);

    await useCase.execute({ customerId: 'u-1', items: [{ productId: 'p-1', qty: 1 }] });

    expect(mockPayments.charge).toHaveBeenCalledOnce();
    expect(mockOrders.save).toHaveBeenCalledOnce();
  });
});
```

---

## The Senior's Final Advice

> "3 layers, Clean Architecture, Clean Code, and Functional Programming all point toward the same truth: **separate concerns, make behavior explicit, eliminate hidden state.**
>
> A junior developer writes code that works. A mid-level developer writes code that works and is organized. A senior developer writes code that works, is organized, is testable without infrastructure, communicates its intent clearly, and makes wrong states unrepresentable.
>
> Functional Programming is the style that achieves this: pure functions are testable by definition (no hidden state), immutability prevents a whole class of bugs (you can't accidentally modify something), and function composition lets you build complex behavior from simple, verified building blocks.
>
> Applied to Clean Architecture: pure functions in the Entity layer, functional composition in Use Cases, and side effects pushed to the edges (adapters). This is not a theoretical ideal — it's what Netflix, Stripe, and Airbnb actually write."
