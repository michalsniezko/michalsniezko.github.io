---
layout: default
title: Terraform Project Structure & Remote Backend
parent: Infrastructure as Code
nav_order: 1
---

## Terraform Project Structure & Remote Backend

**Flow:** Engineer writes HCL → commits to Git → CI runs `terraform plan` → reviewer approves → CI runs `terraform apply` → state is stored remotely in S3, locked via DynamoDB.

Without a remote backend, the state file (`terraform.tfstate`) lives on whoever's laptop ran `apply` last. A second engineer runs `apply` from their machine with stale state and overwrites infrastructure. DynamoDB locking prevents concurrent applies from corrupting state.

### Project Layout

```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── ecs-service/
│       └── ...
└── backend.tf
```

### Remote Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "acme-terraform-state"
    key            = "environments/dev/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}
```

### DynamoDB Lock Table (One-Time Setup)

```hcl
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Initializing

```bash
cd infrastructure/environments/dev
terraform init -backend-config="key=environments/dev/terraform.tfstate"
```

> **Security Note:** The S3 state file contains every resource attribute in plaintext — including database passwords, private keys, and API tokens that Terraform manages. Enable S3 bucket versioning (to recover from corruption), enforce SSE-S3 or SSE-KMS encryption, and restrict bucket access to the CI service role only. Never commit `.tfstate` to Git.
