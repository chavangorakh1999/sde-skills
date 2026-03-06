---
name: service-decomposition
description: "Monolith-to-microservices: identify seams by business capability and bounded contexts (DDD), event contracts, migration sequencing. Use when deciding whether and how to break up a monolith."
---

## Service Decomposition

The hardest question about microservices is: "Should we?" The second hardest is: "Where do we draw the boundaries?"

Don't decompose because microservices are trendy. Decompose when a specific problem demands it.

### Context

Service or monolith to analyze: **$ARGUMENTS**

---

### When to Decompose

**Valid reasons to break up a monolith:**
- Independent scaling: one component needs 10x more compute than others
- Independent deployment: teams blocked on each other's releases
- Technology mismatch: one component needs Python (ML), rest is Node.js
- Fault isolation: one component's failures bringing down the whole system
- Team autonomy at scale: > 6 engineers on one codebase creates coordination cost
- Compliance boundary: PCI/HIPAA data must be in its own isolated environment

**Invalid reasons (resist these):**
- "Microservices are modern" — not a technical reason
- "The monolith is messy" — clean the monolith first
- < 5 engineers — overhead exceeds benefit
- < 2 years of product-market fit — service boundaries will be wrong and expensive to change

**Rule of thumb:** A well-structured monolith beats a poorly-designed microservices architecture every time. Get boundaries right in the monolith first, then extract.

---

### Step 1: Identify Bounded Contexts (DDD)

A bounded context is a business domain with its own language, data, and rules.

```
Identify by asking:
- "Who owns this data? Who is responsible when it's wrong?"
- "What team would change this independently?"
- "Does this concept mean the same thing across the whole system?"

Example: E-commerce monolith
- "Product" means different things:
  - Catalog team: product with description, images, SEO
  - Inventory team: product as a stock-keeping unit (SKU) with count
  - Shipping team: product as a physical item with weight/dimensions
  -> These are three different bounded contexts sharing a concept name

Bounded contexts found:
+------------------+------------------------+--------------------------+
| Context          | Domain Language         | Data It Owns             |
+------------------+------------------------+--------------------------+
| Product Catalog  | Product, Category, Tag  | product_info, categories |
| Inventory        | SKU, Stock Level        | inventory, warehouses    |
| Orders           | Order, Line Item        | orders, order_items      |
| Payments         | Payment, Refund         | payments, transactions   |
| Shipping         | Shipment, Carrier       | shipments, tracking      |
| Users/Auth       | Customer, Session       | users, sessions          |
+------------------+------------------------+--------------------------+
```

---

### Step 2: Identify the Seams

A seam is where you can split the code with minimal coupling.

```javascript
// Signs of a natural seam:
// - A service class that only calls other classes from one context
// - A DB table that's only joined with tables from the same context
// - A team that owns a cohesive set of features
// - A component with different scaling characteristics

// Signs of problematic coupling (seam is NOT here):
// - Shared mutable state between contexts
// - Direct in-process function calls crossing context boundaries
// - Tables JOIN'ed across what would be service boundaries
// - A "utils" or "common" module used by everything

// Example seam analysis for UserService:
class UserService {
  // Auth concern (natural seam — own context)
  async login() { ... }
  async logout() { ... }
  async refreshToken() { ... }

  // Profile concern (natural seam — different team, different change rate)
  async updateProfile() { ... }
  async uploadAvatar() { ... }

  // Billing concern (natural seam — PCI compliance, different team)
  async addPaymentMethod() { ... }
  async updateBillingAddress() { ... }
}
// Three distinct bounded contexts masquerading as one service
```

---

### Step 3: Map the Dependencies

Before extracting, understand what talks to what:

```
Create a dependency map:
UserService    -> DB (users, sessions)
               -> EmailService (welcome email, password reset)
               -> NotificationService (account events)
               -> BillingService (subscription status check)

OrderService   -> DB (orders, order_items)
               -> ProductService (product info, pricing)
               -> InventoryService (stock check)
               -> PaymentService (charge)
               -> ShippingService (create shipment)
               -> NotificationService (order updates)

// High fan-out services (OrderService calls 5 services) are:
// a) harder to extract (more integration points)
// b) more likely to cause cascading failures
// c) candidates for saga orchestration
```

---

### Step 4: Event Contract Design

When services communicate via events, define the contract before implementation:

```javascript
// Event schema example (use JSON Schema or TypeScript interface for enforcement)

// order-service emits:
const OrderPlacedEvent = {
  eventType: 'order.placed',
  version: '1.0',
  schema: {
    orderId: 'string (UUID)',
    customerId: 'string (UUID)',
    items: [{ productId: 'string', quantity: 'number', unitPrice: 'number (cents)' }],
    total: 'number (cents)',
    currency: 'string (ISO 4217)',
    placedAt: 'string (ISO 8601 UTC)'
  }
};

// inventory-service consumes order.placed and emits:
const StockReservedEvent = {
  eventType: 'inventory.stock_reserved',
  version: '1.0',
  schema: {
    orderId: 'string',
    reservationId: 'string',
    reservedAt: 'string'
  }
};

// payment-service consumes inventory.stock_reserved and emits:
const PaymentCapturedEvent = {
  eventType: 'payment.captured',
  version: '1.0',
  schema: {
    orderId: 'string',
    paymentId: 'string',
    amount: 'number (cents)',
    capturedAt: 'string'
  }
};

// Rule: events are immutable facts — never change the schema of a published event
// Instead: create v2 event type, publish both until consumers are migrated
```

---

### Step 5: Decomposition Sequencing

Extract in dependency order (least dependencies first):

```
Extraction sequence (easiest to hardest):
1. Notification/Email service — high fan-in but no upstream dependencies, easy to extract async
2. Auth service — clear boundaries, mostly read-heavy, well-understood
3. User Profile service — medium complexity, can be extracted after auth
4. Product Catalog — read-heavy, can be extracted without impacting write paths
5. Inventory service — needs careful saga design with orders
6. Payments service — highest risk (financial), extract last with most care

General principle:
- Extract leaf services first (they depend on nothing)
- Extract core services last (everything depends on them)
- Defer the trickiest integrations until you've built confidence
```

---

### Step 6: Anti-Patterns to Avoid

```
Distributed Monolith: services extracted but still sharing a database
  - Looks like microservices, acts like a monolith
  - Every "service" is still coupled through shared tables
  - Fix: each service owns its data; communicate via API or events

Chatty Services: service A makes 10 synchronous calls to service B per request
  - Network latency multiplied by call count
  - Fix: aggregate data at read time (BFF pattern), or denormalize

Shared Library Coupling: all services depend on a "common" package
  - Changing common requires deploying all services
  - Fix: duplicate is better than wrong abstraction across services

Premature Decomposition: services extracted before bounded contexts are stable
  - As the product evolves, you'll be moving functionality between services
  - Fix: stabilize domain model in the monolith first
```

---

### Output Format

```
## Service Decomposition: [System Name]

### Should We Decompose?
[Evidence for/against — don't skip this]

### Bounded Contexts Found
| Context | Domain Language | Data It Owns | Team Owner |

### Dependency Map
[Diagram of current inter-service calls]

### Extraction Sequence
1. [Service name] — reason to extract first
2. ...

### Event Contracts
[Key events with their schemas]

### Migration Strategy
[Strangler fig phases for the first extraction]

### Risks
[What could go wrong, how to mitigate]
```
