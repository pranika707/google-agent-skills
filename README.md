#  Google Cloud Agent Skills

A collection of [Agent Skills](https://agentskills.io) for Google Cloud Platform (GCP), Firebase, BigQuery, and related Google technologies — ready to use with any skills-compatible AI agent (Claude Code, GitHub Copilot, Cursor, Gemini CLI, VS Code, and more).

---

##  Skills Included

| Skill | Description |
|-------|-------------|
| [`gcloud-deploy`](./gcloud-deploy/) | Deploy and manage GCP resources via `gcloud` CLI |
| [`bigquery-analyst`](./bigquery-analyst/) | Write, run, and optimize BigQuery SQL queries |
| [`cloud-run-deploy`](./cloud-run-deploy/) | Build and deploy containerized apps to Cloud Run |
| [`firebase-deploy`](./firebase-deploy/) | Deploy Firebase Hosting, Functions, Firestore rules |
| [`gke-kubectl`](./gke-kubectl/) | Manage Google Kubernetes Engine clusters with `kubectl` |
| [`iam-security`](./iam-security/) | Audit and manage GCP IAM roles and permissions |
| [`terraform-gcp`](./terraform-gcp/) | Provision GCP infrastructure with Terraform |
| [`pubsub-workflows`](./pubsub-workflows/) | Design and manage Pub/Sub topics, subscriptions, and event flows |
| [`cloud-functions`](./cloud-functions/) | Write, deploy, and test Google Cloud Functions (Gen 1 \u0026 2) |
| [`monitoring-alerting`](./monitoring-alerting/) | Set up Cloud Monitoring dashboards, metrics, and alerts |

---

## 🛠 How to Use

### With Claude Code
```bash
claude skills add https://github.com/YOUR_USERNAME/google-agent-skills
```

### With Cursor
Add this repo's path to your Cursor settings under **Skills Directories**.

### With GitHub Copilot / VS Code
Reference individual skill folders in your `.github/copilot-instructions.md` or workspace settings.

### With Gemini CLI
```bash
gemini skills add ./google-agent-skills/bigquery-analyst
```

### Manual / Any Agent
Point your agent at any `SKILL.md` file. The agent will load the instructions when a relevant task is detected.

---

## 📋 Prerequisites

Depending on the skill you're using, you may need:

- [`gcloud` CLI](https://cloud.google.com/sdk/docs/install) — authenticated with `gcloud auth login`
- [`terraform`](https://developer.hashicorp.com/terraform/install) — v1.5+
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/) — configured for your GKE cluster
- [`firebase-tools`](https://firebase.google.com/docs/cli) — `npm install -g firebase-tools`
- [`docker`](https://docs.docker.com/get-docker/) — for Cloud Run container builds
- A GCP project with billing enabled

---

##  Contributing

Contributions welcome! To add a new skill:

1. Create a folder: `your-skill-name/`
2. Add a `SKILL.md` with valid frontmatter (`name`, `description`)
3. Optionally add `scripts/`, `references/`, and `assets/`
4. Open a pull request

See the [Agent Skills specification](https://agentskills.io/specification) for format details.

---


