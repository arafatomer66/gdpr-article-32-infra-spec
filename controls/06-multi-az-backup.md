# Control 06 — Multi-AZ Backup Infrastructure

**Article:** 32(1)(c)
**Effort:** Medium
**Implementation Time:** 1–3 days

---

## Requirement

> "...the ability to restore the availability and access to personal data in a timely manner in the event of a physical or technical incident"

Backups must exist. They must be in geographically separated locations. They must be sufficient to meet your stated Recovery Point Objective (RPO).

---

## Backup Coverage Matrix

| Data Store | Backup Method | Retention | RPO |
|-----------|--------------|-----------|-----|
| RDS PostgreSQL | Automated snapshots | 30 days | 5 min (PITR) |
| S3 personal data | Versioning + replication | 90 days | Near-zero |
| Application secrets | Vault snapshots | 30 days | 1 hour |
| Elasticsearch | Snapshot to S3 | 14 days | 1 hour |

---

## Implementation

### RDS: Multi-AZ + Automated Backups

```hcl
resource "aws_db_instance" "main" {
  identifier              = "main-db"
  engine                  = "postgres"
  engine_version          = "15.4"
  instance_class          = "db.t3.medium"
  allocated_storage       = 100
  storage_encrypted       = true
  kms_key_id              = aws_kms_key.personal_data.arn

  # Multi-AZ: synchronous standby in a second AZ
  multi_az                = true

  # Automated backups
  backup_retention_period = 30          # 30 days of daily snapshots
  backup_window           = "02:00-03:00"  # UTC — low-traffic window
  maintenance_window      = "sun:04:00-sun:05:00"

  # Point-in-time recovery — continuous WAL archiving
  # Enables restore to any second within the retention window
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  deletion_protection = true  # prevent accidental drops

  tags = {
    Compliance = "GDPR-Art32"
    DataClass  = "PersonalData"
  }
}
```

### S3: Versioning + Cross-Region Replication

```hcl
# Primary bucket
resource "aws_s3_bucket" "personal_data_primary" {
  bucket = "acme-personal-data-ap-south-1"
}

resource "aws_s3_bucket_versioning" "personal_data_primary" {
  bucket = aws_s3_bucket.personal_data_primary.id
  versioning_configuration { status = "Enabled" }
}

# Replica bucket in a second region
resource "aws_s3_bucket" "personal_data_replica" {
  provider = aws.eu-west-1
  bucket   = "acme-personal-data-eu-west-1"
}

# Replication rule
resource "aws_s3_bucket_replication_configuration" "personal_data" {
  bucket = aws_s3_bucket.personal_data_primary.id
  role   = aws_iam_role.s3_replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.personal_data_replica.arn
      storage_class = "STANDARD_IA"

      encryption_configuration {
        replica_kms_key_id = aws_kms_key.personal_data_replica.arn
      }
    }
  }
}
```

### S3 Lifecycle: Retain 90 Days, Then Archive

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "personal_data" {
  bucket = aws_s3_bucket.personal_data_primary.id

  rule {
    id     = "archive-old-versions"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}
```

### Backup Monitoring: Alert on Missed Backups

AWS does not publish a `BackupRetentionPeriod` CloudWatch metric. Use EventBridge to catch RDS backup failures instead:

```hcl
# Alert when an automated RDS snapshot fails
resource "aws_cloudwatch_event_rule" "rds_backup_failed" {
  name        = "rds-backup-failed"
  description = "Fires when an automated RDS snapshot fails — GDPR Art 32"

  event_pattern = jsonencode({
    source      = ["aws.rds"]
    detail-type = ["RDS DB Snapshot Event"]
    detail = {
      EventCategories = ["backup"]
      Message         = [{ prefix = "Automated snapshot failed" }]
    }
  })
}

resource "aws_cloudwatch_event_target" "rds_backup_failed_sns" {
  rule      = aws_cloudwatch_event_rule.rds_backup_failed.name
  target_id = "notify-ops"
  arn       = aws_sns_topic.ops_alerts.arn
}
```

Also monitor the `SnapshotStorageUsed` metric to confirm snapshots are being created:

```hcl
resource "aws_cloudwatch_metric_alarm" "rds_no_snapshots" {
  alarm_name          = "rds-no-snapshots"
  comparison_operator = "LessThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "SnapshotStorageUsed"
  namespace           = "AWS/RDS"
  period              = 86400
  statistic           = "Maximum"
  threshold           = 0
  alarm_description   = "No RDS snapshot storage in use — backups may be disabled"
  alarm_actions       = [aws_sns_topic.ops_alerts.arn]

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.id
  }
}
```

---

## Verify Backups Exist

```bash
# List recent RDS snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier main-db \
  --snapshot-type automated \
  --query 'DBSnapshots[*].{Time:SnapshotCreateTime, Status:Status, Encrypted:Encrypted}' \
  --output table

# Confirm S3 versioning is active
aws s3api get-bucket-versioning \
  --bucket acme-personal-data-ap-south-1

# Check replication status
aws s3api get-bucket-replication \
  --bucket acme-personal-data-ap-south-1
```

---

## Audit Evidence

| Auditor Question | Evidence to Produce |
|-----------------|-------------------|
| Where are backups stored? | Terraform config showing multi-AZ RDS + S3 cross-region replication |
| How long are backups retained? | `backup_retention_period = 30` + S3 lifecycle policy |
| Are backups encrypted? | Snapshot metadata showing `Encrypted: true` + KMS key ARN |
| When was the last successful backup? | `describe-db-snapshots` output showing timestamps |

---

## Common Mistakes

- **`backup_retention_period = 0`** — disables automated backups entirely; Terraform default is 0
- **Single-AZ RDS** — an AZ outage takes down both primary and backups simultaneously
- **Unencrypted backup buckets** — backup data is still personal data; KMS applies
- **No replication monitoring** — replication can silently fail for weeks without an alert
- **Manual backups only** — manual snapshots require human discipline; automated backups do not

---

[← Unique Identity](05-unique-identity.md) | [Next: Disaster Recovery →](07-disaster-recovery.md)
