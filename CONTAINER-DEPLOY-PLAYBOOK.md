---
name: cloud-run-deploy
description: >
  Build Docker containers and deploy them to Google Cloud Run. Use when the user wants
  to deploy a web service, REST API, background worker, or containerized app to Cloud Run,
  set up continuous deployment from source, configure environment variables or secrets,
  set up traffic splitting, handle custom domains, or manage Cloud Run revisions.
  Triggers on "Cloud Run", "container deploy", "serverless container", "run deploy".
license: Apache-2.0
compatibility: Requires gcloud CLI, Docker, and a GCP project with Cloud Run API enabled
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: Cloud Run, Artifact Registry, Cloud Build, Secret Manager
---

# Cloud Run Deploy

## Prerequisites

```bash
# Enable required APIs
gcloud services enable run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com

# Configure Docker to use gcloud credentials
gcloud auth configure-docker us-central1-docker.pkg.dev
```

## Deploy from Source (Simplest)

Cloud Run can build and deploy directly from source using Buildpacks — no Dockerfile needed:

```bash
# Deploy from current directory (auto-detects language)
gcloud run deploy my-service \
  --source . \
  --region us-central1 \
  --allow-unauthenticated

# Deploy from a GitHub repo
gcloud run deploy my-service \
  --source https://github.com/user/repo \
  --region us-central1
```

## Full Docker Workflow

### Step 1 — Write a Dockerfile

```dockerfile
# Example: Node.js app
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
```

### Step 2 — Build & Push to Artifact Registry

```bash
PROJECT_ID=$(gcloud config get-value project)
REGION=us-central1
IMAGE=$REGION-docker.pkg.dev/$PROJECT_ID/my-repo/my-service:latest

# Create repository (once)
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=$REGION

# Build with Cloud Build (no local Docker required)
gcloud builds submit --tag $IMAGE

# OR build locally and push
docker build -t $IMAGE .
docker push $IMAGE
```

### Step 3 — Deploy to Cloud Run

```bash
gcloud run deploy my-service \
  --image $IMAGE \
  --region $REGION \
  --platform managed \
  --allow-unauthenticated \
  --port 8080 \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 0 \
  --max-instances 10 \
  --concurrency 80
```

## Environment Variables & Secrets

```bash
# Set environment variables at deploy time
gcloud run deploy my-service \
  --image $IMAGE \
  --region $REGION \
  --set-env-vars="NODE_ENV=production,LOG_LEVEL=info"

# Use Secret Manager for sensitive values
gcloud secrets create db-password --data-file=./password.txt

gcloud run deploy my-service \
  --image $IMAGE \
  --region $REGION \
  --set-secrets="DB_PASSWORD=db-password:latest"

# Update secrets without redeploying
gcloud run services update my-service \
  --region $REGION \
  --update-secrets="API_KEY=api-key:latest"
```

## Traffic Splitting (Canary / Blue-Green)

```bash
# Deploy a new revision without sending traffic
gcloud run deploy my-service \
  --image $IMAGE_V2 \
  --region $REGION \
  --no-traffic \
  --tag canary

# Send 10% traffic to canary
gcloud run services update-traffic my-service \
  --region $REGION \
  --to-tags canary=10

# Fully promote after validation
gcloud run services update-traffic my-service \
  --region $REGION \
  --to-latest

# Rollback to a specific revision
gcloud run services update-traffic my-service \
  --region $REGION \
  --to-revisions my-service-00001-abc=100
```

## Custom Domains

```bash
# Map a custom domain
gcloud run domain-mappings create \
  --service my-service \
  --domain api.example.com \
  --region $REGION

# Verify domain (follow DNS instructions printed by the above command)
gcloud run domain-mappings describe \
  --domain api.example.com \
  --region $REGION
```

## Service Account & IAM

```bash
# Create a dedicated service account
gcloud iam service-accounts create my-service-sa \
  --display-name="My Service Account"

# Grant it the roles it needs
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:my-service-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataViewer"

# Deploy using that service account
gcloud run deploy my-service \
  --image $IMAGE \
  --region $REGION \
  --service-account my-service-sa@$PROJECT_ID.iam.gserviceaccount.com
```

## Useful Commands

```bash
# View logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=my-service" \
  --limit 50 --format="table(timestamp, textPayload)"

# Describe service
gcloud run services describe my-service --region $REGION

# List all services
gcloud run services list --region $REGION

# Delete service
gcloud run services delete my-service --region $REGION
```

## Common Issues

| Problem | Fix |
|---------|-----|
| Container fails to start | Ensure app listens on `PORT` env var (Cloud Run injects this) |
| Cold start latency | Set `--min-instances 1` to keep a warm instance |
| Memory exceeded | Increase `--memory` (max 32Gi) |
| Request timeout | Set `--timeout` up to 3600s; consider async patterns for long jobs |
| 403 on public service | Add `--allow-unauthenticated` or grant invoker role |
