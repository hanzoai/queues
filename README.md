# Hanzo Queues

High-throughput distributed work queues for reliable, scalable job processing.

## Features

- **Priority Queues** - Multiple priority levels with fair scheduling
- **Dead Letter Queues** - Automatic failed message routing for analysis
- **Rate Limiting** - Per-queue and per-consumer rate controls
- **Exactly-Once Processing** - Deduplication and idempotency guarantees
- **Delayed Messages** - Schedule messages for future delivery
- **Batch Operations** - Efficient bulk enqueue and dequeue
- **Consumer Groups** - Distributed processing with automatic rebalancing
- **Observability** - Built-in metrics, tracing, and monitoring

## Quick Start

### Docker

```bash
docker run -d \
  -p 8080:8080 \
  -e NATS_URL=nats://localhost:4222 \
  -e REDIS_URL=redis://localhost:6379 \
  hanzoai/queues:latest
```

### Docker Compose

```yaml
version: '3.8'
services:
  queues:
    image: hanzoai/queues:latest
    ports:
      - "8080:8080"
    environment:
      NATS_URL: nats://nats:4222
      REDIS_URL: redis://redis:6379
    depends_on:
      - nats
      - redis

  nats:
    image: nats:latest
    command: ["--jetstream"]
    ports:
      - "4222:4222"
```

## SDK Examples

### Python

```python
from hanzo import Queues

queues = Queues(api_key="your-api-key")

# Create a queue
queue = queues.create(
    name="email-notifications",
    config={
        "max_retries": 3,
        "visibility_timeout": 30,
        "dead_letter_queue": "email-dlq",
        "rate_limit": {"requests": 100, "per": "second"}
    }
)

# Enqueue a message
message = queues.enqueue(
    queue="email-notifications",
    payload={"to": "user@example.com", "template": "welcome"},
    priority=1,  # Higher = more urgent
    delay=60,    # Delay 60 seconds
    dedup_id="welcome-user-123"  # Prevent duplicates
)

# Consume messages
@queues.consumer("email-notifications", concurrency=10)
async def process_email(message):
    # Process the message
    await send_email(message.payload)
    # Message auto-acknowledged on success
    # Auto-retried on exception

# Or manual consumption
messages = queues.receive(queue="email-notifications", max_messages=10)
for msg in messages:
    try:
        process(msg)
        queues.ack(msg.id)
    except Exception:
        queues.nack(msg.id)  # Return to queue
```

### TypeScript

```typescript
import { Queues } from '@hanzo/sdk';

const queues = new Queues({ apiKey: 'your-api-key' });

// Create a queue with rate limiting
const queue = await queues.create({
  name: 'image-processing',
  config: {
    maxRetries: 5,
    visibilityTimeout: 300,
    deadLetterQueue: 'image-dlq',
    rateLimit: { requests: 50, per: 'second' }
  }
});

// Enqueue with priority
await queues.enqueue({
  queue: 'image-processing',
  payload: { imageUrl: 'https://...', operations: ['resize', 'compress'] },
  priority: 2,
  metadata: { userId: '123' }
});

// Batch enqueue
await queues.enqueueBatch({
  queue: 'image-processing',
  messages: [
    { payload: { imageUrl: '...' } },
    { payload: { imageUrl: '...' } }
  ]
});

// Consumer with automatic scaling
queues.consume({
  queue: 'image-processing',
  concurrency: 20,
  handler: async (message) => {
    await processImage(message.payload);
  },
  onError: async (error, message) => {
    console.error('Failed:', error);
    // Message automatically retried or sent to DLQ
  }
});
```

### Go

```go
package main

import (
    "github.com/hanzoai/go-sdk/queues"
)

func main() {
    client := queues.New("your-api-key")

    // Create a queue
    queue, err := client.Create(queues.CreateParams{
        Name: "order-processing",
        Config: queues.Config{
            MaxRetries:        3,
            VisibilityTimeout: 60,
            DeadLetterQueue:   "order-dlq",
        },
    })

    // Enqueue a message
    msg, err := client.Enqueue(queues.EnqueueParams{
        Queue:    "order-processing",
        Payload:  map[string]interface{}{"orderId": "123"},
        Priority: 1,
    })

    // Consume messages
    client.Consume("order-processing", queues.ConsumerConfig{
        Concurrency: 10,
        Handler: func(msg *queues.Message) error {
            return processOrder(msg.Payload)
        },
    })
}
```

## API Reference

### Create a Queue

```bash
curl -X POST https://queues.hanzo.ai/v1/queues \
  -H "Authorization: Bearer $HANZO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-queue",
    "config": {
      "maxRetries": 3,
      "visibilityTimeout": 30,
      "deadLetterQueue": "my-dlq"
    }
  }'
```

### Enqueue a Message

```bash
curl -X POST https://queues.hanzo.ai/v1/queues/my-queue/messages \
  -H "Authorization: Bearer $HANZO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "payload": {"key": "value"},
    "priority": 1,
    "delay": 0
  }'
```

### Receive Messages

```bash
curl "https://queues.hanzo.ai/v1/queues/my-queue/messages?maxMessages=10" \
  -H "Authorization: Bearer $HANZO_API_KEY"
```

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `NATS_URL` | NATS JetStream connection string | required |
| `REDIS_URL` | Redis for rate limiting and dedup | required |
| `PORT` | HTTP server port | `8080` |
| `LOG_LEVEL` | Logging level | `info` |
| `DEFAULT_VISIBILITY_TIMEOUT` | Default message lock duration | `30s` |
| `DEFAULT_MAX_RETRIES` | Default retry attempts | `3` |
| `MAX_BATCH_SIZE` | Maximum batch operation size | `100` |

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Queues API                          │
└─────────────────────────┬───────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Queue A  │    │ Queue B  │    │   DLQ    │
    │ Priority │    │ Priority │    │          │
    └────┬─────┘    └────┬─────┘    └──────────┘
         │               │
         └───────┬───────┘
                 ▼
    ┌─────────────────────────────────────────┐
    │           NATS JetStream                │
    │     (Persistence & Replication)         │
    └─────────────────────────────────────────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌────────┐  ┌────────┐  ┌────────┐
│Consumer│  │Consumer│  │Consumer│
│ Group  │  │ Group  │  │ Group  │
└────────┘  └────────┘  └────────┘
```

## Queue Patterns

### Priority Queue

```python
# High priority messages processed first
queues.enqueue(queue="tasks", payload={...}, priority=10)  # Urgent
queues.enqueue(queue="tasks", payload={...}, priority=1)   # Normal
```

### Delayed Queue

```python
# Process message after 1 hour
queues.enqueue(queue="reminders", payload={...}, delay=3600)
```

### Dead Letter Queue

```python
# Failed messages automatically routed to DLQ after max retries
dlq_messages = queues.receive(queue="my-dlq", max_messages=10)
for msg in dlq_messages:
    analyze_failure(msg)
    # Optionally replay: queues.enqueue(queue="original", payload=msg.payload)
```

## License

MIT License - see [LICENSE](LICENSE) for details.

---

Part of the [Hanzo](https://hanzo.ai) execution layer.
