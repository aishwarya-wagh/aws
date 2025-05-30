You're right - the `role_arn` parameter isn't valid in the `terraform_remote_state` data source configuration. Let me provide the correct approach for cross-account state access and secret copying entirely through Terraform:

## Correct Cross-Account Secret Copy with Terraform

### 1. Source Account Setup (Account A)
```hcl
# outputs.tf
output "secret_value" {
  value     = data.aws_secretsmanager_secret_version.source.secret_string
  sensitive = true
}

output "source_kms_arn" {
  value = data.aws_secretsmanager_secret.source.kms_key_id
}
```

### 2. Destination Account (Account B) - Proper Implementation
```hcl
# providers.tf
provider "aws" {
  region = "us-east-1"
  alias  = "destination"
}

provider "aws" {
  region = "us-east-1"
  alias  = "source"
  assume_role {
    role_arn     = "arn:aws:iam::ACCOUNT_A_ID:role/terraform-read-role"
    session_name = "cross_account_secret_copy"
  }
}

# Get source secret directly (no remote state needed)
data "aws_secretsmanager_secret" "source" {
  provider = aws.source
  name     = "airflow/service/box"
}

data "aws_secretsmanager_secret_version" "source" {
  provider  = aws.source
  secret_id = data.aws_secretsmanager_secret.source.id
}

# Create destination secret
resource "aws_secretsmanager_secret" "destination" {
  provider    = aws.destination
  name        = "new/path/for/secret"
  kms_key_id  = aws_kms_key.destination_key.arn
}

resource "aws_secretsmanager_secret_version" "destination" {
  provider      = aws.destination
  secret_id     = aws_secretsmanager_secret.destination.id
  secret_string = data.aws_secretsmanager_secret_version.source.secret_string
}

# KMS Key for destination
resource "aws_kms_key" "destination_key" {
  provider = aws.destination
  policy   = data.aws_iam_policy_document.kms_policy.json
}

data "aws_iam_policy_document" "kms_policy" {
  statement {
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::ACCOUNT_B_ID:root"]
    }
    actions   = ["kms:*"]
    resources = ["*"]
  }
}

# Consumer access policy
resource "aws_iam_policy" "consumer_access" {
  provider = aws.destination
  name     = "secret-consumer-access"
  policy   = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Action    = ["secretsmanager:GetSecretValue"],
      Resource  = aws_secretsmanager_secret.destination.arn
    }]
  })
}
```

## Key Fixes and Improvements

1. **Removed Invalid `role_arn`**:
   - Replaced problematic `terraform_remote_state` with direct cross-account secret access
   - Used provider aliases with `assume_role` for clean cross-account access

2. **Simplified Architecture**:
   - Direct secret read from source account
   - No dependency on state files

3. **Proper Provider Configuration**:
   ```hcl
   provider "aws" {
     alias = "source"
     assume_role {
       role_arn = "arn:aws:iam::ACCOUNT_A_ID:role/terraform-read-role"
     }
   }
   ```

4. **Added Missing Components**:
   - KMS key with proper policy
   - Consumer IAM policy

## Required IAM Setup

### In Source Account (Account A):
Create a role with this trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::ACCOUNT_B_ID:root"
    },
    "Action": "sts:AssumeRole",
    "Condition": {}
  }]
}
```

### In Destination Account (Account B):
Ensure your Terraform execution role has:
```json
{
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::ACCOUNT_A_ID:role/terraform-read-role"
}
```

## Verification
1. Initialize Terraform:
   ```bash
   terraform init
   ```

2. Plan/Apply:
   ```bash
   terraform plan
   terraform apply
   ```

3. Verify secret copy:
   ```bash
   aws secretsmanager get-secret-value \
     --secret-id "new/path/for/secret" \
     --region us-east-1
   ```

This solution is 100% Terraform-managed with no CLI or manual steps required. Would you like me to add any specific components like notification triggers or rotation rules?
