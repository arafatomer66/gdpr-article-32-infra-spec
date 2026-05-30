<div align="center">

# GDPR Article 32
### Infrastructure Specification

**For engineers, by engineers.**
Article 32 is not a compliance document. It is an infrastructure specification.
Every clause maps to a concrete control. Every control requires verifiable evidence.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Controls](https://img.shields.io/badge/Controls-9-green.svg)](#controls)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/arafatomer66/gdpr-article-32-infra-spec/pulls)

</div>

---

## The Core Idea

```
POLICY:    "We encrypt personal data."

EVIDENCE:  "Here is the KMS key ID, its 90-day rotation schedule,
            a CloudTrail entry showing last key use, and the role
            policy proving only the identity service can decrypt."
```

Auditors do not read policy documents. They request database schema dumps, CloudTrail excerpts, CI/CD pipeline configs, DR test logs, and pen test reports with remediation tickets. This repository gives you all of it.

---

## Article 32 → Control Mapping

```
Article 32(1)(a)  pseudonymisation and encryption of personal data
  ├── Control 01  Pseudonymisation
  ├── Control 02  Encryption at Rest (KMS)
  └── Control 03  Application-Layer Encryption

Article 32(1)(b)  ongoing confidentiality, integrity, availability and resilience
  ├── Control 04  Automatic Session Logoff
  └── Control 05  Unique User Identification (IRSA)

Article 32(1)(c)  ability to restore availability and access in a timely manner
  ├── Control 06  Multi-AZ Backup Infrastructure
  └── Control 07  Tested Disaster Recovery

Article 32(1)(d)  regular testing, assessing and evaluating effectiveness
  ├── Control 08  Automated Vulnerability Scanning
  └── Control 09  Annual Penetration Testing
```

---

## Controls

| # | Control | Clause | Effort | Time |
|---|---------|--------|--------|------|
| [01](controls/01-pseudonymisation.md) | Pseudonymisation | 32(1)(a) | Medium | 3–5 days |
| [02](controls/02-encryption-at-rest.md) | Encryption at Rest (KMS) | 32(1)(a) | Low | 30 min – 1 day |
| [03](controls/03-application-layer-encryption.md) | Application-Layer Encryption | 32(1)(a) | High | 3–5 days |
| [04](controls/04-session-management.md) | Automatic Session Logoff | 32(1)(b) | Low | 1–2 days |
| [05](controls/05-unique-identity.md) | Unique User Identification | 32(1)(b) | Medium | 2–4 days |
| [06](controls/06-multi-az-backup.md) | Multi-AZ Backup Infrastructure | 32(1)(c) | Medium | 1–3 days |
| [07](controls/07-disaster-recovery.md) | Tested Disaster Recovery | 32(1)(c) | Medium | 1 week |
| [08](controls/08-vulnerability-scanning.md) | Automated Vulnerability Scanning | 32(1)(d) | Low | 1–2 days |
| [09](controls/09-penetration-testing.md) | Annual Penetration Testing | 32(1)(d) | High | 4–8 weeks |

Each control file contains:
- The exact Article 32 clause it satisfies
- Working implementation code (Python, TypeScript, SQL, Terraform, YAML)
- Verification commands to confirm the control is active
- An audit evidence table — what to produce when asked
- Common mistakes that break the control silently

---

## Audit Evidence Cheatsheet

| Auditor Question | What They Want to See |
|-----------------|----------------------|
| Can you re-identify pseudonymised data? | DB role grants — only the identity service can JOIN the identifier table |
| Can your DBAs read customer data? | Direct query returning `vault:v1:...` ciphertext + Vault policy proof |
| Who accessed this user's record? | CloudTrail logs with unique IAM role ARNs per service, no shared accounts |
| What is your RTO? Show me proof. | Monthly DR test logs: snapshot ID, restore duration, row count verification |
| When was your last pen test? | CREST report < 12 months + remediation tickets closed within SLA |

---

## Implementation Order

Start here. Each phase builds on the previous.

```
Phase 1 — 1–2 days
  ├── Control 02  KMS customer-managed key + automatic rotation
  └── Control 08  Trivy in CI/CD pipeline blocking CRITICAL/HIGH

Phase 2 — 3–5 days
  ├── Control 04  15-min inactivity timeout, 8-hr absolute session limit
  └── Control 05  IRSA — one IAM role per service, no shared credentials

Phase 3 — 1 week
  ├── Control 01  Pseudonymisation schema + role-based access
  └── Control 06  Multi-AZ RDS + S3 cross-region replication

Phase 4 — 1–2 weeks
  ├── Control 03  Application-layer encryption via HashiCorp Vault transit
  └── Control 07  First DR restore test + documented RTO/RPO

Phase 5 — Schedule 4–6 weeks out
  └── Control 09  CREST-certified penetration test engagement
```

**Full coverage: 2–4 weeks active work, pen test scheduled in parallel.**

---

## Stack Coverage

This specification uses the following stack. Controls are adaptable to other providers.

| Layer | Technology |
|-------|-----------|
| Cloud | AWS (KMS, RDS, S3, IAM, CloudTrail, EKS) |
| Infrastructure as Code | Terraform |
| Secret Management | HashiCorp Vault |
| Container Scanning | Trivy |
| CI/CD | GitHub Actions |
| Database | PostgreSQL |
| Application | Python / TypeScript examples |

---

## What This Is Not

- Not a legal opinion. Consult a GDPR-qualified lawyer for interpretations specific to your organisation.
- Not exhaustive. Article 32 also requires organisational measures (training, access reviews, incident response). This repository covers the technical controls only.
- Not AWS-only. The control objectives apply universally. The implementations use AWS as the concrete example.

---

## References

- [GDPR Article 32 — Official Text](https://gdpr-info.eu/art-32-gdpr/)
- [EDPB Guidelines on Personal Data Breach Notification](https://edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-92022-personal-data-breach-notification-under_en)
- [AWS KMS Best Practices](https://docs.aws.amazon.com/kms/latest/developerguide/best-practices.html)
- [CREST Penetration Testing Standards](https://www.crest-approved.org/)
- [NIST SP 800-53 Rev 5 — Control Mapping Reference](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [OWASP Testing Guide v4.2](https://owasp.org/www-project-web-security-testing-guide/)

---

## Contributing

Found an error, a better implementation, or a missing control? Open a PR.

Useful contributions:
- Corrections to CLI flags, API names, or Terraform resource schemas
- Equivalent implementations for Azure, GCP, or non-AWS stacks
- Real-world audit evidence examples (with sensitive data removed)
- Additional controls not covered here (e.g., Article 32(2) risk assessment process)

---

## Related

**[GDPR Engineer Playbook →](https://github.com/arafatomer66/gdpr-engineer-playbook)**
The 13 other GDPR articles engineers own — Art 5, 6, 9, 17, 20, 25, 28, 30, 33, 34, 35, 37, 83.
Same format: legal text → code → audit evidence.

---

<div align="center">

*Part of a series on GDPR for engineers*
[This Repo: Article 32 Spec](https://github.com/arafatomer66/gdpr-article-32-infra-spec) · [GDPR Engineer Playbook](https://github.com/arafatomer66/gdpr-engineer-playbook)

</div>
