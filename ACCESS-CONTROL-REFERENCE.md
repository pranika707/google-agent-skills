---
name: iam-security
description: >
  Audit, configure, and manage Google Cloud IAM (Identity and Access Management) roles,
  policies, service accounts, and permissions. Use when the user wants to grant or revoke
  access, create service accounts, audit who has access to a project, set up least-privilege
  permissions, manage organization policies, review IAM bindings, or troubleshoot
  PERMISSION_DENIED errors. Triggers on "IAM", "permissions", "roles", "service account",
  "access control", "PERMISSION_DENIED", "principal", "policy binding".
license: Apache-2.0
compatibility: Requires gcloud CLI with project owner or Security Admin role
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: IAM, Resource Manager, Security Command Center, Service Accounts
---

# IAM & Security

## Viewing Current Permissions

```bash
# Get all IAM bindings on a project
gcloud projects get-iam-policy PROJECT_ID

# Get IAM policy as JSON (easier to parse)
gcloud projects get-iam-policy PROJECT_ID --format=json

# Filter for a specific principal
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --format="table(bindings.role)" \
  --filter="bindings.members:user:alice@example.com"

# List all service accounts
gcloud iam service-accounts list

# Get roles for a specific service account
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --format="table(bindings.role)" \
  --filter="bindings.members:serviceAccount:my-sa@PROJECT_ID.iam.gserviceaccount.com"
```

## Granting & Revoking Roles

```bash
# Grant a role to a user
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:alice@example.com" \
  --role="roles/bigquery.dataViewer"

# Grant to a group
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="group:engineers@example.com" \
  --role="roles/container.developer"

# Grant to a service account
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:my-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# Revoke a role
gcloud projects remove-iam-policy-binding PROJECT_ID \
  --member="user:alice@example.com" \
  --role="roles/bigquery.dataViewer"
```

## Service Accounts

```bash
# Create a service account
gcloud iam service-accounts create my-service-sa \
  --display-name="My Service Account" \
  --description="SA for my-service Cloud Run app"

# Grant roles to the SA
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:my-service-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/datastore.user"

# Create and download a key (avoid when possible — prefer Workload Identity)
gcloud iam service-accounts keys create key.json \
  --iam-account=my-service-sa@PROJECT_ID.iam.gserviceaccount.com

# List existing keys
gcloud iam service-accounts keys list \
  --iam-account=my-service-sa@PROJECT_ID.iam.gserviceaccount.com

# Delete a key (rotate regularly!)
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=my-service-sa@PROJECT_ID.iam.gserviceaccount.com

# Disable a service account
gcloud iam service-accounts disable \
  my-service-sa@PROJECT_ID.iam.gserviceaccount.com
```

## Custom Roles

```bash
# List predefined roles matching a keyword
gcloud iam roles list --filter="title:BigQuery"

# Get details of a predefined role
gcloud iam roles describe roles/bigquery.dataViewer

# Create a custom role from YAML
cat > custom-role.yaml << 'EOF'
title: "Custom Read-Only BigQuery Role"
description: "Can list and read BQ resources but not export"
stage: GA
includedPermissions:
- bigquery.datasets.get
- bigquery.tables.get
- bigquery.tables.list
- bigquery.jobs.create
EOF

gcloud iam roles create customBQReader \
  --project=PROJECT_ID \
  --file=custom-role.yaml

# Update custom role
gcloud iam roles update customBQReader \
  --project=PROJECT_ID \
  --add-permissions=bigquery.tables.getData
```

## Least Privilege Audit

Use this pattern to audit over-permissioned principals:

```bash
# Find everyone with Editor or Owner role (high risk)
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --format="table(bindings.role, bindings.members)" \
  --filter="bindings.role:(roles/editor OR roles/owner)"

# Find all service accounts with owner role
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --format="table(bindings.members)" \
  --filter="bindings.role=roles/owner AND bindings.members:serviceAccount"
```

## Organization Policies

```bash
# List org policies on a project
gcloud resource-manager org-policies list --project=PROJECT_ID

# View a specific policy
gcloud resource-manager org-policies describe \
  constraints/compute.requireShieldedVm \
  --project=PROJECT_ID

# Enforce a constraint (org-level)
gcloud resource-manager org-policies enable-enforce \
  constraints/iam.disableServiceAccountKeyCreation \
  --organization=ORG_ID
```

## Impersonation & Short-Lived Credentials

```bash
# Impersonate a service account (requires roles/iam.serviceAccountTokenCreator)
gcloud config set auth/impersonate_service_account \
  my-sa@PROJECT_ID.iam.gserviceaccount.com

# Generate a short-lived token
gcloud auth print-access-token \
  --impersonate-service-account=my-sa@PROJECT_ID.iam.gserviceaccount.com

# Stop impersonating
gcloud config unset auth/impersonate_service_account
```

## Common Permission Roles Reference

| Use Case | Recommended Role |
|----------|-----------------|
| View GCS buckets | `roles/storage.objectViewer` |
| Deploy to Cloud Run | `roles/run.admin` |
| Invoke Cloud Run | `roles/run.invoker` |
| Query BigQuery | `roles/bigquery.dataViewer` + `roles/bigquery.jobUser` |
| Deploy to GKE | `roles/container.developer` |
| Read Secret Manager | `roles/secretmanager.secretAccessor` |
| Full project admin | `roles/owner` (use sparingly!) |

## Security Best Practices

1. **Avoid service account keys** — Use Workload Identity or short-lived tokens instead.
2. **Rotate keys regularly** — Audit with `gcloud iam service-accounts keys list`.
3. **Avoid Editor/Owner roles** — Grant only the specific roles needed.
4. **Enable VPC Service Controls** for sensitive data.
5. **Use groups over individual users** — Manage access at group level.
6. **Audit with Cloud Audit Logs** — All IAM changes are logged automatically.
