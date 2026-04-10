# Infrastructure as Code: Terraform and GitOps

## Overview

Infrastructure as Code (IaC) manages infrastructure through version-controlled configuration files. This guide covers IaC principles, Terraform best practices, and GitOps patterns for banking GenAI platforms.

## IaC Principles

```
1. Declarative: Define desired state, not steps to get there
2. Version-controlled: All changes tracked in git
3. Idempotent: Running twice produces same result
4. Testable: Infrastructure tested before production
5. Modular: Reusable components
6. Documented: Resources and dependencies clear
```

## Terraform Banking Infrastructure

```hcl
# modules/genai-platform/main.tf
module "genai_platform" {
  source = "./modules/genai-platform"
  
  environment = var.environment
  region      = var.region
  
  # OpenShift cluster
  cluster_name     = "banking-genai-${var.environment}"
  cluster_version  = "4.14"
  node_count       = var.environment == "production" ? 6 : 3
  
  # PostgreSQL
  db_instance_class = var.environment == "production" ? "db.r6g.2xlarge" : "db.r6g.large"
  db_multi_az      = var.environment == "production"
  db_backup_retention = 30
  
  # Redis
  redis_node_type = var.environment == "production" ? "cache.r6g.large" : "cache.t3.micro"
  redis_cluster   = var.environment == "production"
  
  # Kafka
  kafka_broker_count = var.environment == "production" ? 3 : 1
  
  tags = {
    Project     = "banking-genai"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# outputs.tf
output "cluster_endpoint" {
  value = module.genai_platform.cluster_endpoint
}

output "db_endpoint" {
  value     = module.genai_platform.db_endpoint
  sensitive = true
}
```

## GitOps Pattern

```
Git Repository (Infrastructure Code)
       │
       ▼
  CI Pipeline (terraform plan)
       │
       ▼
  PR Review + Approval
       │
       ▼
  Merge to main
       │
       ▼
  CD Pipeline (terraform apply)
       │
       ▼
  Infrastructure Updated
```

## Cross-References

- **Terraform**: See [terraform.md](terraform.md) for Terraform details
- **Drift Detection**: See [drift-detection.md](drift-detection.md) for state monitoring

## Interview Questions

1. **What are the principles of Infrastructure as Code?**
2. **How do you structure Terraform modules for a banking platform?**
3. **What is GitOps? How does it apply to infrastructure?**
4. **How do you handle Terra state in a team environment?**
5. **How do you test infrastructure changes before applying?**
6. **What happens when Terraform state drifts from reality?**

## Checklist: IaC Best Practices

- [ ] All infrastructure defined as code
- [ ] Code version-controlled and reviewed
- [ ] Terraform state stored remotely with locking
- [ ] Modules reusable and versioned
- [ ] Plan output reviewed before apply
- [ ] Infrastructure changes deployed via CI/CD
- [ ] Drift detection enabled
- [ ] Secrets not in Terraform code
- [ ] Resource tagging consistent
- [ ] Destroy plan tested for cleanup
