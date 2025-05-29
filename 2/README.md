Yes! You can **manage the S3 bucket policy using Terraform** in the **source account** where the state bucket resides. Here's how:

---

### **Option 1: If You Manage the Source Bucket with Terraform**
Add this to your **source account's Terraform configuration** (e.g., `s3_backend.tf`):

```hcl
resource "aws_s3_bucket_policy" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id  # Your existing bucket resource

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          AWS = "arn:aws:iam::DESTINATION_ACCOUNT_ID:root"  # Destination account
        },
        Action = [
          "s3:GetObject",
          "s3:ListBucket",
        ],
        Resource = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/margeta01/us-east-1/*",
        ]
      }
    ]
  })
}
```

**Prerequisite**:  
Ensure you have the bucket defined in Terraform:
```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "your-state-bucket"  # Must match your backend bucket
  # Enable versioning for state safety
  versioning {
    enabled = true
  }
}
```

---

### **Option 2: If the Bucket Exists but Isn’t Managed by Terraform**
Use a **data source** to reference the existing bucket, then attach the policy:

```hcl
data "aws_s3_bucket" "existing_state_bucket" {
  bucket = "your-state-bucket"  # Name of your existing bucket
}

resource "aws_s3_bucket_policy" "grant_destination_access" {
  bucket = data.aws_s3_bucket.existing_state_bucket.id

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect    = "Allow",
        Principal = { AWS = "arn:aws:iam::DESTINATION_ACCOUNT_ID:root" },
        Action    = ["s3:GetObject", "s3:ListBucket"],
        Resource  = [
          data.aws_s3_bucket.existing_state_bucket.arn,
          "${data.aws_s3_bucket.existing_state_bucket.arn}/margeta01/us-east-1/*",
        ]
      }
    ]
  })
}
```

---

### **Key Notes:**
1. **Cross-Account Access**:  
   The policy grants the **destination account** (`DESTINATION_ACCOUNT_ID`) read access to the state files.

2. **KMS Encryption (If Used)**:  
   If the bucket uses KMS, add a **KMS key policy** (in the source account):
   ```hcl
   resource "aws_kms_key_policy" "terraform_state" {
     key_id = aws_kms_key.terraform_state.key_id
     policy = jsonencode({
       Version = "2012-10-17",
       Statement = [
         {
           Sid    = "AllowDestinationAccountToDecrypt",
           Effect = "Allow",
           Principal = {
             AWS = "arn:aws:iam::DESTINATION_ACCOUNT_ID:root"
           },
           Action = [
             "kms:Decrypt",
             "kms:DescribeKey",
           ],
           Resource = "*"
         }
       ]
     })
   }
   ```

3. **Apply Carefully**:  
   Run `terraform plan` to verify changes before applying:
   ```bash
   terraform plan -target=aws_s3_bucket_policy.terraform_state
   terraform apply -target=aws_s3_bucket_policy.terraform_state
   ```

---

### **Verification**
After applying:
1. Check the bucket policy in AWS Console:  
   ```bash
   aws s3api get-bucket-policy --bucket your-state-bucket
   ```
2. Test access from the destination account:  
   ```bash
   aws s3 ls s3://your-state-bucket/margeta01/us-east-1/ --profile destination-account
   ```

---

### **Troubleshooting**
- **Error: "Bucket policy too large"** → Compact the policy or use IAM roles instead.
- **"Access Denied"** → Verify the KMS key policy (if encryption is enabled).

Would you like me to provide the full Terraform module for managing state bucket + KMS policies together?
