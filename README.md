# Cloud Infrastructure Plugin for Claude Code

Manage all your cloud infrastructure from within Claude Code. Supports AWS, Terraform/OpenTofu,
Kubernetes, GCP, Azure, Vercel, Supabase, Render, Railway, and Upstash.

## What it does

- **Inspect** — list resources, check health, view running services across any platform
- **Deploy** — push changes to app platforms or apply IaC with a safe plan-first workflow
- **Debug** — tail logs, diagnose crashes, explain errors in plain language
- **Configure** — manage env vars, secrets, and access controls

## Supported platforms

| Platform | Capabilities |
|---|---|
| AWS | EC2, ECS, Lambda, S3, RDS, CloudWatch, IAM |
| Terraform / OpenTofu | init, validate, plan, apply, state, workspaces |
| Kubernetes | pods, deployments, Helm, logs, exec, scaling |
| GCP | Cloud Run, GKE, Cloud Functions, GCS |
| Azure | App Service, AKS, Container Apps, Azure Functions |
| Vercel | deployments, logs, env vars, domains |
| Render | services, deploys, env vars (via API) |
| Railway | services, deploys, logs, env vars, DB shells |
| Supabase | migrations, Edge Functions, local dev, type gen |
| Upstash | Redis REST API, database stats, key management |

## Slash commands

| Command | Description |
|---|---|
| `/infra-status` | Show status of cloud resources |
| `/infra-deploy` | Deploy changes to any platform |
| `/infra-logs` | View and tail service logs |
| `/infra-plan` | Run Terraform/OpenTofu plan safely |
| `/infra-migrate` | Apply database migrations |

## Skill (auto-triggered)

The `infra` skill triggers automatically when you describe infrastructure tasks in natural
language — no slash command needed. Examples:

- _"My ECS service is crashing, what's wrong?"_
- _"Apply my Terraform changes to staging"_
- _"What pods are running in the payments namespace?"_
- _"Deploy the frontend to Vercel production"_
- _"Run the Supabase migrations and regenerate my types"_

## Installation

```bash
claude plugin install https://github.com/thangnc/cloud-infrastructure
```

Or install locally:

```bash
claude plugin install /path/to/cloud-infrastructure
```

## Prerequisites

Install the CLIs for the platforms you use:

```bash
# AWS
brew install awscli

# Terraform / OpenTofu
brew install terraform
brew install opentofu

# Kubernetes
brew install kubectl
brew install helm

# GCP
brew install --cask google-cloud-sdk

# Azure
brew install azure-cli

# Vercel
npm install -g vercel

# Railway
npm install -g @railway/cli

# Supabase
brew install supabase/tap/supabase
```

Render and Upstash are managed via REST API — no CLI needed, just your API keys.

## License

MIT
