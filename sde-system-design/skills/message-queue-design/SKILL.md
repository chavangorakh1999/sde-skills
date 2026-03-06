---
name: message-queue-design
description: "Kafka/SQS/RabbitMQ/Bull topology, ordering guarantees, consumer groups, DLQ patterns. Use when decoupling async work from the request path or building event-driven pipelines."
---

## Message Queue Design

Queues decouple producers from consumers, absorb traffic spikes, and enable async processing. Choose the right tool for your durability, ordering, and throughput requirements.

### Context

Use case or system: **$ARGUMENTS**

---

### When to Use a Queue

Use a queue when:
- Work takes > 500ms and doesn't need to block the HTTP response (email sending, PDF generation, video transcoding)
- You need to absorb write spikes without dropping requests (order processing at sale time)
- Multiple services need to react to the same event (order placed -> billing, inventory, notification)
- You need retry with backoff for unreliable external calls (third-party API, webhook delivery)

Don't use a queue when:
- The caller needs the result synchronously (use sync RPC/HTTP)
- You only have one consumer and simple retry (HTTP retry with backoff is simpler)
- The processing is fast and failure is acceptable (just do it inline)

---

### Choosing the Right Queue

| | **Bull/BullMQ** | **RabbitMQ** | **SQS** | **Kafka** |
|---|---|---|---|---|
| **Best for** | Node.js job queues | Multi-language task routing | AWS-native async jobs | High-throughput event streaming |
| **Throughput** | ~10K jobs/sec | ~50K msgs/sec | ~3K msgs/sec per queue | >1M msgs/sec |
| **Ordering** | FIFO per queue | Per-queue FIFO | Best-effort (FIFO queue: strict) | Per-partition ordered |
| **Durability** | Redis persistence | Disk (durable queues) | S3-backed, 14-day retention | Configurable, up to years |
| **Exactly-once** | No | No (at-least-once) | No (SQS FIFO: ~yes) | No (idempotent consumer needed) |
| **Complexity** | Low | Medium | Low | High |
| **Use when** | Node.js app, Redis already used | Multi-language, complex routing | AWS shop, simple async | Event log, high-throughput, replay |

---

### Bull/BullMQ (Node.js — start here)

```javascript
// BullMQ with Redis backend — great for Node.js microservices

import { Queue, Worker, QueueEvents } from 'bullmq';

const connection = { host: 'localhost', port: 6379 };

// Producer: add jobs to queue
const emailQueue = new Queue('email-notifications', { connection });

async function sendWelcomeEmail(userId, email) {
  await emailQueue.add(
    'welcome-email',
    { userId, email },
    {
      attempts: 3,                  // retry up to 3 times
      backoff: {
        type: 'exponential',
        delay: 2000                 // 2s, 4s, 8s
      },
      removeOnComplete: { count: 100 },  // keep last 100 completed jobs
      removeOnFail: { count: 500 }       // keep last 500 failed jobs for inspection
    }
  );
}

// Consumer: process jobs
const worker = new Worker(
  'email-notifications',
  async (job) => {
    const { userId, email } = job.data;
    await sendEmailViaProvider(email, 'Welcome!', generateWelcomeTemplate(userId));
    // Return value is stored in job.returnvalue
    return { sent: true, timestamp: new Date().toISOString() };
  },
  {
    connection,
    concurrency: 5,              // process up to 5 jobs simultaneously
    limiter: {                   // rate limit to 100 jobs per second
      max: 100,
      duration: 1000
    }
  }
);

// Handle failures
worker.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed: ${err.message}`);
  // After all attempts exhausted, job moves to 'failed' state
  // Monitor failed jobs, alert if count exceeds threshold
});

// Dead Letter Queue pattern with BullMQ:
// Failed jobs (after all retries) stay in 'failed' state
// Inspect with: await emailQueue.getFailed(0, 50)
// Retry manually: await job.retry()
// Or move to a separate DLQ queue for human review
```

---

### SQS (AWS-native)

```javascript
// Standard Queue: at-least-once, best-effort ordering, high throughput
// FIFO Queue: exactly-once processing, strict ordering, 300 TPS (3000 with batching)

import { SQSClient, SendMessageCommand, ReceiveMessageCommand, DeleteMessageCommand } from '@aws-sdk/client-sqs';

const sqs = new SQSClient({ region: 'us-east-1' });

// Producer
async function enqueueOrderProcessing(orderId, orderData) {
  await sqs.send(new SendMessageCommand({
    QueueUrl: process.env.ORDER_QUEUE_URL,
    MessageBody: JSON.stringify({ orderId, ...orderData }),
    // For FIFO queue, add:
    // MessageGroupId: customerId,      // ordering within a customer's orders
    // MessageDeduplicationId: orderId  // exactly-once within 5-minute window
  }));
}

// Consumer (long-polling pattern)
async function processOrders() {
  while (true) {
    const response = await sqs.send(new ReceiveMessageCommand({
      QueueUrl: process.env.ORDER_QUEUE_URL,
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 20,            // long polling — reduces empty receives
      VisibilityTimeout: 60           // hide message for 60s while processing
    }));

    for (const message of response.Messages ?? []) {
      try {
        await processOrder(JSON.parse(message.Body));
        // Delete only after successful processing
        await sqs.send(new DeleteMessageCommand({
          QueueUrl: process.env.ORDER_QUEUE_URL,
          ReceiptHandle: message.ReceiptHandle
        }));
      } catch (err) {
        console.error(`Failed to process message ${message.MessageId}:`, err);
        // Don't delete — message becomes visible again after VisibilityTimeout
        // After maxReceiveCount failures -> moves to DLQ automatically
      }
    }
  }
}

// DLQ configuration (in Terraform/CDK):
// redrive_policy = {
//   deadLetterTargetArn: order_dlq.arn
//   maxReceiveCount: 3              -- move to DLQ after 3 failed attempts
// }
```

---

### Kafka (high-throughput event streaming)

```javascript
// Kafka: topics -> partitions -> offsets
// Partition = unit of parallelism and ordering
// Consumer group = logical consumer with distributed partition assignment

import { Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092']
});

// Producer
const producer = kafka.producer({
  // Idempotent producer: exactly-once delivery to broker (no duplicate messages)
  idempotent: true,
  // Required for idempotent: acks must be -1 (all in-sync replicas acknowledge)
});

await producer.connect();

async function publishOrderEvent(orderId, eventType, data) {
  await producer.send({
    topic: 'order-events',
    messages: [{
      // Key determines partition assignment — orders from same customer go to same partition
      key: data.customerId,
      value: JSON.stringify({ eventType, orderId, ...data, timestamp: Date.now() })
    }]
  });
}

// Consumer
const consumer = kafka.consumer({ groupId: 'inventory-service' });
await consumer.connect();
await consumer.subscribe({ topic: 'order-events', fromBeginning: false });

await consumer.run({
  // autoCommit: false for manual offset management (process-then-commit)
  autoCommitInterval: 5000,  // commit offsets every 5 seconds
  eachMessage: async ({ topic, partition, message, heartbeat }) => {
    const event = JSON.parse(message.value.toString());

    try {
      await processOrderEvent(event);
      // Offset committed automatically per autoCommitInterval
    } catch (err) {
      // Kafka has no built-in DLQ — implement your own:
      // Option 1: Send to a dead-letter topic
      await producer.send({
        topic: 'order-events-dlq',
        messages: [{ key: message.key, value: message.value, headers: { error: err.message } }]
      });
      // Option 2: Log error and continue (skip poison pill)
    }
  }
});
```

---

### DLQ (Dead Letter Queue) Pattern

```javascript
// Every queue MUST have a DLQ. Without it, poison pill messages block the queue forever.

// DLQ handling workflow:
// 1. Message fails maxReceiveCount times
// 2. Moves to DLQ automatically (SQS) or via code (Kafka, Bull)
// 3. Alert fires when DLQ depth > 0
// 4. On-call engineer investigates: is this a bug? bad data? transient failure?
// 5. Fix the consumer or the data, then replay from DLQ

// DLQ message should include:
// - Original message body (unchanged)
// - Error message/stack trace
// - Number of attempts
// - Timestamps of each failure
// - Source queue name

// DLQ monitoring:
// Metric: dlq_depth (messages in DLQ)
// Alert: dlq_depth > 0 for 5 minutes -> PagerDuty
// SLO: DLQ messages investigated within 1 business hour
```

---

### Ordering Guarantees

```
No ordering guarantee:
  SQS Standard, most distributed queues
  Use when: parallel processing, order doesn't matter
  Example: sending marketing emails — doesn't matter if email B sends before email A

Per-partition ordering (Kafka):
  Messages with same key go to same partition, are ordered within partition
  Use when: events for same entity must be processed in order
  Example: user_id as key -> all events for a user are ordered

Strict global ordering (SQS FIFO, single Bull queue):
  All messages processed in exact send order
  Throughput limited (SQS FIFO: 300 TPS per message group)
  Use when: financial ledger, inventory deductions where order is critical
```

---

### Output Format

```
## Message Queue Design: [System]

### Queue Technology Choice
[Bull / SQS / Kafka / RabbitMQ with rationale]

### Topic/Queue Structure
| Queue/Topic | Producer | Consumer(s) | Ordering | DLQ |

### Message Schema
[JSON schema for each message type]

### Error Handling
[Retry policy, backoff, DLQ configuration, alerting]

### Scalability
[Partition count, consumer scaling, throughput targets]

### Tradeoffs
[At-least-once vs exactly-once, ordering vs throughput, etc.]
```
