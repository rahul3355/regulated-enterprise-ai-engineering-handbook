# Terraform: Modules, State Management, and Workspaces

## Overview

Terraform manages infrastructure through declarative configuration. This guide covers module design, state management, and workspace strategies for banking environments.

## Module Structure

```
modules/
└── banking-genai/
    ├── main.tf          # Resources
    ├── variables.tf     # Input variables
    ├── outputs.tf       # Output values
    ├── providers.tf     # Provider configuration
    └── README.md        # Documentation

environments/
├── dev/
│   ├── main.tf
│   ├── variables.tfvars
│   └── backend.tf
├── staging/
│   ├── main.tf
│   ├── variables.tfvars
│   └── backend.tf
└── production/
    ├── main.tf
    ├── variables.tfvars
    └── backend.tf
```

## State Management

```hcl
# backend.tf - Remote state with locking
terraform {
  backend "s3" {
    bucket         = "banking-terraform-state"
    key            = "genai-platform/production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "banking-terraform-locks"
    acl            = "private"
    
    # Prevent accidental deletion
    workspace_key_prefix = "env"
  }
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "banking-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Workspaces

```bash
# Workspace management
terraform workspace new dev
terraform workspace new staging
terraform workspace new production

terraform workspace select production
terraform plan -var-file="production.tfvars"
terraform apply -var-file="production.tfvars"

# List workspaces
terraform workspace list

# Show current workspace
terraform workspace show
```

## Cross-References

- **IaC**: See [infrastructure-as-code.md](infrastructure-as-code.md) for principles
- **Drift Detection**: See [drift-detection.md](drift-detection.md) for state monitoring

## Interview Questions

1. **How do you structure Terraform modules for multiple environments?**
2. **How do you manage Terraform state in a team?**
3. **What are Terraform workspaces? When do you use them?**
4. **How do you handle secrets in Terraform?**
5. **What happens when two people run terraform apply simultaneously?**
6. **How do you import existing infrastructure into Terraform?**

## Checklist: Terraform Best Practices

- [ ] Remote state backend configured (S3 + DynamoDB)
- [ ] State encryption enabled
- [ ] State locking configured
- [ ] Modules versioned and reusable
- [ ] Variables validated with constraints
- [ ] Outputs documented
- [ ] Plan reviewed before apply
- [ ] No secrets in state files
- [ ] Workspace strategy chosen (or separate state files)
- [ ] terraform fmt and validate in CI
