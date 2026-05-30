# GDPR Article 32 — Infrastructure Specification

> **For engineers, by engineers.** Article 32 is not a policy document. It is an infrastructure specification. Every clause maps to a concrete control. Every control requires verifiable evidence.

---

## What is Article 32?

GDPR Article 32 mandates that data processors implement **"appropriate technical and organisational measures"** to secure personal data. The regulation does not prescribe exact tools — it defines outcomes. This repository translates those outcomes into engineering controls.

**Legal text → Engineering requirement → Implementation → Audit evidence.**

---

## The 9 Controls at a Glance

| # | Control | Article Clause | Effort |
|---|---------|---------------|--------|
| [01](controls/01-pseudonymisation.md) | Pseudonymisation | 32(1)(a) | Medium |
| [02](controls/02-encryption-at-rest.md) | Encryption at Rest (KMS) | 32(1)(a) | Low |
| [03](controls/03-application-layer-encryption.md) | Application-Layer Encryption | 32(1)(a) | High |
| [04](controls/04-session-management.md) | Automatic Session Logoff | 32(1)(b) | Low |
| [05](controls/05-unique-identity.md) | Unique User Identification | 32(1)(b) | Medium |
| [06](controls/06-multi-az-backup.md) | Multi-AZ Backup Infrastructure | 32(1)(c) | Medium |
| [07](controls/07-disaster-recovery.md) | Tested Disaster Recovery | 32(1)(c) | Medium |
| [08](controls/08-vulnerability-scanning.md) | Automated Vulnerability Scanning | 32(1)(d) | Low |
| [09](controls/09-penetration-testing.md) | Annual Penetration Testing | 32(1)(d) | High |

---

## The Compliance Mindset Shift

```
POLICY:    "We encrypt personal data."
EVIDENCE:  "Here is the KMS key ID, its rotation schedule,
            and a CloudTrail entry showing last key use."
```

Auditors do not read policy documents. They request:
- Database schema dumps
- CloudTrail log excerpts
- CI/CD pipeline configs
- DR test run logs
- Pen test reports with remediation tickets

---

## Quick Reference: Article 32 Text → Control Mapping

```
Article 32(1)(a)  pseudonymisation and encryption of personal data
  → Control 01: Pseudonymisation
  → Control 02: Encryption at Rest
  → Control 03: Application-Layer Encryption

Article 32(1)(b)  ongoing confidentiality, integrity, availability and resilience
  → Control 04: Automatic Session Logoff
  → Control 05: Unique User Identification (IRSA)

Article 32(1)(c)  ability to restore availability and access in a timely manner
  → Control 06: Multi-AZ Backup Infrastructure
  → Control 07: Tested Disaster Recovery

Article 32(1)(d)  regular testing, assessing and evaluating effectiveness
  → Control 08: Automated Vulnerability Scanning
  → Control 09: Annual Penetration Testing
```

---

## Audit Evidence Cheatsheet

| Auditor Question | What They Want to See |
|-----------------|----------------------|
| Can you re-identify pseudonymised data? | DB role grants — only authorized services can JOIN the identifier table |
| Can your DBAs read customer data? | Direct query returning ciphertext + proof decryption keys are in Vault |
| Who accessed this user's record? | CloudTrail logs with unique IAM role ARNs, no shared accounts |
| What is your RTO? Show me proof. | Monthly DR test logs: snapshot ID, restore duration, verification queries |
| When was your last pen test? | CREST/CHECK report < 12 months + remediation tickets for all CRITICAL/HIGH |

---

## Implementation Timeline

| Phase | Controls | Duration |
|-------|----------|----------|
| Phase 1 | KMS key creation (02), Trivy in CI (08) | 1–2 days |
| Phase 2 | Session timeouts (04), IRSA setup (05) | 3–5 days |
| Phase 3 | Pseudonymisation schema (01), Multi-AZ RDS (06) | 1 week |
| Phase 4 | App-layer encryption rollout (03), DR testing (07) | 1–2 weeks |
| Phase 5 | Engage pen test firm (09) | Schedule 4–6 weeks out |

**Total: 2–4 weeks for full coverage (excluding pen test scheduling)**

---

## Controls

- [01 — Pseudonymisation](controls/01-pseudonymisation.md)
- [02 — Encryption at Rest](controls/02-encryption-at-rest.md)
- [03 — Application-Layer Encryption](controls/03-application-layer-encryption.md)
- [04 — Session Management](controls/04-session-management.md)
- [05 — Unique User Identification](controls/05-unique-identity.md)
- [06 — Multi-AZ Backup](controls/06-multi-az-backup.md)
- [07 — Disaster Recovery](controls/07-disaster-recovery.md)
- [08 — Vulnerability Scanning](controls/08-vulnerability-scanning.md)
- [09 — Penetration Testing](controls/09-penetration-testing.md)

---

## References

- [GDPR Article 32 — Official Text](https://gdpr-info.eu/art-32-gdpr/)
- [EDPB Guidelines on Personal Data Breach Notification](https://edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-92022-personal-data-breach-notification-under_en)
- [AWS KMS Best Practices](https://docs.aws.amazon.com/kms/latest/developerguide/best-practices.html)
- [CREST Pen Testing Standards](https://www.crest-approved.org/)
- [NIST SP 800-53 — Mapping Reference](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)

---

> Contributions welcome. Open a PR with corrections, additional controls, or real-world implementation examples.
