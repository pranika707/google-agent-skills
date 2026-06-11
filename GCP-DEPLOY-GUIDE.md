---
name: gcloud-deploy
description: >
  Deploy and manage Google Cloud Platform resources using the gcloud CLI.
  Use when the user wants to deploy apps to App Engine, manage Compute Engine VMs,
  configure VPCs, set up Cloud SQL, manage GCS buckets, switch projects, or run
  any gcloud command. Also triggers for "GCP", "Google Cloud", "App Engine", "gcloud auth",
  or "cloud project" tasks.
license: Apache-2.0
compatibility: Requires gcloud CLI installed and authenticated via `gcloud auth login`
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: App Engine, Compute Engine, Cloud SQL, GCS, VPC, IAM
---

# gcloud Deploy & Manage

## Prerequisites

Before running any commands, verify authentication and project:

```bash
gcloud auth list                    # Check active account
gcloud config get-value project     # Check active project
gcloud config set project PROJECT_ID  # Switch project if needed
```

## Common Workflows

### 1. Deploy to App Engine

```bash
# Standard deploy from app.yaml
gcloud app deploy

# Deploy specific version, no traffic promotion
gcloud app deploy --version=v2 --no-promote

# List versions
gcloud app versions list

# Migrate traffic to new version
gcloud app services set-traffic default --splits=v2=1
```

### 2. Manage Compute Engine VMs

```bash
# Create a VM
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud

# SSH into VM
gcloud compute ssh my-vm --zone=us-central1-a

# List VMs
gcloud compute instances list

# Stop/Start/Delete
gcloud compute instances stop my-vm --zone=us-central1-a
gcloud compute instances start my-vm --zone=us-central1-a
gcloud compute instances delete my-vm --zone=us-central1-a
```

### 3. Manage Cloud Storage Buckets

```bash
# Create bucket
gcloud storage buckets create gs://my-bucket --location=us-central1

# Upload files
gcloud storage cp ./local-file.txt gs://my-bucket/

# Sync folder
gcloud storage rsync ./local-dir gs://my-bucket/dir --recursive

# List objects
gcloud storage ls gs://my-bucket/

# Make bucket public (use with caution)
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=allUsers --role=roles/storage.objectViewer
```

### 4. Cloud SQL

```bash
# Create a Postgres instance
gcloud sql instances create my-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1

# Create database
gcloud sql databases create mydb --instance=my-db

# Connect
gcloud sql connect my-db --user=postgres
```

### 5. Project & Config Management

```bash
# List projects
gcloud projects list

# Create project
gcloud projects create my-new-project --name="My Project"

# Set quota project for billing
gcloud billing projects link my-new-project \
  --billing-account=BILLING_ACCOUNT_ID

# Enable APIs
gcloud services enable run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com
```

## Environment Setup

```bash
# Set default region/zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Create named configuration
gcloud config configurations create staging
gcloud config set project staging-project-id
gcloud config set account staging@example.com
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `PERMISSION_DENIED` | Run `gcloud auth login` or check IAM roles |
| `API not enabled` | Run `gcloud services enable SERVICE_NAME` |
| Wrong project | Run `gcloud config set project PROJECT_ID` |
| Quota exceeded | Check quotas at console.cloud.google.com/iam-admin/quotas |

## Key Flags

- `--project=PROJECT_ID` — override active project for one command
- `--format=json` — output as JSON (useful for scripting)
- `--quiet` / `-q` — skip confirmation prompts
- `--verbosity=debug` — show full request/response for debugging
