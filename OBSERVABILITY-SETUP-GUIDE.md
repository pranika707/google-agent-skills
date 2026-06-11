---
name: monitoring-alerting
description: >
  Set up Google Cloud Monitoring dashboards, custom metrics, uptime checks, log-based
  metrics, and alerting policies with notification channels. Use when the user wants to
  monitor GCP services, set up alerts for errors or latency, create dashboards, write
  Cloud Logging queries, set up uptime monitors, create SLOs, or investigate incidents.
  Triggers on "Cloud Monitoring", "Cloud Logging", "alerting", "dashboard", "uptime check",
  "log query", "metrics", "SLO", or "observability on GCP".
license: Apache-2.0
compatibility: Requires gcloud CLI; optionally Terraform for infrastructure-as-code monitoring
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: Cloud Monitoring, Cloud Logging, Error Reporting, Cloud Trace
---

# Cloud Monitoring & Alerting

## Cloud Logging — Query Reference

The query language is called **Logging Query Language (LQL)**. Use the **Logs Explorer** in the console or `gcloud logging read`.

### Common Queries

```
-- Cloud Run errors in last hour
resource.type="cloud_run_revision"
resource.labels.service_name="my-service"
severity>=ERROR
timestamp>="2024-01-15T10:00:00Z"

-- HTTP 5xx errors on Cloud Run
resource.type="cloud_run_revision"
httpRequest.status>=500

-- Specific log message contains
resource.type="cloud_run_revision"
textPayload=~"database connection failed"

-- Cloud Function execution errors
resource.type="cloud_function"
resource.labels.function_name="my-function"
severity=ERROR

-- BigQuery job failures
resource.type="bigquery_resource"
protoPayload.status.code=5  -- NOT_FOUND
protoPayload.methodName="jobservice.jobcompleted"

-- GKE pod crashes
resource.type="k8s_container"
resource.labels.cluster_name="my-cluster"
severity=ERROR
labels."k8s-pod/app"="my-app"

-- IAM permission denied
protoPayload.@type="type.googleapis.com/google.cloud.audit.AuditLog"
protoPayload.status.code=7
protoPayload.authorizationInfo.granted=false
```

### CLI Queries

```bash
# Read recent error logs
gcloud logging read \
  'resource.type="cloud_run_revision" AND severity>=ERROR' \
  --limit=50 \
  --format="table(timestamp, resource.labels.service_name, textPayload)" \
  --freshness=1h

# Stream logs in real-time
gcloud logging tail \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-service"'
```

## Uptime Checks

```bash
# Create HTTP uptime check
gcloud monitoring uptime-checks create http my-api-check \
  --display-name="My API Health Check" \
  --uri=https://api.example.com/health \
  --period=60 \
  --timeout=10 \
  --content-type=application/json

# List uptime checks
gcloud monitoring uptime-checks list
```

## Alerting Policies

### Alert on High Error Rate (via gcloud)

```bash
# Create notification channel (email)
gcloud alpha monitoring channels create \
  --display-name="Oncall Email" \
  --type=email \
  --channel-labels=email_address=oncall@example.com

# Get the channel ID
gcloud alpha monitoring channels list

# Create alerting policy for Cloud Run error rate
cat > alert-policy.json << 'EOF'
{
  "displayName": "Cloud Run High Error Rate",
  "conditions": [{
    "displayName": "Error rate > 5%",
    "conditionThreshold": {
      "filter": "resource.type=\"cloud_run_revision\" AND metric.type=\"run.googleapis.com/request_count\" AND metric.labels.response_code_class=\"5xx\"",
      "comparison": "COMPARISON_GT",
      "thresholdValue": 0.05,
      "duration": "60s",
      "aggregations": [{
        "alignmentPeriod": "60s",
        "perSeriesAligner": "ALIGN_RATE"
      }]
    }
  }],
  "notificationChannels": ["projects/PROJECT_ID/notificationChannels/CHANNEL_ID"],
  "alertStrategy": {
    "autoClose": "604800s"
  }
}
EOF

gcloud alpha monitoring policies create --policy-from-file=alert-policy.json
```

## Terraform — Monitoring as Code

### Uptime Check + Alert

```hcl
# Notification channel (email)
resource "google_monitoring_notification_channel" "email" {
  display_name = "Oncall Email"
  type         = "email"

  labels = {
    email_address = "oncall@example.com"
  }
}

# Uptime check
resource "google_monitoring_uptime_check_config" "api_health" {
  display_name = "API Health Check"
  timeout      = "10s"
  period       = "60s"

  http_check {
    path         = "/health"
    port         = 443
    use_ssl      = true
    validate_ssl = true
  }

  monitored_resource {
    type = "uptime_url"
    labels = {
      project_id = var.project_id
      host       = "api.example.com"
    }
  }
}

# Alert when uptime check fails
resource "google_monitoring_alert_policy" "uptime_failure" {
  display_name = "API Uptime Failure"
  combiner     = "OR"

  conditions {
    display_name = "Uptime check failed"
    condition_threshold {
      filter          = "metric.type=\"monitoring.googleapis.com/uptime_check/check_passed\" AND resource.type=\"uptime_url\" AND metric.labels.check_id=\"${google_monitoring_uptime_check_config.api_health.uptime_check_id}\""
      comparison      = "COMPARISON_LT"
      threshold_value = 1
      duration        = "60s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_NEXT_OLDER"
        cross_series_reducer = "REDUCE_COUNT_TRUE"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]

  alert_strategy {
    auto_close = "604800s"
  }
}
```

### Log-Based Alert (Error Count)

```hcl
# Create a log-based metric
resource "google_logging_metric" "error_count" {
  name        = "cloud_run_errors"
  description = "Count of ERROR logs from Cloud Run"

  filter = <<-EOF
    resource.type="cloud_run_revision"
    resource.labels.service_name="my-service"
    severity>=ERROR
  EOF

  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "INT64"
    unit        = "1"
  }
}

# Alert when errors exceed 10 per minute
resource "google_monitoring_alert_policy" "high_errors" {
  display_name = "High Cloud Run Error Rate"
  combiner     = "OR"

  conditions {
    display_name = "Error count > 10/min"

    condition_threshold {
      filter          = "metric.type=\"logging.googleapis.com/user/${google_logging_metric.error_count.name}\" AND resource.type=\"cloud_run_revision\""
      comparison      = "COMPARISON_GT"
      threshold_value = 10
      duration        = "60s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_RATE"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]
}
```

## Custom Dashboards

```bash
# Create a dashboard from JSON
cat > dashboard.json << 'EOF'
{
  "displayName": "My Service Dashboard",
  "gridLayout": {
    "columns": "2",
    "widgets": [
      {
        "title": "Request Count",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"run.googleapis.com/request_count\" AND resource.type=\"cloud_run_revision\""
              }
            }
          }]
        }
      },
      {
        "title": "Request Latency (p99)",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"run.googleapis.com/request_latencies\" AND resource.type=\"cloud_run_revision\"",
                "aggregation": {
                  "perSeriesAligner": "ALIGN_PERCENTILE_99"
                }
              }
            }
          }]
        }
      }
    ]
  }
}
EOF

gcloud monitoring dashboards create --config-from-file=dashboard.json
```

## SLO Setup

```bash
# Define an SLO: 99.9% of requests succeed within 200ms
gcloud alpha monitoring slos create \
  --service=my-cloud-run-service \
  --display-name="99.9% Availability SLO" \
  --request-based-sli-latency-threshold=200ms \
  --goal=0.999 \
  --rolling-period-days=30
```

## Quick Incident Investigation Checklist

1. **Check Error Reporting**: `console.cloud.google.com/errors` — auto-groups exceptions.
2. **Check Cloud Trace**: Look for high-latency spans to find slow operations.
3. **Query logs**: Use LQL in Logs Explorer with `severity>=ERROR` and narrow the time range.
4. **Check metrics**: Look at request count, latency p99, and memory usage in Monitoring.
5. **Check recent deployments**: `gcloud run revisions list` or `gcloud app versions list`.
