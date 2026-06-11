---
name: terraform-gcp
description: >
  Provision and manage Google Cloud Platform infrastructure using Terraform.
  Use when the user wants to write Terraform for GCP resources, manage terraform state,
  create reusable modules, set up remote state in GCS, plan or apply infra changes,
  use the google or google-beta provider, import existing GCP resources, or manage
  environments (dev/staging/prod) with Terraform workspaces. Triggers on "Terraform",
  "terraform apply", ".tf files", "IaC", "infrastructure as code on GCP".
license: Apache-2.0
compatibility: Requires Terraform 1.5+, gcloud CLI authenticated
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: All GCP — Terraform manages resources across all services
---

# Terraform for GCP

## Project Bootstrap

### Provider & Remote State

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }

  # Remote state in GCS (create the bucket manually first)
  backend "gcs" {
    bucket = "my-project-tfstate"
    prefix = "terraform/state"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

```hcl
# variables.tf
variable "project_id" {
  type        = string
  description = "GCP Project ID"
}

variable "region" {
  type    = string
  default = "us-central1"
}

variable "env" {
  type    = string
  default = "dev"
}
```

```bash
# Create the state bucket (one-time, before terraform init)
gcloud storage buckets create gs://my-project-tfstate \
  --location=US \
  --uniform-bucket-level-access

gcloud storage buckets update gs://my-project-tfstate \
  --versioning
```

## Common GCP Resources

### Cloud Run Service

```hcl
resource "google_cloud_run_v2_service" "app" {
  name     = "my-app-${var.env}"
  location = var.region

  template {
    containers {
      image = "us-central1-docker.pkg.dev/${var.project_id}/my-repo/my-app:latest"

      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
      }

      env {
        name  = "NODE_ENV"
        value = var.env
      }

      env {
        name = "DB_PASSWORD"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.db_password.secret_id
            version = "latest"
          }
        }
      }
    }

    scaling {
      min_instance_count = var.env == "prod" ? 1 : 0
      max_instance_count = 10
    }
  }
}

# Allow public access
resource "google_cloud_run_v2_service_iam_member" "public" {
  name     = google_cloud_run_v2_service.app.name
  location = var.region
  role     = "roles/run.invoker"
  member   = "allUsers"
}

output "url" {
  value = google_cloud_run_v2_service.app.uri
}
```

### GCS Bucket

```hcl
resource "google_storage_bucket" "assets" {
  name          = "${var.project_id}-assets-${var.env}"
  location      = "US"
  force_destroy = var.env != "prod"

  uniform_bucket_level_access = true

  versioning {
    enabled = var.env == "prod"
  }

  lifecycle_rule {
    condition {
      age = 90
    }
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }
}
```

### BigQuery Dataset & Table

```hcl
resource "google_bigquery_dataset" "analytics" {
  dataset_id  = "analytics_${var.env}"
  location    = "US"
  description = "Analytics dataset for ${var.env}"

  default_partition_expiration_ms = var.env != "prod" ? 7776000000 : null  # 90 days for non-prod
}

resource "google_bigquery_table" "events" {
  dataset_id = google_bigquery_dataset.analytics.dataset_id
  table_id   = "events"
  deletion_protection = var.env == "prod"

  time_partitioning {
    type  = "DAY"
    field = "event_date"
  }

  clustering = ["user_id", "event_type"]

  schema = file("${path.module}/schemas/events.json")
}
```

### Cloud SQL (Postgres)

```hcl
resource "google_sql_database_instance" "main" {
  name             = "postgres-${var.env}"
  database_version = "POSTGRES_15"
  region           = var.region

  settings {
    tier              = var.env == "prod" ? "db-custom-2-7680" : "db-f1-micro"
    availability_type = var.env == "prod" ? "REGIONAL" : "ZONAL"

    backup_configuration {
      enabled    = var.env == "prod"
      start_time = "03:00"
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.vpc.self_link
    }
  }

  deletion_protection = var.env == "prod"
}

resource "google_sql_database" "app_db" {
  name     = "appdb"
  instance = google_sql_database_instance.main.name
}
```

### Service Account with Roles

```hcl
resource "google_service_account" "app_sa" {
  account_id   = "my-app-sa-${var.env}"
  display_name = "My App Service Account (${var.env})"
}

resource "google_project_iam_member" "app_sa_bq" {
  project = var.project_id
  role    = "roles/bigquery.dataViewer"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

resource "google_project_iam_member" "app_sa_gcs" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}
```

## Modules Structure

```
modules/
├── cloud-run/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── gcs-bucket/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
environments/
├── dev/
│   ├── main.tf        # calls modules
│   ├── terraform.tfvars
│   └── backend.tf
├── staging/
│   └── ...
└── prod/
    └── ...
```

## Workflow Commands

```bash
# Initialize (downloads providers, configures backend)
terraform init

# Format code
terraform fmt -recursive

# Validate syntax
terraform validate

# Preview changes
terraform plan -var-file=terraform.tfvars

# Apply (add -auto-approve for CI)
terraform apply -var-file=terraform.tfvars

# Destroy (use with care!)
terraform destroy -var-file=terraform.tfvars -target=google_storage_bucket.temp

# Import existing resource
terraform import google_storage_bucket.assets my-project-assets-dev

# Show current state
terraform show
terraform state list
terraform state show google_cloud_run_v2_service.app
```

## Best Practices

1. **Lock provider versions** — Use `~> 5.0` not `>= 5.0` in `required_providers`.
2. **Use remote state** — Always store state in GCS with versioning enabled.
3. **Separate environments** — Use separate state files per environment, not workspaces for production.
4. **Protect production resources** — Set `deletion_protection = true` on critical resources.
5. **Tag everything** — Use `labels` on all resources for cost attribution.
6. **Use `prevent_destroy` lifecycle** — On databases, buckets, and other stateful resources.

```hcl
resource "google_sql_database_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```
