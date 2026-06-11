---
name: cloud-functions
description: >
  Write, deploy, and test Google Cloud Functions (1st and 2nd gen). Use when the user
  wants to create serverless functions triggered by HTTP, Pub/Sub, Cloud Storage events,
  Firestore events, or scheduled Cloud Scheduler jobs. Also use for Firebase Cloud
  Functions (they share the same runtime). Triggers on "Cloud Functions", "GCF",
  "serverless function", "HTTP trigger", "background function", "Cloud Scheduler",
  "functions deploy", or event-triggered compute on GCP.
license: Apache-2.0
compatibility: Requires gcloud CLI with Cloud Functions API enabled; Node.js 20 or Python 3.12 recommended
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: Cloud Functions, Cloud Scheduler, Pub/Sub, Cloud Storage, Firestore
---

# Google Cloud Functions

## Gen 2 vs Gen 1

**Always use Gen 2** for new functions. Gen 2 functions run on Cloud Run under the hood and offer:
- Longer timeouts (up to 60 minutes)
- More memory (up to 32 GiB)
- Concurrency support
- Better cold start performance

## HTTP Functions

### Node.js (Gen 2)

```javascript
// index.js
const { onRequest } = require("firebase-functions/v2/https");

exports.hello = onRequest(
  {
    region: "us-central1",
    memory: "256MiB",
    timeoutSeconds: 60,
    cors: true, // Enable CORS for browser clients
  },
  async (req, res) => {
    if (req.method !== "POST") {
      return res.status(405).send("Method Not Allowed");
    }

    const { name } = req.body;
    res.json({ message: `Hello, ${name || "World"}!` });
  }
);
```

### Python (Gen 2)

```python
# main.py
import functions_framework
import json

@functions_framework.http
def hello(request):
    """HTTP Cloud Function."""
    if request.method == "OPTIONS":
        # Handle CORS preflight
        headers = {
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Methods": "POST",
            "Access-Control-Allow-Headers": "Content-Type",
        }
        return ("", 204, headers)

    data = request.get_json(silent=True) or {}
    name = data.get("name", "World")

    headers = {"Access-Control-Allow-Origin": "*"}
    return (json.dumps({"message": f"Hello, {name}!"}), 200, headers)
```

## Pub/Sub Triggered Functions

```javascript
// index.js
const { onMessagePublished } = require("firebase-functions/v2/pubsub");

exports.processPubSub = onMessagePublished(
  { topic: "my-events", region: "us-central1" },
  async (event) => {
    const data = event.data.message.json; // auto-decoded JSON
    const attributes = event.data.message.attributes;

    console.log("Received:", data, attributes);

    // Process the message
    await handleEvent(data, attributes);
    // Return without throwing = ACK
  }
);
```

```python
# main.py
import functions_framework
import base64, json

@functions_framework.cloud_event
def process_pubsub(cloud_event):
    """Pub/Sub triggered function."""
    message_data = base64.b64decode(cloud_event.data["message"]["data"])
    data = json.loads(message_data)
    attributes = cloud_event.data["message"].get("attributes", {})

    print(f"Processing: {data}")
    handle_event(data, attributes)
```

## Cloud Storage Triggered Functions

```javascript
// Runs when a file is uploaded to a bucket
const { onObjectFinalized } = require("firebase-functions/v2/storage");
const { Storage } = require("@google-cloud/storage");

exports.processUpload = onObjectFinalized(
  { bucket: "my-uploads-bucket" },
  async (event) => {
    const { name, contentType, size } = event.data;
    console.log(`New file: ${name} (${contentType}, ${size} bytes)`);

    // Example: resize image, parse CSV, trigger pipeline...
  }
);
```

## Scheduled Functions (Cloud Scheduler)

```javascript
// Runs every day at midnight UTC
const { onSchedule } = require("firebase-functions/v2/scheduler");

exports.dailyCleanup = onSchedule(
  {
    schedule: "0 0 * * *",   // cron syntax
    timeZone: "UTC",
    region: "us-central1",
    memory: "512MiB",
  },
  async (event) => {
    console.log("Running daily cleanup...");
    await deleteOldRecords();
    await generateDailySummary();
  }
);
```

## Firestore Triggers

```javascript
const { onDocumentCreated, onDocumentUpdated } = require("firebase-functions/v2/firestore");

// Triggered when a new order is created
exports.onOrderCreated = onDocumentCreated("orders/{orderId}", async (event) => {
  const order = event.data.data();
  const orderId = event.params.orderId;

  await sendOrderConfirmationEmail(order.customerEmail, orderId);
  await updateInventory(order.items);
});

// Triggered when a user document changes
exports.onUserUpdated = onDocumentUpdated("users/{userId}", async (event) => {
  const before = event.data.before.data();
  const after = event.data.after.data();

  if (before.email !== after.email) {
    await syncEmailToMailingList(before.email, after.email);
  }
});
```

## Deployment

```bash
# Gen 2 HTTP function (Node.js)
gcloud functions deploy hello \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=. \
  --entry-point=hello \
  --trigger-http \
  --allow-unauthenticated \
  --memory=256MB \
  --timeout=60s \
  --set-env-vars="NODE_ENV=production"

# Gen 2 Pub/Sub function (Python)
gcloud functions deploy process_pubsub \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=process_pubsub \
  --trigger-topic=my-events \
  --memory=512MB

# Gen 2 Scheduled function
gcloud functions deploy daily_cleanup \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=. \
  --entry-point=dailyCleanup \
  --trigger-http \
  --schedule="0 0 * * *" \
  --time-zone="UTC"

# Set secrets
gcloud functions deploy my-function \
  --set-secrets="API_KEY=projects/PROJECT_ID/secrets/api-key:latest"
```

## Local Development & Testing

```bash
# Install Functions Framework
npm install @google-cloud/functions-framework  # Node.js
pip install functions-framework                # Python

# Run locally
npx functions-framework --target=hello --port=8080
functions-framework --target=hello --port=8080  # Python

# Test HTTP function
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice"}'

# Simulate Pub/Sub event
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{
    "message": {
      "data": "'$(echo -n '"{"userId":"123"}"' | base64)'",
      "attributes": {"type": "user.signup"}
    }
  }'
```

## `package.json` (Node.js)

```json
{
  "name": "my-functions",
  "version": "1.0.0",
  "main": "index.js",
  "engines": { "node": "20" },
  "dependencies": {
    "@google-cloud/bigquery": "^7.0.0",
    "firebase-functions": "^6.0.0"
  }
}
```

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Function failed to start` | Uncaught top-level error | Wrap initializations in try/catch |
| `Error: deadline exceeded` | Function ran over timeout | Increase `--timeout` or refactor to async |
| `PERMISSION_DENIED` | Missing IAM role | Grant roles to the function's service account |
| Memory limit exceeded | Processing too much data | Increase `--memory` or stream data |
