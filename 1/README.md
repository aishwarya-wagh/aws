Based on your file structure and the task of migrating secrets between paths in different accounts, here's how to organize your Terraform configuration:

## Recommended Structure for Secrets Migration

```
terraform-manifest-vault-secrets-migration/
├── common/                  # Shared modules/resources
│   ├── kms.tf               # KMS key configuration
│   └── providers.tf         # Common provider setup
├── margeta01/               # Source account configs
│   ├── us-east-1/
│   │   ├── backend.tf       # Remote state for source
│   │   ├── data.tf          # Data sources for secrets
│   │   ├── outputs.tf       # Output secret values/ARNs
│   │   └── providers.tf
├── destination-account/      # Target account configs
│   ├── us-east-1/
│   │   ├── backend.tf       # Remote state for destination
│   │   ├── imports.tf       # Import source outputs
│   │   ├── secrets.tf       # New secret resources
│   │   └── providers.tf
└── scripts/                 # Migration helper scripts
```

## Implementation Steps

1. **In source account (margeta01/us-east-1)**:

```hcl
# data.tf
data "aws_secretsmanager_secret" "source" {
  name = "airflow/service/box"
}

data "aws_secretsmanager_secret_version" "source" {
  secret_id = data.aws_secretsmanager_secret.source.id
}

# outputs.tf
output "secret_arn" {
  value = data.aws_secretsmanager_secret.source.arn
}

output "secret_value" {
  value     = data.aws_secretsmanager_secret_version.source.secret_string
  sensitive = true
}
```

2. **In destination account (create new folder structure)**:

```hcl
# imports.tf
data "terraform_remote_state" "source" {
  backend = "s3"
  config = {
    bucket = "your-state-bucket"
    key    = "margeta01/us-east-1/terraform.tfstate"
    region = "us-east-1"
  }
}

# secrets.tf
resource "aws_secretsmanager_secret" "destination" {
  name        = "new/path/for/secret"
  kms_key_id  = module.kms.key_arn
  tags        = {
    Consumer = "airflow-service"
  }
}

resource "aws_secretsmanager_secret_version" "destination" {
  secret_id     = aws_secretsmanager_secret.destination.id
  secret_string = data.terraform_remote_state.source.outputs.secret_value
}
```

## Key Recommendations:

1. **State Isolation**: Maintain separate state files for each account/region
2. **Cross-Account Access**:
   - Configure KMS key policies to allow cross-account access
   - Set up required IAM roles for Terraform to assume
3. **Secret Tagging**: Consistently tag secrets with `Consumer = "airflow-service"` for access control
4. **Execution Flow**:
   - First apply in source account to expose outputs
   - Then apply in destination account to create new secret

Would you like me to provide specific configurations for any of these components, such as the KMS key policy or IAM roles for cross-account access?
