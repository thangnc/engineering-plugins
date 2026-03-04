# Architecture Patterns for Terraform Projects

## Table of Contents
1. Flat Structure
2. Workspace-Based
3. Directory-Per-Environment
4. Terragrunt Wrapper
5. Terraform Cloud / Enterprise
6. Decision Matrix

---

## 1. Flat Structure

Best for: Small projects, single environment, solo developer.

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── versions.tf
├── terraform.tfvars
└── backend.tf
```

**Pros:** Simple, easy to understand, minimal overhead.
**Cons:** Doesn't scale. No environment isolation. One state file for everything.

**When to use:** Proof of concepts, personal projects, learning Terraform, single-environment tools.

**When to move on:** As soon as you need a second environment or a second team member.

---

## 2. Workspace-Based Multi-Environment

Best for: Small-to-medium teams managing 2-4 environments with identical infrastructure topology.

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── versions.tf
├── backend.tf
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

Usage:
```bash
terraform workspace select dev
terraform plan -var-file=environments/dev.tfvars
terraform apply -var-file=environments/dev.tfvars
```

**Pros:** Single codebase for all environments. Easy to keep environments in sync. Built-in Terraform feature.
**Cons:** All environments share the same backend config. Workspace name is implicit, easy to forget. Hard to have structural differences between environments.

**Key pattern — reference workspace in code:**
```hcl
locals {
  environment = terraform.workspace
  name_prefix = "${local.environment}-${var.project_name}"
}

resource "aws_instance" "web" {
  instance_type = var.instance_type  # Different per .tfvars
  tags = {
    Environment = local.environment
  }
}
```

**Guard against accidental production changes:**
```hcl
# In CI/CD, add a check:
# if [ "$(terraform workspace show)" = "prod" ] && [ "$BRANCH" != "main" ]; then
#   echo "ERROR: Production changes only allowed from main branch"
#   exit 1
# fi
```

---

## 3. Directory-Per-Environment

Best for: Medium-to-large teams where environments differ structurally or need independent state management.

```
infrastructure/
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── database/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── dev/
│   │   ├── main.tf          # Calls modules with dev params
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   ├── backend.tf       # Separate state per env
│   │   └── providers.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   ├── backend.tf
│   │   └── providers.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       ├── backend.tf
│       └── providers.tf
└── global/                   # Shared resources (IAM, DNS zones)
    ├── main.tf
    ├── backend.tf
    └── terraform.tfvars
```

**Pros:** Complete isolation between environments. Each environment can diverge structurally. Independent state files. Clear CI/CD targeting.
**Cons:** Code duplication in environment `main.tf` files (mitigated by modules). More files to maintain.

**Key pattern — environment main.tf calls shared modules:**
```hcl
# environments/dev/main.tf
module "networking" {
  source = "../../modules/networking"

  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
}

module "compute" {
  source = "../../modules/compute"

  environment   = "dev"
  vpc_id        = module.networking.vpc_id
  subnet_ids    = module.networking.private_subnet_ids
  instance_type = "t3.micro"
  instance_count = 1
}
```

**Cross-environment data sharing via remote state:**
```hcl
# In staging, reference dev's VPC peering info:
data "terraform_remote_state" "dev_networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "environments/dev/networking.tfstate"
    region = "us-east-1"
  }
}
```

---

## 4. Terragrunt Wrapper

Best for: Large organizations with many environments, accounts, and regions that need DRY configuration.

```
infrastructure/
├── terragrunt.hcl            # Root config (backend, provider defaults)
├── modules/                   # Reusable Terraform modules
│   ├── vpc/
│   ├── eks/
│   └── rds/
└── live/
    ├── terragrunt.hcl        # Common config for all envs
    ├── dev/
    │   ├── terragrunt.hcl    # Dev-specific overrides
    │   ├── us-east-1/
    │   │   ├── vpc/
    │   │   │   └── terragrunt.hcl
    │   │   └── eks/
    │   │       └── terragrunt.hcl
    │   └── eu-west-1/
    │       └── vpc/
    │           └── terragrunt.hcl
    └── prod/
        ├── terragrunt.hcl
        └── us-east-1/
            ├── vpc/
            │   └── terragrunt.hcl
            └── eks/
                └── terragrunt.hcl
```

**Pros:** Extreme DRY — no duplicated backend or provider config. Dependency management between modules. Multi-account/region support. Run all or targeted subsets.
**Cons:** Additional tool dependency. Learning curve. Debugging can be harder.

**Key pattern — root terragrunt.hcl generates backend config:**
```hcl
# live/terragrunt.hcl
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "mycompany-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

---

## 5. Terraform Cloud / Enterprise

Best for: Organizations wanting managed state, policy enforcement, and team workflows without self-managing backends.

Features: Remote state with built-in locking, VCS integration, Sentinel policy-as-code, cost estimation, private module registry, SSO and RBAC.

```hcl
terraform {
  cloud {
    organization = "mycompany"
    workspaces {
      tags = ["networking", "production"]
    }
  }
}
```

**Pros:** Fully managed state. Built-in policy enforcement. Audit logging. Team access controls.
**Cons:** Vendor lock-in. Cost at scale. Network connectivity requirements.

---

## 6. Decision Matrix

| Factor | Flat | Workspaces | Dir-Per-Env | Terragrunt | TF Cloud |
|--------|------|------------|-------------|------------|----------|
| Team size | 1 | 2-5 | 5-20 | 10+ | Any |
| Environments | 1 | 2-4 identical | 2+ (can differ) | Many | Many |
| State isolation | None | Partial | Full | Full | Full |
| Code duplication | N/A | Low | Medium | Very low | Low |
| Learning curve | Minimal | Low | Medium | High | Medium |
| CI/CD complexity | Simple | Medium | Medium | High | Low |
| Cost | Free | Free | Free | Free | Paid |

**Migration path:** Most teams evolve Flat → Workspaces → Directory-Per-Env → Terragrunt or TF Cloud as complexity grows. Plan your state migration before switching patterns.
