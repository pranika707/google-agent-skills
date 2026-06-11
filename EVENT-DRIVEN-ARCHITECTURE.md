---
name: pubsub-workflows
description: >
  Design and implement event-driven architectures using Google Cloud Pub/Sub.
  Use when the user wants to create Pub/Sub topics or subscriptions, publish messages,
  process events with push or pull subscribers, set up dead-letter topics, implement
  fan-out patterns, connect Pub/Sub to Cloud Run or Cloud Functions, or design
  async workflows. Triggers on "Pub/Sub", "pubsub", "topic", "subscription",
  "event-driven", "message queue", "async events on GCP".
license: Apache-2.0
compatibility: Requires gcloud CLI or google-cloud-pubsub Python/Node SDK
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: Pub/Sub, Cloud Run, Cloud Functions, BigQuery (streaming)
---

# Cloud Pub/Sub Workflows

## Core Concepts

- **Topic** — Named channel where publishers send messages
- **Subscription** — Named resource representing the stream of messages from a topic
- **Pull subscription** — Subscriber explicitly requests messages (good for batch workers)
- **Push subscription** — Pub/Sub delivers messages to a webhook URL (good for Cloud Run/Functions)
- **Dead-letter topic** — Receives messages that couldn't be processed after N attempts

## Setup via CLI

```bash
# Create a topic
gcloud pubsub topics create my-events

# Create a pull subscription
gcloud pubsub subscriptions create my-worker-sub \
  --topic=my-events \
  --ack-deadline=60 \
  --message-retention-duration=7d

# Create a push subscription (delivers to Cloud Run)
gcloud pubsub subscriptions create my-push-sub \
  --topic=my-events \
  --push-endpoint=https://my-service-xyz-uc.a.run.app/events \
  --push-auth-service-account=my-sa@PROJECT_ID.iam.gserviceaccount.com \
  --ack-deadline=30

# Create a dead-letter topic and attach it
gcloud pubsub topics create my-events-dlq

gcloud pubsub subscriptions modify-push-config my-worker-sub \
  --push-endpoint=""  # switch back to pull if needed

gcloud pubsub subscriptions update my-worker-sub \
  --dead-letter-topic=my-events-dlq \
  --max-delivery-attempts=5

# Publish a test message
gcloud pubsub topics publish my-events \
  --message='{"type":"user.signup","userId":"123"}' \
  --attribute="source=api,env=prod"

# Pull messages manually (testing)
gcloud pubsub subscriptions pull my-worker-sub \
  --limit=10 \
  --auto-ack
```

## Python: Publish Messages

```python
from google.cloud import pubsub_v1
import json

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("my-project", "my-events")

def publish_event(event_type: str, data: dict, **attributes):
    message_data = json.dumps(data).encode("utf-8")
    future = publisher.publish(
        topic_path,
        message_data,
        type=event_type,
        **attributes
    )
    message_id = future.result()  # Blocks until published
    print(f"Published message {message_id}")
    return message_id

# Usage
publish_event("user.signup", {"userId": "123", "email": "alice@example.com"})
```

## Python: Pull Subscriber (Worker)

```python
from google.cloud import pubsub_v1
import json
import time

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path("my-project", "my-worker-sub")

def process_message(message):
    try:
        data = json.loads(message.data.decode("utf-8"))
        event_type = message.attributes.get("type")

        print(f"Processing {event_type}: {data}")

        # Do your work here...
        handle_event(event_type, data)

        message.ack()  # Acknowledge success
    except Exception as e:
        print(f"Error processing message: {e}")
        message.nack()  # Re-deliver (up to max-delivery-attempts)

def handle_event(event_type, data):
    if event_type == "user.signup":
        send_welcome_email(data["email"])
    elif event_type == "order.placed":
        process_order(data["orderId"])

# Start streaming pull
streaming_pull_future = subscriber.subscribe(subscription_path, callback=process_message)
print(f"Listening on {subscription_path}...")

try:
    streaming_pull_future.result(timeout=None)  # Run forever
except KeyboardInterrupt:
    streaming_pull_future.cancel()
```

## Cloud Run Push Handler (Node.js)

```javascript
// server.js — Cloud Run service receiving Pub/Sub push messages
const express = require("express");
const app = express();
app.use(express.json());

app.post("/events", async (req, res) => {
  const pubsubMessage = req.body?.message;
  if (!pubsubMessage) {
    return res.status(400).send("Missing Pub/Sub message");
  }

  // Decode base64 payload
  const data = JSON.parse(
    Buffer.from(pubsubMessage.data, "base64").toString("utf-8")
  );
  const attributes = pubsubMessage.attributes || {};

  console.log("Received event:", attributes.type, data);

  try {
    await handleEvent(attributes.type, data);
    res.status(204).send(); // 2xx = ACK
  } catch (err) {
    console.error("Processing error:", err);
    res.status(500).send(); // 5xx = NACK (will be redelivered)
  }
});

async function handleEvent(type, data) {
  switch (type) {
    case "user.signup":
      await sendWelcomeEmail(data.email);
      break;
    case "order.placed":
      await processOrder(data.orderId);
      break;
    default:
      console.warn("Unknown event type:", type);
  }
}

app.listen(process.env.PORT || 8080);
```

## Fan-Out Pattern

One topic → multiple independent subscribers:

```bash
# Single topic
gcloud pubsub topics create order-events

# Multiple subscribers for different concerns
gcloud pubsub subscriptions create inventory-sub --topic=order-events
gcloud pubsub subscriptions create email-sub --topic=order-events
gcloud pubsub subscriptions create analytics-sub --topic=order-events
```

Each subscription gets its own independent copy of every message.

## BigQuery Streaming Subscription

```bash
# Stream Pub/Sub messages directly into BigQuery (no code needed)
gcloud pubsub subscriptions create bq-streaming-sub \
  --topic=my-events \
  --bigquery-table=my-project:dataset.events \
  --use-topic-schema \
  --write-metadata  # includes message ID, publish time etc.
```

## Monitoring & Debugging

```bash
# View subscription backlog (oldest_unacked_message_age is key)
gcloud pubsub subscriptions describe my-worker-sub

# List all topics
gcloud pubsub topics list

# List subscriptions for a topic
gcloud pubsub topics list-subscriptions my-events

# Seek subscription to a timestamp (replay messages)
gcloud pubsub subscriptions seek my-worker-sub \
  --time=2024-01-15T10:00:00Z

# View dead-letter messages
gcloud pubsub subscriptions pull my-events-dlq --limit=10
```

## Best Practices

1. **Always set a dead-letter topic** — prevents poison messages from blocking the queue.
2. **Use `message.nack()` for retryable errors**, `message.ack()` after logging for non-retryable.
3. **Make handlers idempotent** — messages can be delivered more than once.
4. **Set `ack-deadline` longer than your processing time** — default 10s is often too short.
5. **Use message attributes for routing** — avoid decoding payload just to determine type.
6. **Monitor `oldest_unacked_message_age`** — growing backlog means consumers can't keep up.
