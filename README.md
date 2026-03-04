# Cloud Infrastructure Skills for Claude Code

A collection of Claude Code skills for managing cloud infrastructure — starting with a Terraform/OpenTofu specialist that covers traditional cloud providers and modern developer platforms.

## Skills

### `terraform-specialist`

Expert-level guidance for Terraform and OpenTofu — from writing your first resource block to enterprise module libraries with remote state, CI/CD pipelines, and multi-platform stacks.

**Triggers automatically when you:**
- Mention Terraform, OpenTofu, `.tf` files, HCL, IaC
- Ask about `terraform plan`, `apply`, `import`, state management, workspaces, modules
- Need to provision AWS, Azure, or GCP infrastructure
- Manage modern platforms via Terraform: Vercel, Railway, Render, Supabase, Upstash
- Say things like _"Terraform my Vercel project"_ or _"IaC for my Next.js + Supabase stack"_

**What it covers:**

| Topic               | Details                                               |
|---------------------|-------------------------------------------------------|
| Architecture patterns | Flat, workspaces, directory-per-env, Terragrunt     |
| Module design       | Reusable modules, versioning, Terratest               |
| State management    | S3, Azure Storage, GCS backends, migration, DR        |
| CI/CD               | GitHub Actions, GitLab CI, Azure DevOps, Terraform Cloud |
| Modern platforms    | Vercel, Railway, Render, Supabase, Upstash providers  |
| Full-stack wiring   | Env var propagation across multi-provider stacks      |
| Security            | Secrets management, least privilege, policy-as-code   |

**Reference documentation** (`skills/terraform-specialist/references/`):

- `architecture-patterns.md` — Pattern comparison with decision matrices
- `state-management.md` — Backend setup, migration, disaster recovery
- `module-patterns.md` — Complete module examples with testing
- `cicd-pipelines.md` — Ready-to-use pipeline definitions
- `modern-platform-stack.md` — Vercel/Railway/Render/Supabase/Upstash guide

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