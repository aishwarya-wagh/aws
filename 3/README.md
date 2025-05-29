Based on the error messages and your configuration, here's the complete solution to fix the S3 access and secret migration issues:

### Root Cause Analysis
1. **403 Forbidden Error**: The destination account cannot read the S3 bucket (`margeta01-tf-state-us-east-1`) containing the Terraform state.
2. **Broken References**: Typos in resource names (`aws_secret%manager_secret` should be `aws_secretsmanager_secret`).
3. **Incorrect Output References**: The outputs in the remote state don't match what's being referenced.

---

### Step 1: Fix Source Account Configuration
Update the **source account's** Terraform to properly output required values:

```hcl
# source_account/main.tf
data "aws_secretsmanager_secret" "source" {
  name = "/airflow/vault/snowflake/divs/maestro/box"
}

data "aws_secretsmanager_secret_version" "source" {
  secret_id = data.aws_secretsmanager_secret.source.id
}

output "secret_arn" {
  value = data.aws_secretsmanager_secret.source.arn
}

output "secret_value" {
  value     = data.aws_secretsmanager_secret_version.source.secret_string
  sensitive = true
}

output "kms_key_arn" {
  value = data.aws_secretsmanager_secret.source.kms_key_id
}
```

---

### Step 2: Add S3 Bucket Policy (in Source Account)
Create this policy in the **source account** that owns the S3 bucket:

```hcl
# source_account/s3_policy.tf
resource "aws_s3_bucket_policy" "allow_destination_read" {
  bucket = "margeta01-tf-state-us-east-1" # Your state bucket name

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          AWS = "arn:aws:iam::168712969681:root" # Destination account ID
        },
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ],
        Resource = [
          "arn:aws:s3:::margeta01-tf-state-us-east-1",
          "arn:aws:s3:::margeta01-tf-state-us-east-1/margeta01/us-east-1/*"
        ]
      }
    ]
  })
}
```

---

### Step 3: Fix Destination Configuration
Correct the destination account's Terraform files:

```hcl
# destination_account/imports.tf
data "terraform_remote_state" "source" {
  backend = "s3"
  config = {
    bucket         = "margeta01-tf-state-us-east-1"
    key            = "margeta01/us-east-1/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "margeta01-tf-locks-us-east-1"
    role_arn       = "arn:aws:iam::SOURCE_ACCOUNT_ID:role/terraform-read-role" # If cross-account
  }
}

# destination_account/secrets.tf
resource "aws_secretsmanager_secret" "airflow" {
  name        = "/airflow/vault/snowflake/divs/maestro/box/airflow"
  kms_key_id  = data.terraform_remote_state.source.outputs.kms_key_arn
  tags = {
    Consumer = "airflow"
  }
}

resource "aws_secretsmanager_secret_version" "airflow" {
  secret_id     = aws_secretsmanager_secret.airflow.id
  secret_string = data.terraform_remote_state.source.outputs.secret_value
}
```

---

### Step 4: Add KMS Policy (if encrypted)
If the state bucket uses KMS encryption, add this in the **source account**:

```hcl
# source_account/kms_policy.tf
resource "aws_kms_key_policy" "state_bucket" {
  key_id = "alias/your-kms-key-alias" # Replace with actual KMS key

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Sid    = "AllowDestinationAccount",
        Effect = "Allow",
        Principal = {
          AWS = "arn:aws:iam::168712969681:root" # Destination account
        },
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey"
        ],
        Resource = "*"
      }
    ]
  })
}
```

---

### Verification Steps
1. **Apply source changes first**:
   ```bash
   cd source_account/
   terraform init && terraform apply
   ```

2. **Verify bucket policy exists**:
   ```bash
   aws s3api get-bucket-policy --bucket margeta01-tf-state-us-east-1
   ```

3. **Run destination Terraform**:
   ```bash
   cd destination_account/
   terraform init && terraform plan
   ```

---

### Common Fixes for Shown Errors
| Error | Solution |
|-------|----------|
| `403 Forbidden` | Add correct S3 bucket policy and KMS policy |
| `No such file` | Verify exact S3 key path matches remote state |
| `Invalid resource` | Fix typos (`aws_secret%manager` → `aws_secretsmanager`) |
| `Missing outputs` | Ensure source outputs match destination references |

Would you like me to provide the exact IAM role configuration for cross-account access?
