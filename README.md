# Copying Secrets Between Paths in Different Accounts with Terraform

To copy a secret from one path to another (especially across different AWS accounts) and manage KMS access, you'll need to use a combination of AWS Secrets Manager and KMS resources in Terraform. Here's how to accomplish this for your scenario where the original path is 'airflow/service/box':

## 1. Basic Approach for Cross-Account Secret Copy

```hcl
# In the source account (where the original secret exists)
data "aws_secretsmanager_secret" "original" {
  name = "airflow/service/box"
}

data "aws_secretsmanager_secret_version" "original" {
  secret_id = data.aws_secretsmanager_secret.original.id
}

# In the destination account (where you want to copy the secret to)
resource "aws_secretsmanager_secret" "copy" {
  name = "new/path/for/secret" # Change this to your desired path
  kms_key_id = aws_kms_key.secrets_key.arn # Your KMS key for encryption
}

resource "aws_secretsmanager_secret_version" "copy" {
  secret_id     = aws_secretsmanager_secret.copy.id
  secret_string = data.aws_secretsmanager_secret_version.original.secret_string
}
```

## 2. Cross-Account Access with KMS

For cross-account access, you need to:

1. Set up KMS key policy in the destination account to allow the source account to use the key
2. Set up a consumer tag for access control

```hcl
# KMS Key in destination account
resource "aws_kms_key" "secrets_key" {
  description             = "KMS key for secrets encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  
  policy = data.aws_iam_policy_document.kms_policy.json
}

data "aws_iam_policy_document" "kms_policy" {
  # Allow root account (adjust with your account ID)
  statement {
    sid    = "EnableRoot"
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::DESTINATION_ACCOUNT_ID:root"]
    }
    actions   = ["kms:*"]
    resources = ["*"]
  }
  
  # Allow source account to use this key for secrets
  statement {
    sid    = "AllowSourceAccount"
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::SOURCE_ACCOUNT_ID:root"]
    }
    actions = [
      "kms:Encrypt",
      "kms:Decrypt",
      "kms:ReEncrypt*",
      "kms:GenerateDataKey*",
      "kms:DescribeKey"
    ]
    resources = ["*"]
  }
  
  # Allow specific consumers via tags
  statement {
    sid    = "AllowTaggedConsumers"
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["*"]
    }
    actions = [
      "kms:Decrypt",
      "kms:DescribeKey"
    ]
    resources = ["*"]
    condition {
      test     = "StringEquals"
      variable = "aws:ResourceTag/Consumer"
      values   = ["airflow-service"]
    }
  }
}

# Add consumer tag to the KMS key
resource "aws_kms_alias" "secrets_key" {
  name          = "alias/secrets-key-for-airflow"
  target_key_id = aws_kms_key.secrets_key.key_id
}

resource "aws_kms_key_policy" "secrets_key" {
  key_id = aws_kms_key.secrets_key.id
  policy = data.aws_iam_policy_document.kms_policy.json
}
```

## 3. Managing the Secret with Consumer Tag

```hcl
resource "aws_secretsmanager_secret" "copy" {
  name        = "new/path/for/secret"
  description = "Copied from airflow/service/box"
  kms_key_id  = aws_kms_key.secrets_key.arn
  
  tags = {
    Consumer = "airflow-service" # This tag enables access via KMS policy
  }
}
```

## 4. Complete Cross-Account Solution

For a complete cross-account solution, you'll need:

1. Terraform in source account to read the secret
2. Terraform in destination account to create the new secret
3. Appropriate IAM roles that allow the copy operation

Here's how to set up the IAM permissions:

```hcl
# In source account - IAM policy to allow reading the secret
resource "aws_iam_policy" "read_secret" {
  name        = "ReadAirflowServiceBox"
  description = "Allow reading airflow/service/box secret"
  
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action   = ["secretsmanager:GetSecretValue"],
        Effect   = "Allow",
        Resource = "arn:aws:secretsmanager:REGION:SOURCE_ACCOUNT_ID:secret:airflow/service/box-*"
      },
      {
        Action   = ["secretsmanager:DescribeSecret"],
        Effect   = "Allow",
        Resource = "arn:aws:secretsmanager:REGION:SOURCE_ACCOUNT_ID:secret:airflow/service/box-*"
      }
    ]
  })
}

# In destination account - IAM policy to allow writing the new secret
resource "aws_iam_policy" "write_secret" {
  name        = "WriteNewSecret"
  description = "Allow creating new secret"
  
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action   = [
          "secretsmanager:CreateSecret",
          "secretsmanager:PutSecretValue",
          "secretsmanager:UpdateSecret"
        ],
        Effect   = "Allow",
        Resource = "arn:aws:secretsmanager:REGION:DESTINATION_ACCOUNT_ID:secret:new/path/for/secret-*"
      },
      {
        Action   = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey"
        ],
        Effect   = "Allow",
        Resource = aws_kms_key.secrets_key.arn
      }
    ]
  })
}
```

## Important Notes:

1. Replace placeholder values (SOURCE_ACCOUNT_ID, DESTINATION_ACCOUNT_ID, REGION) with your actual values
2. The KMS key policy is critical for cross-account access
3. The consumer tag "airflow-service" must match in both the KMS policy and the secret tags
4. You may need to set up additional IAM roles for the actual copy operation between accounts
5. Consider using AWS STS for temporary credentials if doing this programmatically

Would you like me to elaborate on any specific part of this process?
