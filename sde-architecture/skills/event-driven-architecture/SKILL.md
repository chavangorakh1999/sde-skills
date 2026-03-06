---
name: event-driven-architecture
description: "Event sourcing, CQRS, outbox pattern, at-least-once vs exactly-once delivery, event schema design and versioning. Use when designing event-driven systems or evaluating whether to adopt event sourcing."
---

## Event-Driven Architecture

Events are facts about things that happened. They're immutable, past tense ("order was placed"), and can be replayed to rebuild state. Not every system benefits from EDA — know when to use it.

### Context

System or design problem: **$ARGUMENTS**

---

### Event-Driven Patterns Overview

```
Simple async (use this first):
  Request -> Service A -> Queue -> Service B
  Loose coupling, async processing, easy retry
  Not event sourcing — just messaging

Event sourcing (use when you need the full history):
  Instead of storing current state, store every event that led to current state
  Can replay events to rebuild state at any point in time
  Audit log comes for free

CQRS (Command Query Responsibility Segregation):
  Write side: accepts commands, validates, emits events
  Read side: subscribes to events, builds optimized read models
  Write and read models are separated
  Often paired with event sourcing but not required
```

---

### Simple Event-Driven (Start Here)

```javascript
// Domain events — facts about what happened in your system
// Emitted by the domain service, consumed by other services async

// Event definition
const OrderPlacedEvent = {
  type: 'ORDER_PLACED',
  version: 1,
  payload: {
    orderId: 'string',
    customerId: 'string',
    items: [{ productId: 'string', quantity: 'number', price: 'number' }],
    total: 'number',
    placedAt: 'ISO 8601'
  }
};

// Publishing events (with BullMQ)
class OrderService {
  async place(orderData) {
    const order = await this.orderRepo.create(orderData);

    // Emit event — handlers run asynchronously
    await this.eventBus.publish('ORDER_PLACED', {
      orderId: order.id,
      customerId: order.customerId,
      items: order.items,
      total: order.total,
      placedAt: order.createdAt.toISOString()
    });

    return order;
  }
}

// Consuming events
class InventoryHandler {
  async handle(event) {
    if (event.type !== 'ORDER_PLACED') return;
    for (const item of event.payload.items) {
      await this.inventory.reserve(item.productId, item.quantity, event.payload.orderId);
    }
  }
}

class EmailHandler {
  async handle(event) {
    if (event.type !== 'ORDER_PLACED') return;
    await this.mailer.sendOrderConfirmation(event.payload.customerId, event.payload);
  }
}
```

---

### Outbox Pattern (Guaranteed Delivery)

**Problem:** If you publish an event after writing to the DB, and the publish fails, the event is lost. You have inconsistency between your DB state and your events.

```javascript
// Outbox pattern: write event to DB in the same transaction as the state change
// A separate process polls the outbox table and publishes events
// Atomicity is guaranteed by the DB transaction

// Step 1: Write state + event to DB atomically
async function placeOrder(orderData) {
  return db.transaction(async (trx) => {
    // Write the domain state
    const order = await trx('orders').insert(orderData).returning('*');

    // Write the event to the outbox (same transaction)
    await trx('outbox_events').insert({
      id: uuid(),
      type: 'ORDER_PLACED',
      aggregate_id: order[0].id,
      payload: JSON.stringify(order[0]),
      created_at: new Date()
    });

    return order[0];
  });
}

// Step 2: Outbox relay process (runs independently, polls outbox table)
async function relayOutboxEvents() {
  while (true) {
    const events = await db('outbox_events')
      .where('published_at', null)
      .orderBy('created_at', 'asc')
      .limit(100)
      .forUpdate()      // row-level lock for concurrency
      .skipLocked();    // skip events being processed by another relay instance

    for (const event of events) {
      try {
        await messageQueue.publish(event.type, JSON.parse(event.payload));
        await db('outbox_events')
          .where({ id: event.id })
          .update({ published_at: new Date() });
      } catch (err) {
        logger.error('Failed to relay event', { eventId: event.id, err });
        // Retry next polling cycle
      }
    }

    if (events.length === 0) await sleep(1000);  // poll every 1s when idle
  }
}

// Cleanup: delete published events older than 7 days
// DELETE FROM outbox_events WHERE published_at < NOW() - INTERVAL '7 days'
```

---

### Event Sourcing

Store every event that happened, not just current state. Current state = replay of events.

```javascript
// When to use event sourcing:
// - Audit trail is legally required (financial, healthcare, compliance)
// - You need to rebuild state at any point in time (debugging, analytics)
// - Temporal queries: "what was the account balance on 2024-01-15?"
// - Complex business processes where state transitions matter

// When NOT to use:
// - Simple CRUD applications (massive overhead for no benefit)
// - Team unfamiliar with the pattern (steep learning curve, easy to get wrong)
// - You need low-latency reads (read side requires projection, adds complexity)

// Event store schema (simplified)
// CREATE TABLE events (
//   id          UUID PRIMARY KEY,
//   stream_id   UUID NOT NULL,   -- the aggregate ID (orderId, userId, etc.)
//   event_type  VARCHAR(100) NOT NULL,
//   event_data  JSONB NOT NULL,
//   event_version INT NOT NULL,  -- sequence within the stream
//   created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
// );
// CREATE UNIQUE INDEX ON events(stream_id, event_version);  -- optimistic locking

// Aggregate class
class Order {
  constructor() {
    this.id = null;
    this.status = 'pending';
    this.items = [];
    this.total = 0;
    this._version = 0;
    this._pendingEvents = [];
  }

  // Command handler: validate and emit event
  place(orderData) {
    if (this.status !== 'pending') throw new Error('Order already placed');
    this._apply({
      type: 'ORDER_PLACED',
      data: { ...orderData, placedAt: new Date().toISOString() }
    });
  }

  // Command handler
  cancel(reason) {
    if (!['pending', 'placed'].includes(this.status)) {
      throw new Error(`Cannot cancel order in status: ${this.status}`);
    }
    this._apply({ type: 'ORDER_CANCELLED', data: { reason, cancelledAt: new Date().toISOString() } });
  }

  // State reconstruction
  _apply(event) {
    switch (event.type) {
      case 'ORDER_PLACED':
        this.status = 'placed';
        this.items = event.data.items;
        this.total = event.data.total;
        break;
      case 'ORDER_CANCELLED':
        this.status = 'cancelled';
        break;
    }
    this._version++;
    this._pendingEvents.push(event);
  }

  // Rebuild from event history
  static fromHistory(events) {
    const order = new Order();
    for (const event of events) order._apply(event);
    order._pendingEvents = [];  // historical events already published
    return order;
  }
}
```

---

### CQRS

```javascript
// Write side: commands change state, emit events
// Read side: event handlers build optimized read models (denormalized, fast queries)

// Write side command handler
class PlaceOrderCommand {
  async handle({ customerId, items }) {
    const order = new Order();
    order.place({ customerId, items, total: calculateTotal(items) });

    // Persist events
    await eventStore.appendToStream(`order-${order.id}`, order._pendingEvents, order._version);
    return order.id;
  }
}

// Read side: event handler builds a flat, queryable view
// This is separate from the write model — optimized for reads
class OrderSummaryProjection {
  async handle(event) {
    switch (event.type) {
      case 'ORDER_PLACED':
        await db('order_summaries').insert({
          id: event.streamId,
          customer_id: event.data.customerId,
          status: 'placed',
          total: event.data.total,
          item_count: event.data.items.length,
          placed_at: event.data.placedAt
        });
        break;

      case 'ORDER_CANCELLED':
        await db('order_summaries')
          .where({ id: event.streamId })
          .update({ status: 'cancelled' });
        break;
    }
  }
}

// Read query hits the projection (denormalized, fast)
app.get('/orders', async (req, res) => {
  const orders = await db('order_summaries')
    .where({ customer_id: req.user.id })
    .orderBy('placed_at', 'desc')
    .limit(20);
  res.json(orders);
});
// No JOIN, no complex query — just a simple table scan with an index
```

---

### Event Schema Versioning

Events are immutable once published. Consumers exist in the wild. Breaking changes are catastrophic.

```javascript
// Rule: never change the schema of a published event
// Instead: create a new version of the event type

// v1 (original)
{ type: 'ORDER_PLACED', version: 1, data: { orderId, customerId, total } }

// v2 (added currency field — non-breaking? still use v2 for explicitness)
{ type: 'ORDER_PLACED', version: 2, data: { orderId, customerId, total, currency: 'USD' } }

// Consumer that handles both versions:
function handleOrderPlaced(event) {
  const currency = event.version >= 2 ? event.data.currency : 'USD';  // default for v1
  // process with currency
}

// Upcasting: transform old events to new format when loading from event store
function upcastEvent(event) {
  if (event.type === 'ORDER_PLACED' && event.version === 1) {
    return { ...event, version: 2, data: { ...event.data, currency: 'USD' } };
  }
  return event;
}
```

---

### Output Format

```
## Event-Driven Design: [System]

### Pattern Choice
[Simple events / Outbox / Event sourcing / CQRS — with rationale]

### Event Catalog
| Event Type | Producer | Consumers | Schema |

### Event Schema
[JSON structure for each event type]

### Delivery Guarantee
[At-least-once with idempotent consumers / Exactly-once via outbox / etc.]

### Versioning Strategy
[How you'll evolve events without breaking consumers]

### Tradeoffs
[Complexity added vs benefits gained]
```
