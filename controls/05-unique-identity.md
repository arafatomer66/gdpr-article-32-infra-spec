# Control 05 — Unique User Identification (IRSA)

**Article:** 32(1)(b)
**Effort:** Medium
**Implementation Time:** 2–4 days

---

## Requirement

> "...the ongoing confidentiality, integrity, availability and resilience of processing systems"

Every action on personal data must be traceable to a specific, unique identity. Shared service accounts make audit trails meaningless — "the application" is not an auditable actor.

---

## The Problem with Shared Accounts

```
# Shared account audit log — useless
2024-01-15 09:12:33 | app-service | SELECT * FROM person_identifiers
2024-01-15 09:45:01 | app-service | UPDATE orders SET status='cancelled'
2024-01-15 10:03:22 | app-service | DELETE FROM person_identifiers WHERE ...
```

Three different services, one shared identity. Which service deleted that record? Unknown.

```
# IRSA audit log — attributable
2024-01-15 09:12:33 | arn:aws:iam::123:role/identity-service-pod  | GET /secrets/person-key
2024-01-15 09:45:01 | arn:aws:iam::123:role/orders-service-pod    | kms:Decrypt
2024-01-15 10:03:22 | arn:aws:iam::123:role/gdpr-deletion-job     | s3:DeleteObject
```

Each service has its own IAM role. Every action is attributable.

---

## Implementation: IAM Roles for Service Accounts (IRSA)

IRSA binds a Kubernetes ServiceAccount to an AWS IAM Role using OIDC federation. The pod gets temporary, auto-rotating credentials scoped to exactly the permissions it needs.

### Step 1: Enable OIDC on your EKS Cluster

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster main-cluster \
  --approve
```

### Step 2: Create IAM Roles per Service

```hcl
# Terraform — one role per service, principle of least privilege

data "aws_iam_openid_connect_provider" "eks" {
  url = data.aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_role" "identity_service" {
  name = "identity-service-pod"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = data.aws_iam_openid_connect_provider.eks.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${data.aws_iam_openid_connect_provider.eks.url}:sub" =
            "system:serviceaccount:production:identity-service"
        }
      }
    }]
  })
}

# Grant only what this service needs
resource "aws_iam_role_policy" "identity_service_kms" {
  name = "identity-service-kms"
  role = aws_iam_role.identity_service.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["kms:Decrypt", "kms:GenerateDataKey"]
      Resource = aws_kms_key.personal_data.arn
    }]
  })
}
```

### Step 3: Bind ServiceAccount to IAM Role

```yaml
# kubernetes/identity-service-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: identity-service
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/identity-service-pod
```

```yaml
# kubernetes/identity-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: identity-service
spec:
  selector:
    matchLabels:
      app: identity-service
  template:
    metadata:
      labels:
        app: identity-service
    spec:
      serviceAccountName: identity-service  # binds the IRSA role
      containers:
        - name: identity-service
          image: identity-service:latest
          # No AWS credentials in env vars — IRSA injects them automatically
```

### Step 4: Verify — No Shared Credentials

```bash
# Confirm no AWS access keys exist for application users
aws iam list-users --query 'Users[*].UserName' --output text | \
  xargs -I {} aws iam list-access-keys --user-name {} \
    --query 'AccessKeyMetadata[?Status==`Active`].UserName' \
    --output text

# Expected: no output (no active access keys for app users)

# Confirm the pod uses IRSA
kubectl exec -n production deploy/identity-service -- \
  aws sts get-caller-identity

# Expected:
# {
#   "UserId": "...",
#   "Account": "123456789012",
#   "Arn": "arn:aws:sts::123456789012:assumed-role/identity-service-pod/..."
# }
```

---

## CloudTrail Verification

```bash
# See all actions taken by the identity-service role
aws cloudtrail lookup-events \
  --lookup-attributes \
    AttributeKey=Username,AttributeValue=identity-service-pod \
  --query 'Events[*].{Time:EventTime,Action:EventName,Resource:Resources[0].ResourceName}' \
  --output table
```

---

## Extending to Human Users

Apply the same principle to human access:

| Access Type | Implementation |
|------------|---------------|
| DBA access | Individual IAM users + MFA, no shared `dba` account |
| Developer prod access | SSO (e.g., AWS IAM Identity Center) — no long-term credentials |
| CI/CD pipeline | OIDC federation from GitHub Actions — no static access keys |
| Emergency access | Break-glass IAM role with CloudTrail alert on assumption |

---

## Audit Evidence

| Auditor Question | Evidence to Produce |
|-----------------|-------------------|
| Who accessed personal data? | CloudTrail showing unique role ARNs per action |
| Do you use shared service accounts? | IAM user list showing no shared accounts; IRSA config |
| Are credentials rotated? | IRSA uses temporary credentials (1-hour TTL, auto-rotated by AWS STS) |
| What access does the app have? | IAM role policy showing least-privilege permissions |

---

## Common Mistakes

- **One IAM role for all services** — still a shared identity; each service needs its own role
- **Hardcoded `AWS_ACCESS_KEY_ID` in environment variables** — static credentials, no rotation, leaks in logs
- **`Action: "*"` in IAM policies** — use exact action names matched to what the service actually needs
- **Not enabling CloudTrail in all regions** — gaps in audit coverage

---

[← Session Management](04-session-management.md) | [Next: Multi-AZ Backup →](06-multi-az-backup.md)
