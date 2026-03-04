---
name: terraform-specialist
description: Infrastructure as Code specialist for Terraform/OpenTofu covering traditional cloud (AWS, Azure, GCP) and modern developer platforms (Vercel, Railway, Render, Supabase, Upstash). Use this skill whenever the user mentions Terraform, OpenTofu, .tf files, IaC, HCL, terraform plan/apply/import, state management, tfvars, modules, providers, workspaces, or remote backends. Also trigger for provisioning cloud infrastructure, managing drift, importing resources, designing modules, CI/CD for infra changes, or multi-environment strategies. Trigger when managing Vercel projects/domains, Railway services, Render web services, Supabase projects/branches, or Upstash Redis/Kafka via Terraform. Even if the user just says "deploy infrastructure", "Terraform my Vercel project", or "IaC for my Next.js + Supabase stack", use this skill.
---

# Terraform Specialist

Expert-level guidance for Terraform and OpenTofu infrastructure automation — from traditional cloud providers (AWS, Azure, GCP) to modern developer platforms (Vercel, Railway, Render, Supabase, Upstash). Covers everything from writing your first resource block to designing enterprise module libraries with remote state, policy enforcement, and CI/CD pipelines.

## Core Philosophy

These principles guide every piece of Terraform code this skill produces:

1. **DRY (Don't Repeat Yourself)** — Extract reusable patterns into modules. If you write the same resource block twice, it belongs in a module.
2. **State is sacred** — Always configure remote backends with locking. Never manually edit state files. Always back up before migrations.
3. **Plan before apply** — Every change gets reviewed via `terraform plan`. Automate this in CI/CD so no one skips it.
4. **Pin versions for reproducibility** — Lock provider versions, module versions, and the Terraform version itself. Upgrades are intentional, not accidental.
5. **Use data sources over hardcoded values** — Look up AMIs, subnet IDs, and account info dynamically rather than pasting them into code.
6. **Least privilege everywhere** — Terraform service accounts get only the permissions they need, scoped to the resources they manage.

## When to Use This Skill

Use this skill for any Terraform-related task, including but not limited to:

- Writing new Terraform configurations from scratch
- Designing reusable module libraries
- Setting up remote state backends (S3, Azure Storage, GCS, Terraform Cloud)
- Managing multi-environment deployments (dev/staging/prod)
- Importing existing cloud resources into Terraform management
- Debugging state issues, drift detection, and remediation
- Building CI/CD pipelines for infrastructure changes
- Migrating between Terraform versions or from other IaC tools
- Writing validation rules, policy-as-code, and governance
- Optimizing Terraform performance for large-scale deployments
- **Managing modern platform resources**: Vercel projects/domains/firewall, Railway services/variables, Render web services/databases, Supabase projects/settings/branches, Upstash Redis/Kafka/QStash
- **Wiring multi-platform stacks**: Connecting environment variables and credentials across Vercel + Supabase + Upstash + Railway/Render

## Workflow Overview

When the user asks for Terraform help, follow this general workflow:

### Step 1: Understand the Scope

Before writing any code, clarify:
- **What provider(s)?** Traditional cloud (AWS, Azure, GCP), modern platforms (Vercel, Railway, Render, Supabase, Upstash), or a combination?
- **What resources?** Networking, compute, databases, Kubernetes, frontend hosting, serverless functions, managed databases, caching?
- **What environment strategy?** Single env, workspaces, separate state files, or Terragrunt?
- **Existing infrastructure?** Greenfield or importing existing resources?
- **Team size and workflow?** Solo developer or team with CI/CD and approvals?

### Step 2: Choose the Right Architecture

Based on scope, select the appropriate pattern:

| Scenario | Recommended Pattern |
|----------|-------------------|
| Small project, single env | Flat structure with local or single remote state |
| Multi-environment (dev/staging/prod) | Workspaces OR directory-per-environment |
| Team with shared modules | Module registry + remote state with locking |
| Enterprise / multi-team | Terraform Cloud/Enterprise or Terragrunt with strict module versioning |
| Existing infra to manage | Import workflow with state migration plan |
| Modern full-stack (Vercel+Supabase+Upstash) | Flat or workspace-based with multi-provider wiring |
| Frontend + dedicated backend (Vercel+Railway) | Module-per-service with shared variable outputs |
| Multi-platform with backends (Render+Supabase+Upstash) | Directory-per-environment with reusable platform modules |

**Reference:** See `references/architecture-patterns.md` for detailed guidance on each pattern.

### Step 3: Write the Code

Follow these structural conventions for all Terraform code:

**File Organization (standard module layout):**
```
module-name/
├── main.tf          # Primary resource definitions
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output value declarations
├── providers.tf     # Provider configuration and version constraints
├── versions.tf      # Terraform version constraints
├── locals.tf        # Local value computations
├── data.tf          # Data source lookups
├── backend.tf       # Backend/state configuration (root modules only)
├── terraform.tfvars # Default variable values (never commit secrets)
└── README.md        # Module documentation
```

**Naming Conventions:**
- Resources: `resource_type` + `_` + `descriptive_name` (e.g., `aws_instance.web_server`)
- Variables: `snake_case`, descriptive, with `description` and `type` always set
- Outputs: Match the resource attribute they expose (e.g., `instance_id`, `vpc_cidr`)
- Modules: `kebab-case` directory names (e.g., `networking/`, `compute-cluster/`)
- Tags: Always include `Name`, `Environment`, `ManagedBy = "terraform"`, `Project`

**Code Quality Rules:**
- Every variable has a `description`, `type`, and (where sensible) a `default` or `validation` block
- Every output has a `description`
- Use `terraform fmt` formatting — never deviate
- Use `terraform validate` before committing
- Group related resources in the same file; split into separate files when a single file exceeds ~150 lines
- Use `locals` to compute derived values rather than repeating expressions
- Prefer `for_each` over `count` for resources that need stable addressing

### Step 4: Configure State and Backend

Always configure remote state for anything beyond local experimentation.

**AWS S3 Backend Example:**
```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "environments/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

**Azure Storage Backend Example:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
  }
}
```

**GCS Backend Example:**
```hcl
terraform {
  backend "gcs" {
    bucket = "mycompany-terraform-state"
    prefix = "environments/production"
  }
}
```

**Reference:** See `references/state-management.md` for backend setup scripts, migration procedures, and disaster recovery.

### Step 5: Provide tfvars Examples

Always include example `.tfvars` files showing how to customize the configuration for different environments. Example:

```hcl
# environments/dev.tfvars
environment    = "dev"
instance_type  = "t3.micro"
instance_count = 1
enable_monitoring = false

# environments/prod.tfvars
environment    = "prod"
instance_type  = "t3.large"
instance_count = 3
enable_monitoring = true
```

### Step 6: Show Plan Output

When demonstrating Terraform code, always show what `terraform plan` would produce so the user can verify the expected behavior before applying. Use comments to annotate the plan output:

```
# terraform plan -var-file=environments/dev.tfvars

Terraform will perform the following actions:

  # aws_instance.web_server will be created
  + resource "aws_instance" "web_server" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t3.micro"
      + tags          = {
          + "Environment" = "dev"
          + "ManagedBy"   = "terraform"
          + "Name"        = "dev-web-server"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

## Module Design Guide

Modules are the primary unit of reuse. Design them well.

**When to create a module:**
- You've written the same resource pattern more than once
- A logical grouping of resources always deploys together (e.g., VPC + subnets + route tables)
- You want to enforce standards (tagging, naming, security policies) across teams

**Module interface design:**
- Keep the input surface small — expose only what callers need to customize
- Use sensible defaults so modules work out of the box
- Validate inputs with `validation` blocks to catch errors early
- Output everything downstream modules or root configurations might need
- Document with inline `description` fields and a README

**Module versioning:**
- Use semantic versioning (MAJOR.MINOR.PATCH)
- Pin module sources to specific versions or tags
- Never use `ref=main` in production — always use a release tag

```hcl
module "vpc" {
  source  = "git::https://github.com/mycompany/terraform-modules.git//networking/vpc?ref=v2.1.0"
  # OR from registry:
  # source  = "app.terraform.io/mycompany/vpc/aws"
  # version = "~> 2.1"

  environment = var.environment
  cidr_block  = var.vpc_cidr
}
```

**Reference:** See `references/module-patterns.md` for complete module examples including testing with Terratest.

## Importing Existing Resources

When the user has existing cloud resources they want to bring under Terraform management:

1. **Audit** — List existing resources (use cloud CLI tools or resource explorer)
2. **Write config** — Create the Terraform resource blocks that match the existing state
3. **Import** — Use `terraform import` (or `import` blocks in TF 1.5+) to map resources to config
4. **Plan** — Run `terraform plan` to verify zero diff (no changes planned)
5. **Iterate** — Fix any drift between config and actual state until the plan is clean

**Modern import block syntax (Terraform 1.5+):**
```hcl
import {
  to = aws_instance.web_server
  id = "i-1234567890abcdef0"
}
```

**Generate config from imports (Terraform 1.5+):**
```bash
terraform plan -generate-config-out=generated.tf
```

This generates HCL for imported resources automatically. Review and refine the generated code before committing.

## CI/CD Integration

Infrastructure changes should go through the same rigor as application code.

**Recommended pipeline stages:**
1. `terraform fmt -check` — Enforce formatting
2. `terraform validate` — Syntax and type checking
3. `terraform plan` — Preview changes, save plan file
4. **Manual approval** — Human reviews the plan output
5. `terraform apply <plan-file>` — Apply the reviewed plan
6. **Post-apply validation** — Run smoke tests or policy checks

**Pre-commit hooks** — Catch issues before they reach CI:
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_docs
```

**Reference:** See `references/cicd-pipelines.md` for complete GitHub Actions, GitLab CI, and Azure DevOps pipeline examples.

## Provider Configuration

Always pin provider versions explicitly.

**Traditional cloud providers:**

```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}
```

**Modern platform providers:**

```hcl
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

Only include the providers you actually need — don't declare all of them if you're only using Vercel + Supabase, for example.

**Provider maturity notes:**
- **Vercel** (`vercel/vercel`) — Official, stable, well-maintained. Supports projects, deployments, domains, firewall, and environment variables.
- **Railway** (`terraform-community-providers/railway`) — Community-maintained, actively developed. Covers projects, services, variables, custom domains. Not official from Railway.
- **Render** (`render-oss/render`) — Official but in early access. Expect possible breaking changes; pin versions tightly.
- **Supabase** (`supabase/supabase`) — Official, stable. Manages projects, settings, branches, and API keys.
- **Upstash** (`upstash/upstash`) — Official, stable. Manages Redis databases, Kafka clusters/topics, and QStash.

**Reference:** See `references/modern-platform-stack.md` for detailed provider configuration, resource examples, full-stack composition patterns, and environment variable wiring guides.

After running `terraform init`, commit the `.terraform.lock.hcl` file. This ensures all team members and CI use the exact same provider binaries.

## Modern Platform Stack Quick-Start

When the user wants to Terraform a Vercel/Railway/Render + Supabase + Upstash stack, read `references/modern-platform-stack.md` for complete examples. Here's the essential pattern:

**1. Configure providers** — Only include the ones you need. All authenticate via API tokens stored in environment variables.

**2. Create backend services first** — Supabase (database/auth) and Upstash (cache/queues) produce connection strings that frontend/backend services need.

**3. Wire environment variables** — The critical step. Use Terraform outputs from Supabase and Upstash as inputs to Vercel/Railway/Render environment variables:
```hcl
# Supabase project outputs → Vercel env vars
# Upstash Redis outputs → Vercel env vars + Railway/Render env vars
```

**4. Security rules for variable wiring:**
- `SUPABASE_ANON_KEY` — Safe for client-side (Vercel `NEXT_PUBLIC_*`), protected by Row Level Security
- `SUPABASE_SERVICE_ROLE_KEY` — Server-only (Railway/Render), bypasses RLS
- `DATABASE_URL` — Server-only, never expose to frontend
- `UPSTASH_REDIS_REST_TOKEN` — Mark as `sensitive = true`

**5. Common stack compositions:**
- **Serverless full-stack**: Vercel + Supabase + Upstash (no dedicated backend, everything runs on edge/serverless)
- **Frontend + backend**: Vercel + Railway + Supabase + Upstash (Railway for WebSockets, cron, workers)
- **All-in-one hosting**: Render + Supabase + Upstash (Render handles both frontend and backend)

## Common Pitfalls and How to Avoid Them

| Pitfall | Prevention |
|---------|-----------|
| State file corruption | Remote backend with locking + regular backups |
| Secrets in `.tfvars` or state | Use vault/secrets manager references, never plaintext |
| Unreviewed applies | CI/CD with mandatory plan review step |
| Provider version drift | Pin versions + commit lock file |
| Monolithic configurations | Break into modules at ~200 resource threshold |
| `count` index shifting | Use `for_each` with stable keys instead |
| Hardcoded values | Use `data` sources and `locals` for derivation |
| Missing tags | Use `default_tags` in provider config or policy-as-code |
| Leaking Supabase service_role_key to frontend | Only pass to server-side services (Railway/Render), never `NEXT_PUBLIC_*` |
| Platform API tokens in code | Use env vars (`VERCEL_API_TOKEN`, etc.), never in `.tf` files |
| Recreating Supabase project instead of importing | Always `import` existing projects — recreation destroys all data |
| Render provider breaking changes | Pin to exact version during early access period |
| Missing `team_id` on Vercel resources | Always specify when using team accounts |

## Security Considerations

- Never store secrets in Terraform files, state, or version control
- Use dynamic secrets from HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault
- Configure `default_tags` to enforce `ManagedBy = "terraform"` across all resources
- Use Sentinel, OPA, or tfsec for policy enforcement
- Restrict state file access with IAM policies — state files contain sensitive data
- Rotate Terraform service account credentials regularly
- Enable state encryption at rest and in transit

**Modern platform-specific security:**
- Platform API tokens (Vercel, Railway, Render, Supabase, Upstash) are high-privilege — they can create, modify, and delete all resources. Treat them like root credentials.
- Supabase `service_role_key` bypasses Row Level Security — only inject into server-side services, never into Vercel client-side env vars (`NEXT_PUBLIC_*`)
- Upstash Redis passwords and REST tokens are stored in Terraform state — ensure state is encrypted and access-restricted
- Use separate API tokens per environment when possible (e.g., separate Vercel tokens for dev vs prod)
- All five platform providers store sensitive outputs in state — this reinforces why remote state with encryption is mandatory

## Reference Files Summary

The `references/` directory contains deep-dive documentation:

- **`architecture-patterns.md`** — Detailed comparison of flat, workspace, directory-per-env, and Terragrunt patterns with decision matrices and migration guides
- **`state-management.md`** — Backend setup scripts for all major providers, state migration procedures, disaster recovery, and troubleshooting common state issues
- **`module-patterns.md`** — Complete reusable module examples for networking, compute, databases, and Kubernetes across AWS/Azure/GCP, including testing with Terratest
- **`cicd-pipelines.md`** — Ready-to-use pipeline definitions for GitHub Actions, GitLab CI, Azure DevOps, and Terraform Cloud, with approval gates and policy checks
- **`modern-platform-stack.md`** — Complete guide to Terraform providers for Vercel, Railway, Render, Supabase, and Upstash. Includes unified provider configuration, resource examples for each platform, full-stack composition patterns (serverless, frontend+backend, all-in-one), environment variable wiring maps, and secrets management for platform tokens

## Output Checklist

When completing a Terraform task, verify you have provided:

- [ ] Properly structured `.tf` files following the standard layout
- [ ] `versions.tf` with pinned Terraform and provider versions
- [ ] Backend configuration for remote state (unless user specifies local)
- [ ] Example `.tfvars` files for at least two environments
- [ ] Sample `terraform plan` output showing expected behavior
- [ ] Module documentation with input/output descriptions
- [ ] Security considerations relevant to the specific resources
- [ ] CI/CD integration guidance if the user's workflow warrants it