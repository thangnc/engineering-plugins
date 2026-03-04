# Modern Platform Stack — Vercel, Railway, Render, Supabase, Upstash

This reference covers Terraform providers for the modern developer platform stack. These platforms are commonly combined to build full-stack applications: Vercel/Render for frontend/backend hosting, Railway for backend services, Supabase for Postgres + auth + storage, and Upstash for serverless Redis/Kafka/QStash.

## Table of Contents
1. Provider Overview and Maturity
2. Unified Provider Configuration
3. Vercel Provider
4. Railway Provider
5. Render Provider
6. Supabase Provider
7. Upstash Provider
8. Full-Stack Composition Patterns
9. Environment Variable Wiring
10. Secrets Management for Platform Tokens

---

## 1. Provider Overview and Maturity

| Provider | Source | Status | Auth Method |
|----------|--------|--------|-------------|
| Vercel | `vercel/vercel` | Official, stable | API token (`VERCEL_API_TOKEN`) |
| Railway | `terraform-community-providers/railway` | Community, active | API token (`RAILWAY_TOKEN`) |
| Render | `render-oss/render` | Official, early access | API key + owner ID (`RENDER_API_KEY`) |
| Supabase | `supabase/supabase` | Official, stable | Access token (`SUPABASE_ACCESS_TOKEN`) |
| Upstash | `upstash/upstash` | Official, stable | Email + API key (`UPSTASH_EMAIL`, `UPSTASH_API_KEY`) |

**Important notes:**
- The Railway provider is community-maintained, not official from Railway. It's actively developed and production-usable but may lag behind Railway's API.
- The Render provider is in early access — expect possible breaking changes between versions. Pin versions tightly.
- All providers authenticate via API tokens. Never hardcode tokens in `.tf` files — use environment variables or a secrets manager.

---

## 2. Unified Provider Configuration

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    vercel = {
      source  = "vercel/vercel"
      version = "~> 2.0"
    }
    railway = {
      source  = "terraform-community-providers/railway"
      version = "~> 0.4"
    }
    render = {
      source  = "render-oss/render"
      version = "~> 1.0"
    }
    supabase = {
      source  = "supabase/supabase"
      version = "~> 1.0"
    }
    upstash = {
      source  = "upstash/upstash"
      version = "~> 1.0"
    }
  }
}
```

```hcl
# providers.tf
# All tokens should come from environment variables, not hardcoded values.
# Set: VERCEL_API_TOKEN, RAILWAY_TOKEN, RENDER_API_KEY, RENDER_OWNER_ID,
#       SUPABASE_ACCESS_TOKEN, UPSTASH_EMAIL, UPSTASH_API_KEY

provider "vercel" {
  # Reads VERCEL_API_TOKEN from environment
  # Optional: team = var.vercel_team_id
}

provider "railway" {
  # Reads RAILWAY_TOKEN from environment
}

provider "render" {
  # Reads RENDER_API_KEY and RENDER_OWNER_ID from environment
}

provider "supabase" {
  # Reads SUPABASE_ACCESS_TOKEN from environment
}

provider "upstash" {
  email   = var.upstash_email
  api_key = var.upstash_api_key
}
```

```hcl
# variables.tf — sensitive provider credentials passed via tfvars or env
variable "upstash_email" {
  description = "Email registered with Upstash"
  type        = string
  sensitive   = true
}

variable "upstash_api_key" {
  description = "Upstash API key from console"
  type        = string
  sensitive   = true
}

variable "vercel_team_id" {
  description = "Vercel team ID (optional, for team-scoped resources)"
  type        = string
  default     = null
}
```

---

## 3. Vercel Provider

### Key Resources

**vercel_project** — The core resource. Creates a Vercel project linked to a Git repository.

```hcl
resource "vercel_project" "frontend" {
  name      = "${var.project_name}-${var.environment}"
  framework = "nextjs"

  team_id = var.vercel_team_id

  git_repository = {
    type = "github"
    repo = "myorg/my-frontend"
  }

  environment = [
    {
      key    = "NEXT_PUBLIC_SUPABASE_URL"
      value  = supabase_project.main.endpoint
      target = ["production", "preview"]
    },
    {
      key    = "NEXT_PUBLIC_SUPABASE_ANON_KEY"
      value  = data.supabase_apikeys.main.anon_key
      target = ["production", "preview"]
    },
    {
      key    = "UPSTASH_REDIS_REST_URL"
      value  = upstash_redis_database.cache.endpoint
      target = ["production"]
    },
    {
      key       = "UPSTASH_REDIS_REST_TOKEN"
      value     = upstash_redis_database.cache.rest_token
      target    = ["production"]
      sensitive = true
    },
  ]
}
```

**vercel_project_domain** — Custom domain assignment.

```hcl
resource "vercel_project_domain" "primary" {
  project_id = vercel_project.frontend.id
  team_id    = var.vercel_team_id
  domain     = var.environment == "prod" ? "app.example.com" : "${var.environment}.app.example.com"
}
```

**vercel_deployment** — Trigger a deployment from Terraform (useful for non-Git workflows).

```hcl
resource "vercel_deployment" "main" {
  project_id = vercel_project.frontend.id
  team_id    = var.vercel_team_id

  ref = var.git_ref  # Branch or commit SHA

  environment = {
    DEPLOY_ENV = var.environment
  }
}
```

**vercel_firewall_config** — Manage WAF rules via IaC.

```hcl
resource "vercel_firewall_config" "main" {
  project_id = vercel_project.frontend.id
  team_id    = var.vercel_team_id

  rules {
    rule {
      name      = "block-ai-bots"
      active    = true
      action    = { type = "deny" }
      condition_group = [{
        conditions = [{
          type = "user_agent"
          op   = "re"
          value = "GPTBot|ClaudeBot|ChatGPT"
        }]
      }]
    }
  }
}
```

### Vercel Tips
- Always specify `team_id` when working with team-scoped resources
- Use `environment` blocks on `vercel_project` to wire secrets from other providers
- The `framework` field enables automatic build settings (nextjs, sveltekit, nuxtjs, etc.)
- For monorepos, set `root_directory` on the project resource

---

## 4. Railway Provider

### Key Resources

**railway_project** — A Railway project is the top-level container.

```hcl
resource "railway_project" "backend" {
  name = "${var.project_name}-${var.environment}"
}
```

**railway_service** — A deployable service within a project.

```hcl
resource "railway_service" "api" {
  project_id = railway_project.backend.id
  name       = "api-server"
}
```

**railway_variable** — Environment variables for a service.

```hcl
resource "railway_variable" "database_url" {
  environment_id = railway_project.backend.default_environment.id
  service_id     = railway_service.api.id
  name           = "DATABASE_URL"
  value          = "postgresql://postgres:${var.supabase_db_password}@db.${supabase_project.main.id}.supabase.co:5432/postgres"
}

resource "railway_variable" "redis_url" {
  environment_id = railway_project.backend.default_environment.id
  service_id     = railway_service.api.id
  name           = "REDIS_URL"
  value          = "rediss://default:${upstash_redis_database.cache.password}@${upstash_redis_database.cache.endpoint}:6379"
}
```

**railway_custom_domain** — Custom domain for a service.

```hcl
resource "railway_custom_domain" "api" {
  environment_id = railway_project.backend.default_environment.id
  service_id     = railway_service.api.id
  domain         = "api.example.com"
}
```

### Railway Tips
- The community provider covers projects, services, variables, custom domains, TCP proxies, and volumes
- Railway handles deployments via Git push — Terraform manages the infrastructure, not the deploy trigger
- Use `railway_variable` to wire connections between Supabase, Upstash, and your services
- For private networking between Railway services, services within the same project can communicate via internal DNS

---

## 5. Render Provider

### Key Resources

**render_web_service** — Deploy a web service.

```hcl
resource "render_web_service" "api" {
  name    = "${var.project_name}-api-${var.environment}"
  plan    = var.environment == "prod" ? "standard" : "free"
  region  = "oregon"
  runtime = "node"

  start_command = "npm start"
  build_command = "npm install && npm run build"

  repo_url = "https://github.com/myorg/my-api"
  branch   = var.environment == "prod" ? "main" : "develop"

  env_vars = {
    "DATABASE_URL" = {
      value = "postgresql://postgres:${var.supabase_db_password}@db.${supabase_project.main.id}.supabase.co:5432/postgres"
    }
    "REDIS_URL" = {
      value = upstash_redis_database.cache.endpoint
    }
    "NODE_ENV" = {
      value = var.environment == "prod" ? "production" : "development"
    }
  }
}
```

**render_postgres** — Managed Postgres (alternative to Supabase for simpler use cases).

```hcl
resource "render_postgres" "db" {
  name    = "${var.project_name}-db-${var.environment}"
  plan    = var.environment == "prod" ? "standard" : "free"
  region  = "oregon"
  version = "16"
}
```

**render_static_site** — Static site hosting.

```hcl
resource "render_static_site" "docs" {
  name     = "${var.project_name}-docs"
  repo_url = "https://github.com/myorg/docs"
  branch   = "main"

  build_command   = "npm run build"
  publish_path    = "dist"
}
```

### Render Tips
- The provider is in early access — pin versions and expect API changes
- Render auto-deploys on Git push by default; Terraform manages service configuration
- Use `render_custom_domain` for domain mapping
- Render's free tier is useful for dev/staging environments

---

## 6. Supabase Provider

### Key Resources

**supabase_project** — Create or import a Supabase project.

```hcl
resource "supabase_project" "main" {
  organization_id   = var.supabase_org_id
  name              = "${var.project_name}-${var.environment}"
  database_password  = var.supabase_db_password
  region            = "ap-southeast-1"
}
```

**supabase_settings** — Configure project API settings.

```hcl
resource "supabase_settings" "main" {
  project_ref = supabase_project.main.id

  api = jsonencode({
    db_schema             = "public,storage,graphql_public"
    db_extra_search_path  = "public,extensions"
    max_rows              = 1000
  })
}
```

**supabase_branch** — Database branching for preview environments.

```hcl
resource "supabase_branch" "preview" {
  count = var.environment == "preview" ? 1 : 0

  project_ref   = supabase_project.main.id
  git_branch    = var.git_branch
  region        = "ap-southeast-1"
}
```

**data.supabase_apikeys** — Retrieve project API keys for wiring to other services.

```hcl
data "supabase_apikeys" "main" {
  project_ref = supabase_project.main.id
}

# Use in other resources:
# data.supabase_apikeys.main.anon_key
# data.supabase_apikeys.main.service_role_key
```

**Importing an existing project:**
```hcl
import {
  to = supabase_project.main
  id = "your-project-ref-id"
}
```

### Supabase Tips
- `database_password` is required on creation — store it in a secrets manager, never in `.tfvars`
- Use `supabase_branch` for preview environments tied to Git branches
- The `supabase_settings` resource takes JSON-encoded configuration blocks for API, auth, and other settings
- Always import existing projects rather than recreating — recreation destroys all data
- The `organization_id` field uses the organization slug, not a UUID

---

## 7. Upstash Provider

### Key Resources

**upstash_redis_database** — Serverless Redis instance.

```hcl
resource "upstash_redis_database" "cache" {
  database_name  = "${var.project_name}-cache-${var.environment}"
  region         = "global"
  primary_region = "us-east-1"
  tls            = true
  eviction       = true

  # Budget cap to prevent runaway costs
  auto_scale = var.environment == "prod"
  budget     = var.environment == "prod" ? 50 : 10
}

# Outputs for wiring to other services
output "redis_endpoint" {
  value = upstash_redis_database.cache.endpoint
}

output "redis_password" {
  value     = upstash_redis_database.cache.password
  sensitive = true
}

output "redis_rest_url" {
  value = upstash_redis_database.cache.rest_url
}
```

**upstash_redis_database** with read replicas (global):

```hcl
resource "upstash_redis_database" "global_cache" {
  database_name  = "${var.project_name}-global-cache"
  region         = "global"
  primary_region = "us-east-1"
  read_regions   = ["eu-west-1", "ap-southeast-1"]
  tls            = true
  eviction       = true
  auto_scale     = true
}
```

**upstash_kafka_cluster** — Serverless Kafka.

```hcl
resource "upstash_kafka_cluster" "events" {
  cluster_name = "${var.project_name}-events-${var.environment}"
  region       = "us-east-1"
  multizone    = var.environment == "prod"
}

resource "upstash_kafka_topic" "user_events" {
  topic_name       = "user-events"
  partitions       = var.environment == "prod" ? 3 : 1
  retention_time   = 604800000  # 7 days in ms
  retention_size   = 1073741824 # 1 GB
  max_message_size = 1048576    # 1 MB
  cleanup_policy   = "delete"
  cluster_id       = upstash_kafka_cluster.events.cluster_id
}
```

**upstash_qstash_topic** — QStash message queue topics.

```hcl
resource "upstash_qstash_topic_v2" "notifications" {
  name = "${var.project_name}-notifications"

  endpoints = [
    "https://api.example.com/webhooks/notify",
    "https://backup.example.com/webhooks/notify",
  ]
}
```

### Upstash Tips
- Use `region = "global"` with `primary_region` for globally distributed Redis
- Set `budget` to cap costs — this throttles the database when the budget is hit
- `auto_scale = true` upgrades plans automatically; combine with `budget` for safety
- `tls = true` is required for new databases and should always be enabled
- The REST URL/token outputs are perfect for serverless environments (Vercel Edge, Cloudflare Workers)

---

## 8. Full-Stack Composition Patterns

### Pattern A: Vercel + Supabase + Upstash (Serverless Full-Stack)

The most common modern stack — ideal for Next.js apps.

```
infrastructure/
├── modules/
│   ├── supabase-project/     # Supabase project + settings
│   ├── upstash-cache/        # Redis + optional Kafka
│   └── vercel-frontend/      # Vercel project + domains + env wiring
├── environments/
│   ├── dev/
│   │   ├── main.tf           # Composes modules
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── terraform.tfvars
│       └── backend.tf
└── versions.tf
```

```hcl
# environments/dev/main.tf
module "supabase" {
  source          = "../../modules/supabase-project"
  project_name    = var.project_name
  environment     = "dev"
  org_id          = var.supabase_org_id
  db_password     = var.supabase_db_password
  region          = "ap-southeast-1"
}

module "cache" {
  source         = "../../modules/upstash-cache"
  project_name   = var.project_name
  environment    = "dev"
  region         = "us-east-1"
  enable_global  = false
  budget         = 10
}

module "frontend" {
  source       = "../../modules/vercel-frontend"
  project_name = var.project_name
  environment  = "dev"
  team_id      = var.vercel_team_id
  git_repo     = "myorg/my-app"

  supabase_url      = module.supabase.project_url
  supabase_anon_key = module.supabase.anon_key
  redis_rest_url    = module.cache.rest_url
  redis_rest_token  = module.cache.rest_token
}
```

### Pattern B: Vercel + Railway + Supabase + Upstash (Frontend + Dedicated Backend)

When you need a long-running backend process (WebSockets, cron jobs, workers).

```hcl
# Frontend on Vercel
module "frontend" {
  source       = "../../modules/vercel-frontend"
  # ... same as above, but API_URL points to Railway
  api_url = module.backend.service_url
}

# Backend on Railway
module "backend" {
  source       = "../../modules/railway-backend"
  project_name = var.project_name
  environment  = var.environment

  # Wire Supabase and Upstash connections
  database_url = module.supabase.connection_string
  redis_url    = module.cache.redis_url
}

# Database on Supabase
module "supabase" {
  source = "../../modules/supabase-project"
  # ...
}

# Cache on Upstash
module "cache" {
  source = "../../modules/upstash-cache"
  # ...
}
```

### Pattern C: Render Full-Stack (Simpler Alternative)

When you want everything on one platform with Terraform managing config.

```hcl
module "api" {
  source = "../../modules/render-service"
  name   = "api"
  # ...
}

module "worker" {
  source = "../../modules/render-service"
  name   = "worker"
  # ...
}

# Still use Supabase for managed Postgres + auth
module "supabase" {
  source = "../../modules/supabase-project"
  # ...
}

# Still use Upstash for serverless Redis
module "cache" {
  source = "../../modules/upstash-cache"
  # ...
}
```

---

## 9. Environment Variable Wiring

The most critical part of multi-platform Terraform is wiring service credentials between providers. Here's the connection map:

```
Supabase ──┬── SUPABASE_URL ──────────────► Vercel env vars
            ├── SUPABASE_ANON_KEY ─────────► Vercel env vars (public)
            ├── SUPABASE_SERVICE_ROLE_KEY ──► Railway/Render env vars (private)
            └── DATABASE_URL ──────────────► Railway/Render env vars

Upstash ───┬── UPSTASH_REDIS_REST_URL ────► Vercel env vars
            ├── UPSTASH_REDIS_REST_TOKEN ──► Vercel env vars (sensitive)
            └── REDIS_URL ─────────────────► Railway/Render env vars
```

**Key security rules:**
- `SUPABASE_ANON_KEY` is safe to expose to the browser (it's a public key with RLS protection)
- `SUPABASE_SERVICE_ROLE_KEY` bypasses RLS — only use in server-side environments (Railway/Render), never in Vercel client-side code
- `UPSTASH_REDIS_REST_TOKEN` is sensitive — mark it as `sensitive = true` in Vercel env vars
- `DATABASE_URL` with direct Postgres connection strings should only go to backend services, not frontend

---

## 10. Secrets Management for Platform Tokens

All five providers require API tokens. Here's how to manage them safely:

**Option A: Environment variables (simplest)**
```bash
export VERCEL_API_TOKEN="..."
export RAILWAY_TOKEN="..."
export RENDER_API_KEY="..."
export RENDER_OWNER_ID="..."
export SUPABASE_ACCESS_TOKEN="..."
export TF_VAR_upstash_email="..."
export TF_VAR_upstash_api_key="..."
```

**Option B: `.env` file with direnv (local dev)**
```bash
# .envrc (add to .gitignore!)
export VERCEL_API_TOKEN="..."
export RAILWAY_TOKEN="..."
# ...
```

**Option C: CI/CD secrets (GitHub Actions)**
```yaml
env:
  VERCEL_API_TOKEN: ${{ secrets.VERCEL_API_TOKEN }}
  RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
  RENDER_API_KEY: ${{ secrets.RENDER_API_KEY }}
  RENDER_OWNER_ID: ${{ secrets.RENDER_OWNER_ID }}
  SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
  TF_VAR_upstash_email: ${{ secrets.UPSTASH_EMAIL }}
  TF_VAR_upstash_api_key: ${{ secrets.UPSTASH_API_KEY }}
```

**Option D: HashiCorp Vault or 1Password CLI (production)**
```bash
# Using 1Password CLI
export VERCEL_API_TOKEN=$(op read "op://DevOps/Vercel/api-token")
```

Never commit any of these tokens to version control. Use `.gitignore` to exclude `.tfvars` files that contain secrets, and use `sensitive = true` on all credential variables.
