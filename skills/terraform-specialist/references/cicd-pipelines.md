# CI/CD Pipelines for Terraform

## Table of Contents
1. Pipeline Architecture
2. GitHub Actions
3. GitLab CI
4. Azure DevOps
5. Terraform Cloud VCS Workflow
6. Pre-Commit Hooks
7. Makefile for Common Operations

---

## 1. Pipeline Architecture

Every Terraform CI/CD pipeline follows this flow:

```
PR Created/Updated
  → terraform fmt -check
  → terraform validate
  → terraform plan (saved to file)
  → Post plan output as PR comment
  → Wait for manual approval

PR Merged to main
  → terraform plan (verify again)
  → terraform apply (using saved plan OR re-plan + apply)
  → Post-apply validation / smoke tests
  → Notify team
```

**Critical rules:**
- Never run `terraform apply` without a prior `terraform plan` review
- Production applies only from the main/release branch
- Use plan files (`-out=tfplan`) so the apply matches exactly what was reviewed
- Store plan files as CI artifacts (they're ephemeral and environment-specific)

---

## 2. GitHub Actions

### Plan on PR, Apply on Merge

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    branches: [main]
    paths: ['infrastructure/**']
  push:
    branches: [main]
    paths: ['infrastructure/**']

permissions:
  contents: read
  pull-requests: write
  id-token: write  # For OIDC auth

env:
  TF_VERSION: "1.7.0"
  WORKING_DIR: "infrastructure/environments/production"

jobs:
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Terraform Init
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform init -input=false

      - name: Terraform Format Check
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform validate

      - name: Terraform Plan
        id: plan
        working-directory: ${{ env.WORKING_DIR }}
        run: |
          terraform plan -input=false -no-color -out=tfplan 2>&1 | tee plan_output.txt
          echo "plan_exit_code=$?" >> $GITHUB_OUTPUT
        continue-on-error: true

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('${{ env.WORKING_DIR }}/plan_output.txt', 'utf8');
            const truncated = plan.length > 60000
              ? plan.substring(0, 60000) + '\n\n... (truncated)'
              : plan;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `### Terraform Plan\n\`\`\`\n${truncated}\n\`\`\``
            });

      - name: Plan Status
        if: steps.plan.outputs.plan_exit_code == '1'
        run: exit 1

  apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production  # Requires manual approval in GitHub
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Terraform Init
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform init -input=false

      - name: Terraform Apply
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform apply -input=false -auto-approve
```

---

## 3. GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - apply

variables:
  TF_VERSION: "1.7.0"
  TF_ROOT: "infrastructure/environments/production"

image:
  name: hashicorp/terraform:$TF_VERSION
  entrypoint: [""]

cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - ${TF_ROOT}/.terraform

before_script:
  - cd ${TF_ROOT}
  - terraform init -input=false

validate:
  stage: validate
  script:
    - terraform fmt -check -recursive
    - terraform validate

plan:
  stage: plan
  script:
    - terraform plan -input=false -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 day
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

apply:
  stage: apply
  script:
    - terraform apply -input=false tfplan
  dependencies:
    - plan
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual  # Requires manual click to apply
  environment:
    name: production
```

---

## 4. Azure DevOps

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/**

pr:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/**

variables:
  tfVersion: '1.7.0'
  workingDir: 'infrastructure/environments/production'
  serviceConnection: 'azure-terraform-sc'

stages:
  - stage: Validate
    jobs:
      - job: ValidateTerraform
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: $(tfVersion)

          - script: |
              cd $(workingDir)
              terraform init -input=false -backend-config="access_key=$(ARM_ACCESS_KEY)"
              terraform fmt -check -recursive
              terraform validate
            displayName: 'Validate'

  - stage: Plan
    dependsOn: Validate
    jobs:
      - job: PlanTerraform
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: $(tfVersion)

          - script: |
              cd $(workingDir)
              terraform init -input=false -backend-config="access_key=$(ARM_ACCESS_KEY)"
              terraform plan -input=false -out=tfplan
            displayName: 'Plan'

          - publish: $(workingDir)/tfplan
            artifact: terraform-plan

  - stage: Apply
    dependsOn: Plan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: ApplyTerraform
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'production'  # Requires approval gate
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: terraform-plan

                - task: TerraformInstaller@1
                  inputs:
                    terraformVersion: $(tfVersion)

                - script: |
                    cd $(workingDir)
                    terraform init -input=false -backend-config="access_key=$(ARM_ACCESS_KEY)"
                    terraform apply -input=false $(Pipeline.Workspace)/terraform-plan/tfplan
                  displayName: 'Apply'
```

---

## 5. Terraform Cloud VCS Workflow

When using Terraform Cloud, VCS integration handles most of the pipeline:

```hcl
# In Terraform Cloud workspace settings:
# - VCS: Connect to your repo
# - Working Directory: infrastructure/environments/production
# - Apply Method: Manual (requires approval in TFC UI)
# - Terraform Version: 1.7.0

terraform {
  cloud {
    organization = "mycompany"
    workspaces {
      name = "production-infrastructure"
    }
  }
}
```

Terraform Cloud automatically runs plan on PR, posts results, and waits for manual confirm before apply.

---

## 6. Pre-Commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
        args:
          - --hook-config=--retry-once-with-cleanup=true
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl
      - id: terraform_docs
        args:
          - --args=--config=.terraform-docs.yml
      - id: terraform_tfsec
        args:
          - --args=--minimum-severity=HIGH

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-merge-conflict
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-yaml
      - id: detect-private-key
```

Install:
```bash
pip install pre-commit
pre-commit install
```

---

## 7. Makefile for Common Operations

```makefile
# Makefile
.PHONY: init plan apply destroy fmt validate lint docs clean

ENV ?= dev
TF_DIR = infrastructure/environments/$(ENV)
TF_FLAGS = -input=false

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

init: ## Initialize Terraform (ENV=dev|staging|prod)
	cd $(TF_DIR) && terraform init $(TF_FLAGS)

plan: init ## Run terraform plan (ENV=dev|staging|prod)
	cd $(TF_DIR) && terraform plan $(TF_FLAGS) -var-file=terraform.tfvars -out=tfplan

apply: ## Apply the saved plan (ENV=dev|staging|prod)
	cd $(TF_DIR) && terraform apply $(TF_FLAGS) tfplan

destroy: ## Destroy infrastructure (ENV=dev|staging|prod) — USE WITH CAUTION
	cd $(TF_DIR) && terraform destroy $(TF_FLAGS) -var-file=terraform.tfvars

fmt: ## Format all Terraform files
	terraform fmt -recursive infrastructure/

validate: init ## Validate Terraform configuration
	cd $(TF_DIR) && terraform validate

lint: ## Run tflint
	cd $(TF_DIR) && tflint --init && tflint

docs: ## Generate module documentation
	terraform-docs markdown table --output-file README.md infrastructure/modules/

clean: ## Remove local Terraform files
	find infrastructure/ -name ".terraform" -type d -exec rm -rf {} +
	find infrastructure/ -name "tfplan" -delete
	find infrastructure/ -name "*.tfstate.backup" -delete

state-backup: ## Backup current state (ENV=dev|staging|prod)
	cd $(TF_DIR) && terraform state pull > "state-backup-$$(date +%Y%m%d-%H%M%S).tfstate"

drift-check: init ## Check for infrastructure drift (ENV=dev|staging|prod)
	cd $(TF_DIR) && terraform plan $(TF_FLAGS) -var-file=terraform.tfvars -detailed-exitcode
```

Usage:
```bash
make plan ENV=dev        # Plan dev environment
make apply ENV=dev       # Apply dev environment
make plan ENV=prod       # Plan production
make drift-check ENV=prod # Check prod for drift
```