# Control 02 — Encryption at Rest (KMS)

**Article:** 32(1)(a)
**Effort:** Low
**Implementation Time:** 30 minutes – 1 day

---

## Requirement

> "...the pseudonymisation and encryption of personal data"

Data stored on disk — databases, object storage, backups, snapshots — must be encrypted. The organization must demonstrate **control over the key material**, not just rely on provider defaults.

---

## AWS-managed vs Customer-managed Keys

| | AWS-managed (SSE-S3) | Customer-managed (CMK) |
|--|---------------------|----------------------|
| Auditor accepts? | Sometimes | **Always** |
| Key rotation control | AWS-controlled | You control (90-day recommended) |
| Key policy / access audit | Limited | Full CloudTrail |
| Cross-account sharing | Not possible | Supported |
| Cost | Free | ~$1/key/month |

**Use customer-managed keys (CMK).** AWS-managed keys cannot demonstrate organizational control — a critical gap during audits.

---

## Implementation

### Create a CMK with Automatic Rotation

```hcl
# Terraform

resource "aws_kms_key" "personal_data" {
  description             = "CMK for personal data encryption — GDPR Art 32"
  deletion_window_in_days = 30
  enable_key_rotation     = true   # 90-day automatic rotation
  rotation_period_in_days = 90

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow RDS to use key"
        Effect = "Allow"
        Principal = { Service = "rds.amazonaws.com" }
        Action   = ["kms:GenerateDataKey", "kms:Decrypt"]
        Resource = "*"
      }
    ]
  })

  tags = {
    Name        = "personal-data-cmk"
    Compliance  = "GDPR-Art32"
    DataClass   = "PersonalData"
  }
}

resource "aws_kms_alias" "personal_data" {
  name          = "alias/personal-data-cmk"
  target_key_id = aws_kms_key.personal_data.key_id
}
```

### Attach CMK to RDS

```hcl
resource "aws_db_instance" "main" {
  identifier        = "main-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.medium"
  allocated_storage = 100

  storage_encrypted = true
  kms_key_id        = aws_kms_key.personal_data.arn

  # ... other config
}
```

### Attach CMK to S3 (Uploads, Exports, Backups)

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "personal_data" {
  bucket = aws_s3_bucket.personal_data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.personal_data.arn
    }
    bucket_key_enabled = true  # reduces KMS API call costs
  }
}

# Block public access — always
resource "aws_s3_bucket_public_access_block" "personal_data" {
  bucket                  = aws_s3_bucket.personal_data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## Verifying Key Rotation

```bash
# Confirm rotation is enabled
aws kms get-key-rotation-status \
  --key-id alias/personal-data-cmk

# Expected output:
# {
#   "KeyRotationEnabled": true,
#   "NextRotationDate": "2025-01-15T00:00:00+00:00"
# }
```

---

## CloudTrail: Key Usage Audit

Every KMS operation (Encrypt, Decrypt, GenerateDataKey) is logged automatically in CloudTrail. To query recent decrypt operations:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=Decrypt \
  --start-time 2024-01-01 \
  --query 'Events[*].{Time:EventTime, User:Username, Key:Resources[0].ResourceName}' \
  --output table
```

---

## Audit Evidence

| Auditor Question | Evidence to Produce |
|-----------------|-------------------|
| Are backups encrypted? | `aws rds describe-db-snapshots` showing `Encrypted: true` and your KMS key ARN |
| Who controls the encryption keys? | KMS key policy + key metadata showing your account owns the CMK |
| When was the key last rotated? | `get-key-rotation-status` output + CloudTrail `RotateKey` event |
| Can AWS access our data? | KMS key policy showing AWS service access is scoped and auditable |

---

## Common Mistakes

- **Using SSE-S3 (AES-256) instead of SSE-KMS** — no key rotation control, no CloudTrail per-object audit
- **Enabling encryption after the fact on RDS** — requires snapshot + restore; plan at creation time
- **Not tagging the KMS key** — makes it impossible to tie keys to data classification during audits
- **Setting `deletion_window_in_days` to 7** — too short; 30 days gives recovery time if accidentally scheduled for deletion

---

[← Pseudonymisation](01-pseudonymisation.md) | [Next: Application-Layer Encryption →](03-application-layer-encryption.md)
