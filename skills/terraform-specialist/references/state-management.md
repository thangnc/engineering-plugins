# State Management Deep Dive

## Table of Contents
1. Backend Setup by Provider
2. State Locking
3. State Migration Procedures
4. Disaster Recovery
5. Troubleshooting Common Issues

---

## 1. Backend Setup by Provider

### AWS S3 + DynamoDB

**Bootstrap script — run once to create the state infrastructure:**
```bash
#!/bin/bash
# bootstrap-state.sh — Creates S3 bucket + DynamoDB for Terraform state
set -euo pipefail

BUCKET_NAME="mycompany-terraform-state"
TABLE_NAME="terraform-state-lock"
REGION="us-east-1"

# Create S3 bucket with versioning and encryption
aws s3api create-bucket \
  --bucket "$BUCKET_NAME" \
  --region "$REGION"

aws s3api put-bucket-versioning \
  --bucket "$BUCKET_NAME" \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket "$BUCKET_NAME" \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms"}}]
  }'

aws s3api put-public-access-block \
  --bucket "$BUCKET_NAME" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name "$TABLE_NAME" \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region "$REGION"

echo "State backend ready: s3://$BUCKET_NAME with lock table $TABLE_NAME"
```

**Backend configuration:**
```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "project-name/environment/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
    # Optional: Use a KMS key for encryption
    # kms_key_id   = "arn:aws:kms:us-east-1:123456789:key/..."
  }
}
```

### Azure Storage Account

**Bootstrap script:**
```bash
#!/bin/bash
set -euo pipefail

RESOURCE_GROUP="rg-terraform-state"
STORAGE_ACCOUNT="stterraformstate$(openssl rand -hex 4)"
CONTAINER="tfstate"
LOCATION="eastus"

az group create --name "$RESOURCE_GROUP" --location "$LOCATION"

az storage account create \
  --resource-group "$RESOURCE_GROUP" \
  --name "$STORAGE_ACCOUNT" \
  --sku Standard_LRS \
  --encryption-services blob \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

az storage container create \
  --name "$CONTAINER" \
  --account-name "$STORAGE_ACCOUNT"

echo "State backend ready: $STORAGE_ACCOUNT/$CONTAINER"
echo "Storage account name: $STORAGE_ACCOUNT"
```

**Backend configuration:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstateXXXX"
    container_name       = "tfstate"
    key                  = "project-name/environment.tfstate"
    # Use Azure AD auth instead of storage keys when possible:
    # use_azuread_auth   = true
  }
}
```

### Google Cloud Storage

**Bootstrap script:**
```bash
#!/bin/bash
set -euo pipefail

PROJECT_ID="my-project"
BUCKET_NAME="${PROJECT_ID}-terraform-state"
REGION="us-central1"

gsutil mb -p "$PROJECT_ID" -l "$REGION" "gs://$BUCKET_NAME"
gsutil versioning set on "gs://$BUCKET_NAME"
gsutil uniformbucketlevelaccess set on "gs://$BUCKET_NAME"

echo "State backend ready: gs://$BUCKET_NAME"
```

**Backend configuration:**
```hcl
terraform {
  backend "gcs" {
    bucket = "my-project-terraform-state"
    prefix = "project-name/environment"
  }
}
```

---

## 2. State Locking

State locking prevents concurrent operations that could corrupt state.

| Backend | Locking Mechanism | Setup Required |
|---------|-------------------|---------------|
| S3 | DynamoDB table | Create table with `LockID` partition key |
| Azure Storage | Blob lease | Built-in, no extra setup |
| GCS | Object versioning | Built-in, no extra setup |
| Terraform Cloud | Built-in | No extra setup |
| Consul | KV store | Configure Consul endpoint |

**If a lock gets stuck** (e.g., CI/CD runner crashed mid-apply):
```bash
# Force unlock — use with extreme caution
terraform force-unlock LOCK_ID

# The lock ID is shown in the error message:
# "Error: Error locking state: Error acquiring the state lock"
# "Lock Info: ID: a1b2c3d4-..."
```

Before force-unlocking, verify that no other operation is actually running. Check CI/CD pipelines, ask teammates, and look at the lock metadata (who, when, operation type).

---

## 3. State Migration Procedures

### Migrating from Local to Remote Backend

```bash
# 1. Back up current state
cp terraform.tfstate terraform.tfstate.backup

# 2. Add backend configuration to backend.tf
# (write the backend block)

# 3. Initialize with migration
terraform init -migrate-state

# 4. Verify state was migrated
terraform plan  # Should show no changes

# 5. Remove local state file (now redundant)
rm terraform.tfstate terraform.tfstate.backup
```

### Migrating Between Remote Backends

```bash
# 1. Pull current state locally
terraform state pull > terraform.tfstate.backup

# 2. Update backend.tf with new backend configuration

# 3. Initialize with migration
terraform init -migrate-state

# 4. Verify
terraform plan  # Should show no changes

# 5. Clean up old backend (after confirming everything works)
```

### Moving Resources Between State Files

```bash
# Remove from source state
terraform state rm module.networking

# Import into destination state (run from destination directory)
terraform import module.networking.aws_vpc.main vpc-12345
# OR use terraform state mv with -state and -state-out flags
```

### Renaming Resources Without Destroy/Recreate

```hcl
# Use moved blocks (Terraform 1.1+)
moved {
  from = aws_instance.web_server
  to   = aws_instance.application_server
}
```

Run `terraform plan` after adding the moved block — it should show the rename with no destroy.

---

## 4. Disaster Recovery

### State Backup Strategy

**Automated backups (S3 example with versioning):**
```bash
# S3 versioning handles this automatically
# To list versions of a state file:
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix environments/production/terraform.tfstate

# Restore a specific version:
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key environments/production/terraform.tfstate \
  --version-id "VERSION_ID" \
  restored-state.tfstate
```

**Manual backup before risky operations:**
```bash
terraform state pull > "backup-$(date +%Y%m%d-%H%M%S).tfstate"
```

### Recovery Procedures

**If state is corrupted:**
1. Stop all Terraform operations immediately
2. Restore from the most recent backup or S3 version
3. Push the restored state: `terraform state push restored-state.tfstate`
4. Verify with `terraform plan`

**If state is lost entirely:**
1. Write Terraform config matching current infrastructure
2. Import every resource: `terraform import <resource_address> <resource_id>`
3. Run `terraform plan` to verify zero diff
4. This is tedious but the only safe recovery path

---

## 5. Troubleshooting Common Issues

### "Error acquiring the state lock"
- Another operation is running, OR a previous operation crashed
- Check CI/CD pipelines and teammates
- If confirmed orphaned: `terraform force-unlock <LOCK_ID>`

### "state snapshot was created by Terraform vX.Y.Z, newer than current"
- Your local Terraform is older than who last modified the state
- Upgrade your Terraform version to match or exceed the state version

### "Resource already exists" during apply
- Resource was created outside Terraform
- Import it: `terraform import <address> <id>`, then re-plan

### Drift detection
```bash
# Detect drift without applying:
terraform plan -detailed-exitcode
# Exit code 0 = no changes, 1 = error, 2 = changes detected

# Refresh state to match reality (doesn't change infrastructure):
terraform apply -refresh-only
```

### State file too large / slow operations
- Split monolithic state into smaller, scoped state files
- Use `-target` for focused operations (sparingly — this is a crutch, not a solution)
- Consider the directory-per-environment or module-per-service pattern
