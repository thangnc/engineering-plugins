# Engineering Plugins for Claude Code

A monorepo of Claude Code plugins providing expert domain knowledge as skills. Each plugin lives under `plugins/` and is independently installable from the marketplace.

## Plugins

### `cloud-devops`

Cloud infrastructure and DevOps skills — starting with a Terraform/OpenTofu specialist that covers traditional cloud providers and modern developer platforms.

**Skills:**

#### `terraform-specialist`

Expert-level guidance for Terraform and OpenTofu — from writing your first resource block to enterprise module libraries with remote state, CI/CD pipelines, and multi-platform stacks.

**Triggers automatically when you:**
- Mention Terraform, OpenTofu, `.tf` files, HCL, IaC
- Ask about `terraform plan`, `apply`, `import`, state management, workspaces, modules
- Need to provision AWS, Azure, or GCP infrastructure
- Manage modern platforms via Terraform: Vercel, Railway, Render, Supabase, Upstash
- Say things like _"Terraform my Vercel project"_ or _"IaC for my Next.js + Supabase stack"_

**What it covers:**

| Topic | Details |
|---|---|
| Architecture patterns | Flat, workspaces, directory-per-env, Terragrunt |
| Module design | Reusable modules, versioning, Terratest |
| State management | S3, Azure Storage, GCS backends, migration, DR |
| CI/CD | GitHub Actions, GitLab CI, Azure DevOps, Terraform Cloud |
| Modern platforms | Vercel, Railway, Render, Supabase, Upstash providers |
| Full-stack wiring | Env var propagation across multi-provider stacks |
| Security | Secrets management, least privilege, policy-as-code |

**Reference documentation** (`plugins/cloud-devops/skills/terraform-specialist/references/`):

- `architecture-patterns.md` — Pattern comparison with decision matrices
- `state-management.md` — Backend setup, migration, disaster recovery
- `module-patterns.md` — Complete module examples with testing
- `cicd-pipelines.md` — Ready-to-use pipeline definitions
- `modern-platform-stack.md` — Vercel/Railway/Render/Supabase/Upstash guide

### `db-design`

Database design and schema modeling skills — featuring a PostgreSQL Prisma specialist for production-grade data modeling with Prisma ORM on Node.js.

**Skills:**

#### `postgresql-prisma-specialist`

Expert-level guidance for designing PostgreSQL schemas using Prisma ORM — from initial data modeling to advanced performance patterns and migrations.

**Triggers automatically when you:**
- Mention database design, schema design, data modeling, Prisma schema
- Ask about PostgreSQL tables, indexes, constraints, relations, migrations
- Need help with query performance, N+1 problems, composite indexes
- Work with JSONB columns, enum types, full-text search, soft deletes
- Ask about multi-tenancy, polymorphic relations, audit trails
- Use Prisma-specific patterns like `@@map`, `@db.*`, or raw SQL
- Say things like _"design a database"_ or _"model this data"_ or _"set up the tables for my app"_

**What it covers:**

| Topic | Details |
|---|---|
| Data modeling | Entity relationships, normalization, Prisma schema patterns |
| Type selection | PostgreSQL types mapped to Prisma field types |
| Indexing strategy | B-tree, GIN, GiST, composite indexes, partial indexes |
| Constraints | Unique, check, foreign keys, cascading rules |
| Performance | Query optimization, N+1 prevention, connection pooling |
| Advanced patterns | Multi-tenancy, soft deletes, audit trails, polymorphic relations |

**Reference documentation** (`plugins/db-design/skills/postgresql-prisma-specialist/references/`):

- `types-and-fields.md` — PostgreSQL types and Prisma field mappings
- `indexes-and-performance.md` — Indexing strategies and performance tuning
- `advanced-patterns.md` — Multi-tenancy, soft deletes, audit trails, and more

## Installation

Install from the marketplace:

```bash
claude plugin install https://github.com/thangnc/engineering-plugins
```

Or install locally (from this repo root):

```bash
claude plugin install ./plugins/cloud-devops
claude plugin install ./plugins/db-design
```

## Repository Structure

```
plugins/
  cloud-devops/
    .claude-plugin/
      plugin.json        # Plugin manifest for the marketplace
    skills/
      terraform-specialist/
        SKILL.md         # Trigger conditions + core instructions
        references/      # Deep-dive documentation
  db-design/
    .claude-plugin/
      plugin.json        # Plugin manifest for the marketplace
    skills/
      postgresql-prisma-specialist/
        SKILL.md         # Trigger conditions + core instructions
        references/      # Deep-dive documentation

.claude-plugin/
  marketplace.json       # Marketplace listing metadata
```

## Prerequisites

Install the CLIs for the platforms you use:

```bash
# Terraform / OpenTofu
brew install terraform
brew install opentofu

# AWS
brew install awscli

# GCP
brew install --cask google-cloud-sdk

# Azure
brew install azure-cli

# Kubernetes
brew install kubectl helm
```

Modern platforms (Vercel, Railway, Render, Supabase, Upstash) are managed via their Terraform providers — no CLI required, just API tokens.

## License

MIT