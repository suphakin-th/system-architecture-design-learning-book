# Architecture 13 — Serverless Architecture

---

## At a Glance

| | |
|---|---|
| **Type** | Function-as-a-Service (FaaS) — no server management |
| **Complexity** | Low-Medium (ops is low; async patterns can be complex) |
| **Best for** | Event-driven workloads, variable traffic, rapid prototyping, low-ops teams |
| **Avoid when** | Long-running processes, low-latency requirements (<10ms), need for persistent connections |

---

## What Is It?

Serverless means you write **functions** instead of services. The cloud provider handles everything else: provisioning, scaling, availability, patching. You pay only for what you use (per invocation).

**"Serverless" doesn't mean no servers — it means YOU don't manage servers.**

---

## Diagram Reference
`./diagram.svg`

---

## Structure

```
HTTP Request → API Gateway → Lambda Function (PlaceOrder)
                                 │
                                 ├──► DynamoDB (write order)
                                 ├──► SQS (publish event)
                                 └──► Return response

SQS event → Lambda Function (ProcessPayment)
                ├──► Stripe API (charge card)
                └──► SNS (publish PaymentCharged event)

SNS event → Lambda Function (SendNotification)
                └──► SES (send email)
```

---

## In Clean Architecture Terms

A Lambda function handler IS the **Controller** (Interface Adapter layer). The business logic should still live in Use Cases and Entities — NOT in the handler.

```typescript
// ❌ BAD: Business logic in the handler
export const handler = async (event: APIGatewayEvent) => {
  const body = JSON.parse(event.body!);
  // VALIDATION, BUSINESS RULES, DB CALLS all in the handler!
  const order = await db.put({ ... });
  await stripe.charges.create({ ... });
  return { statusCode: 200, body: JSON.stringify({ orderId: order.id }) };
};

// ✅ GOOD: Handler is just a thin adapter
export const handler = async (event: APIGatewayEvent) => {
  // Handler = Controller (Interface Adapter layer)
  const request = PlaceOrderRequestMapper.fromApiGateway(event);
  const result = await placeOrderUseCase.execute(request);  // use case call
  return { statusCode: 200, body: JSON.stringify(result) };
};

// Use Case: SAME as non-serverless — platform-agnostic
class PlaceOrderUseCase {
  constructor(
    private orders: IOrderRepository,   // interface
    private payments: IPaymentGateway   // interface
  ) {}
  async execute(req: PlaceOrderRequest): Promise<PlaceOrderResponse> { /* ... */ }
}

// Adapters: DynamoDB implementation (can swap to RDS without changing use case)
class DynamoOrderRepository implements IOrderRepository { /* ... */ }
```

**The UseCase can be unit-tested locally without AWS, without DynamoDB, without Stripe.**

---

## Cold Start Problem

Lambda functions have a "cold start" — when a function hasn't been invoked recently, AWS must initialize the runtime. This adds 100ms-3000ms latency.

**Mitigation strategies:**
- Provisioned Concurrency (keep N instances warm — costs money)
- Keep handlers small (fast initialization)
- Use Lambda Layers for shared code (reduces bundle size)
- Use Rust or Go runtime instead of Node.js/Python (faster cold start)

---

## Resource Consumption

| Resource | Usage | Notes |
|---|---|---|
| **CPU** | Billed per ms of execution | Scales to zero when idle — biggest cost advantage |
| **Memory** | Configurable (128MB-10GB per function) | More memory = more CPU allocated |
| **Concurrency** | Auto-scales to 1000+ concurrent (configurable) | Each request = potentially new Lambda instance |
| **Cold start** | 100ms-3000ms (first invocation) | Provisioned Concurrency eliminates this (at cost) |
| **Cost** | Pay per 100ms execution | Free tier: 1M requests/month — ideal for low traffic |
| **Network** | Stateless — no persistent connections | Use connection pooling (RDS Proxy) for DB |

---

## Benefits

1. **Zero server management** — no EC2, no patching, no capacity planning
2. **Auto-scaling** — 0 to 10,000 concurrent automatically; no configuration
3. **Pay per use** — idle cost = $0 (vs $200/month for idle EC2)
4. **Fast deployment** — `serverless deploy` — functions live in seconds
5. **Infinite scalability** — AWS scales Lambda to meet demand automatically
6. **Event-driven natively** — S3 events, DynamoDB streams, SQS, SNS trigger functions naturally

---

## Problems It Solves Best

| Problem | Why Serverless Wins |
|---|---|
| "Traffic spikes 100x during Black Friday, then drops to near zero" | Lambda scales up automatically, costs $0 when idle |
| "We have 50 small batch jobs that run once a day" | 50 Lambda functions; no need to run 50 EC2 instances 24/7 |
| "MVP that might get zero users" | Cost is zero with zero users; scale to millions without migration |
| "Image resize, PDF generation on upload" | S3 trigger → Lambda → processed result; perfect fit |
| "Webhooks from Stripe/GitHub" | Function wakes on each webhook; no always-running server needed |

---

## Costs / Tradeoffs

1. **Cold starts** — 100ms-3s delay for first request after idle period
2. **Execution time limit** — AWS Lambda: 15-minute max (not for long jobs)
3. **Stateless** — no persistent memory between invocations; use external state (Redis, DynamoDB)
4. **Vendor lock-in** — AWS Lambda syntax differs from GCP Cloud Functions; migration is painful
5. **Debugging is hard** — distributed by nature; need structured logging + X-Ray tracing
6. **Not good for persistent connections** — WebSocket, long-polling require special setup

---

## Big Tech Examples

### Netflix
- **Architecture:** Hundreds of Lambda functions for media processing
- **Use case:** Video encoding pipeline: S3 upload → Lambda trigger → transcoding job dispatch
- **Good at:** Processing millions of videos; scales to 0 when no uploads are happening

### Airbnb
- **Architecture:** Lambda for image processing and data ETL pipelines
- **Use case:** When a host uploads a photo, Lambda resizes it to 5 different dimensions instantly
- **Good at:** Spiky workloads (rush of bookings around holidays) scale automatically

### Coca-Cola
- **Architecture:** Serverless vending machine backend (AWS Lambda case study)
- **Use case:** Each vending machine purchase triggers a Lambda function; 2M transactions/month
- **Good at:** Cost — 70% cost reduction vs EC2-based system

### iRobot (Roomba)
- **Architecture:** AWS Lambda for IoT device events
- **Use case:** Each Roomba vacuum sends cleaning session data → Lambda processes → DynamoDB stores
- **Good at:** 10M+ devices; Lambda scales to match traffic without provisioning

### The Guardian (News)
- **Architecture:** Lambda for breaking news content delivery
- **Use case:** CMS publish event → Lambda → CDN invalidation → updated pages worldwide
- **Good at:** Breaking news spikes (election results) — Lambda handles the burst automatically

---

## Serverless vs Containers

| | Serverless | Containers (Kubernetes) |
|---|---|---|
| **Management** | Zero | Medium (Kubernetes is complex) |
| **Scaling** | Automatic | Manual / HPA config |
| **Cold start** | Yes (100ms-3s) | No (always running) |
| **Cost at low traffic** | Near zero | Fixed ($$ for idle pods) |
| **Long-running jobs** | Not suitable | Perfect |
| **Max execution** | 15 min | Unlimited |
| **State** | External only | Can be stateful |

---

## Senior Architecture Advice

> "Serverless is the right default for event-driven workloads with variable traffic. But never put business logic in the handler — that's where most serverless architectures go wrong. Treat the Lambda handler like an Express controller: it translates the event into a request DTO, calls a use case, translates the response. The use case knows nothing about Lambda. This means you can run the same business logic locally, in a Lambda, in a container, or in a test — without changing a line of business logic."

---

## Key Takeaway

> Serverless Architecture is Clean Architecture where the **Frameworks & Drivers layer is managed by AWS/GCP/Azure**. The handler = Controller (Interface Adapter). The use case = application logic. The DynamoDB adapter = outbound adapter. Clean Architecture's framework independence is what makes serverless functions testable, portable between clouds, and runnable locally.
