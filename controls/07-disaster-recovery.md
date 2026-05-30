# Control 07 — Tested Disaster Recovery

**Article:** 32(1)(c)
**Effort:** Medium
**Implementation Time:** 1 week (first test), then monthly

---

## Requirement

> "...the ability to restore the availability and access to personal data in a timely manner in the event of a physical or technical incident"

Having backups (Control 06) is necessary but insufficient. Article 32 requires the ability **to restore** — which means the restore process must be tested, measured, and documented. An untested backup is not a backup.

---

## Key Metrics

| Metric | Definition | Your Target |
|--------|-----------|-------------|
| **RPO** (Recovery Point Objective) | Maximum data loss acceptable — "how old can the data be?" | ≤ 5 minutes |
| **RTO** (Recovery Time Objective) | Maximum downtime acceptable — "how long can we be down?" | ≤ 4 hours |

Document these targets. Auditors expect defined numbers, not "we'll restore as fast as we can."

---

## Monthly DR Test Runbook

Run this in a non-production account. Document every step.

### Step 1: Identify Test Scope

```
Date:           2024-01-15
Engineer:       [Name]
Snapshot used:  rds:main-db-2024-01-14-02-10
Target account: dr-test-account (123456789999)
```

### Step 2: Restore RDS Snapshot

```bash
# Start the restore — record exact start time
RESTORE_START=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
echo "Restore started at: $RESTORE_START"

aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier dr-test-$(date +%Y%m%d) \
  --db-snapshot-identifier rds:main-db-2024-01-14-02-10 \
  --db-instance-class db.t3.medium \
  --no-multi-az \
  --kms-key-id arn:aws:kms:ap-south-1:123456789999:key/... \
  --region ap-south-1

# Wait for available status
aws rds wait db-instance-available \
  --db-instance-identifier dr-test-$(date +%Y%m%d)

RESTORE_END=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
echo "Restore completed at: $RESTORE_END"
```

### Step 3: Verify Data Integrity

```bash
# Connect to restored instance
ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier dr-test-$(date +%Y%m%d) \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)

psql -h $ENDPOINT -U admin -d maindb <<'EOF'
-- Verify record counts match expectations
SELECT
  'person_identifiers' AS table_name,
  COUNT(*) AS row_count
FROM person_identifiers
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
UNION ALL
SELECT 'order_events', COUNT(*) FROM order_events;

-- Verify most recent record timestamp
SELECT MAX(created_at) AS latest_record FROM orders;

-- Spot-check data integrity
SELECT person_uuid, email FROM person_identifiers LIMIT 5;
EOF
```

### Step 4: Measure and Record

```bash
# Calculate RTO
START_EPOCH=$(date -d "$RESTORE_START" +%s)
END_EPOCH=$(date -d "$RESTORE_END" +%s)
RTO_SECONDS=$(( END_EPOCH - START_EPOCH ))
echo "RTO: ${RTO_SECONDS} seconds ($(( RTO_SECONDS / 60 )) minutes)"

# Calculate RPO from snapshot timestamp
SNAPSHOT_TIME=$(aws rds describe-db-snapshots \
  --db-snapshot-identifier rds:main-db-2024-01-14-02-10 \
  --query 'DBSnapshots[0].SnapshotCreateTime' \
  --output text)
echo "Snapshot time: $SNAPSHOT_TIME"
echo "RPO: gap between snapshot and incident time"
```

### Step 5: Cleanup

```bash
# Delete test instance — do not leave running
aws rds delete-db-instance \
  --db-instance-identifier dr-test-$(date +%Y%m%d) \
  --skip-final-snapshot
```

---

## DR Test Log Template

```markdown
## DR Test — 2024-01-15

**Engineer:** Jane Smith
**Snapshot:** rds:main-db-2024-01-14-02-10
**Snapshot Age:** 23 hours 50 minutes

### Results
| Metric | Target | Actual | Pass? |
|--------|--------|--------|-------|
| RTO    | ≤ 4 hours | 1h 23m | ✅ |
| RPO    | ≤ 5 min   | 4m 10s | ✅ |

### Data Verification
- person_identifiers: 48,231 rows ✅
- orders: 142,890 rows ✅
- Latest record: 2024-01-14 02:05:42 UTC ✅

### Issues Found
- None

### Next Test
2024-02-15 (monthly cadence)
```

---

## Automated DR Test (CI/CD)

For teams with higher maturity, automate the monthly test:

```yaml
# .github/workflows/dr-test.yml
name: Monthly DR Test

on:
  schedule:
    - cron: '0 2 15 * *'  # 15th of every month at 02:00 UTC

jobs:
  dr-test:
    runs-on: ubuntu-latest
    environment: dr-test

    steps:
      - name: Restore latest snapshot
        run: |
          SNAPSHOT=$(aws rds describe-db-snapshots \
            --db-instance-identifier main-db \
            --snapshot-type automated \
            --query 'sort_by(DBSnapshots, &SnapshotCreateTime)[-1].DBSnapshotIdentifier' \
            --output text)
          # ... restore, verify, measure, cleanup

      - name: Post results to Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            { "text": "DR Test complete. RTO: ${{ steps.restore.outputs.rto }}. RPO: ${{ steps.restore.outputs.rpo }}." }
```

---

## Audit Evidence

| Auditor Question | Evidence to Produce |
|-----------------|-------------------|
| What is your RTO? | DR test logs showing measured restore times over last 12 months |
| Have you actually tested restores? | Dated test logs with engineer names, snapshot IDs, row counts |
| What was the last test date? | Test log from < 30 days ago |
| What issues were found and fixed? | Test log "Issues Found" section + remediation tickets |

---

## Common Mistakes

- **Testing in production** — a failed restore test should not affect live data
- **Testing only once** — quarterly minimum, monthly recommended; DR capability degrades silently
- **Not verifying data** — confirming "the instance started" is not a restore test; verify row counts and data integrity
- **No documented RTO/RPO targets** — without targets, you cannot demonstrate you met them

---

[← Multi-AZ Backup](06-multi-az-backup.md) | [Next: Vulnerability Scanning →](08-vulnerability-scanning.md)
