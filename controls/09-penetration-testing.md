# Control 09 — Annual Penetration Testing

**Article:** 32(1)(d)
**Effort:** High
**Implementation Time:** 4–8 weeks (scheduling, execution, remediation)

---

## Requirement

> "...a process for regularly testing, assessing and evaluating the effectiveness of technical and organisational measures for ensuring the security of the processing"

Automated scanners (Control 08) find known CVEs. Penetration testing finds **logic flaws, authentication bypasses, and business logic vulnerabilities** that scanners cannot detect. Manual testing by qualified testers is required at minimum annually.

---

## Automated vs Manual Testing

| | Automated Scanning (Control 08) | Penetration Testing (Control 09) |
|--|--------------------------------|----------------------------------|
| Frequency | Every commit + daily | Annually minimum |
| Finds CVEs | Yes | Yes (plus more) |
| Finds auth bypass | No | Yes |
| Finds business logic flaws | No | Yes |
| Finds chained exploits | No | Yes |
| Auditor weight | Medium | High |
| Cost | Low (CI/CD overhead) | High (£10k–£50k+) |

---

## Certification Standards

Specify certification requirements in your pen test RFP. Auditors look for:

| Certification | Scope | Jurisdiction |
|--------------|-------|-------------|
| **CREST** | Infrastructure + web application | UK, EU, global |
| **CHECK** | UK government systems | UK |
| **OSCP / OSCE** | Individual tester qualification | Global |
| **PTES / OWASP Testing Guide** | Methodology standard | Global |

> For GDPR compliance, **CREST-certified firms are the standard auditors expect.** Individual OSCP certifications on the tester's CV are a secondary signal.

---

## Scope Definition

Define scope precisely before engagement. Ambiguous scope leads to gaps.

```markdown
## Penetration Test Scope — ACME Platform (2024)

### In Scope
- Web application: https://app.acme.com (production-equivalent staging)
- REST API: https://api.acme.com/v1 (all endpoints)
- Authentication flows: login, MFA, password reset, session management
- Authorization: RBAC, data isolation between tenants
- Personal data endpoints: user profile, order history, export/download
- Infrastructure: EKS cluster, RDS, S3, KMS (review only — no destructive tests)
- Third-party integrations: payment gateway callback handling

### Out of Scope
- Production database (use staging with anonymised data copy)
- DoS / DDoS testing
- Third-party services (Stripe, SendGrid)
- Social engineering of employees

### Rules of Engagement
- Testing window: Mon–Fri 09:00–17:00 BST
- Emergency stop contact: [Name] +44 7xxx xxxxxx
- Notify before testing authentication rate limiting (may trigger lockouts)
- All findings embargoed until remediation complete
```

---

## What Good Pen Testers Test

Ensure your scope document and vendor RFP include these categories:

```
Authentication & Session Management
├── Credential stuffing / brute force protection
├── MFA bypass techniques
├── Session fixation and hijacking
├── JWT algorithm confusion (alg:none, RS256→HS256)
└── Password reset flow vulnerabilities

Authorization
├── IDOR (Insecure Direct Object Reference) — can user A read user B's data?
├── Privilege escalation — can regular user access admin functions?
├── Tenant isolation — can tenant A read tenant B's data?
└── API endpoint authorization — unauthenticated access to protected routes

Injection
├── SQL injection (parametrized queries do not always prevent this)
├── NoSQL injection
├── LDAP / XML injection
└── Server-Side Template Injection (SSTI)

Personal Data Specific
├── Mass assignment — can users update fields they shouldn't?
├── Data export — does the export include more data than it should?
├── GDPR erasure — does deletion actually remove data from all stores?
└── Audit log tampering — can users modify their own audit trail?

Infrastructure
├── S3 bucket misconfigurations
├── IAM policy review
├── Security group review
└── Secrets in environment variables / logs
```

---

## Remediation Process

After receiving the report, track all findings in your issue tracker:

```markdown
## Remediation Tracking

| Finding ID | Title                          | Severity | Owner | Due Date   | Status |
|-----------|-------------------------------|----------|-------|------------|--------|
| PT-001    | IDOR on /api/orders/{id}       | CRITICAL | @dev1 | 2024-02-20 | ✅ Fixed |
| PT-002    | JWT alg:none accepted          | HIGH     | @dev2 | 2024-02-27 | ✅ Fixed |
| PT-003    | Rate limiting on password reset| MEDIUM   | @dev1 | 2024-03-15 | 🔄 In Progress |
| PT-004    | Missing HSTS header on API     | LOW      | @dev3 | 2024-04-01 | ⬜ Pending |
```

**Remediation SLAs:**
- CRITICAL: 24 hours (emergency patch)
- HIGH: 7 days
- MEDIUM: 30 days
- LOW: 90 days / risk accepted with sign-off

---

## Retest Process

After remediating CRITICAL and HIGH findings, request a retest from the pen test firm:

```
1. Fix finding PT-001 (IDOR)
2. Deploy to staging
3. Notify pen test firm of fix
4. Firm retests specific finding in isolation
5. Firm issues retest confirmation note
6. Update finding status to "Verified Fixed"
```

The retest confirmation note is part of your audit evidence.

---

## Report Retention

Store pen test reports securely — they contain exploitation paths:

- Encrypt at rest (KMS)
- Access restricted to: CISO, CTO, Lead Engineer, Auditors
- Retain for minimum 3 years
- Include in document management system with access log

---

## Audit Evidence

| Auditor Question | Evidence to Produce |
|-----------------|-------------------|
| When was your last pen test? | Report dated within 12 months, signed by CREST-certified firm |
| What was found? | Executive summary from report (full report available on request) |
| How did you remediate findings? | Issue tracker showing all CRITICAL/HIGH closed within SLA |
| Were fixes verified? | Retest confirmation notes from pen test firm |
| What's your next test date? | Signed contract or letter of engagement with scheduled date |

---

## Budget Guidance

| Test Type | Scope | Typical Cost |
|-----------|-------|-------------|
| Web application only | 5–10 day engagement | £8k–£20k |
| Web + API + mobile | 10–15 day engagement | £15k–£35k |
| Full infrastructure + web | 15–25 day engagement | £25k–£60k |
| Red team exercise | 4–6 weeks | £40k–£100k+ |

*Costs vary significantly by geography, firm reputation, and scope complexity.*

---

## Common Mistakes

- **Self-assessment instead of external test** — internal teams cannot objectively test their own work; auditors reject this
- **Testing only once, at product launch** — annually is the minimum; after major architectural changes, test again
- **No scope document** — testers will miss critical areas; define scope before signing the contract
- **Remediating without retesting** — a fix that doesn't fix the vulnerability is worse than no fix; retest everything CRITICAL/HIGH
- **Sharing the full report broadly** — the report is a roadmap for attackers if leaked; treat it as highly confidential

---

[← Vulnerability Scanning](08-vulnerability-scanning.md) | [← Back to README](../README.md)
