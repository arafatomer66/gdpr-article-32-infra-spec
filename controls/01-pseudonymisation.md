# Control 01 — Pseudonymisation

**Article:** 32(1)(a)
**Effort:** Medium
**Implementation Time:** 3–5 days

---

## Requirement

> "...the pseudonymisation and encryption of personal data"

Store personal data such that an attacker who gains read access to the working dataset cannot identify natural persons without access to a separately-secured identifier lookup table.

---

## What Pseudonymisation Is (and Is Not)

| | Pseudonymisation | Anonymisation |
|--|-----------------|---------------|
| Re-identification possible? | Yes (with key) | No |
| Still personal data under GDPR? | **Yes** | No |
| Reduces risk? | **Yes** | Fully removes risk |
| Practical in live systems? | **Yes** | Often not |

Pseudonymisation does not remove your GDPR obligations — it reduces the risk and harm of a breach.

---

## Implementation

### Schema Design

```sql
-- Restricted identifier table (high-privilege access only)
CREATE TABLE person_identifiers (
    person_uuid   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         TEXT NOT NULL UNIQUE,
    full_name     TEXT NOT NULL,
    passport_no   TEXT,
    created_at    TIMESTAMPTZ DEFAULT now()
);

-- Working dataset uses UUID only — no direct identifiers
CREATE TABLE orders (
    order_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_uuid   UUID NOT NULL REFERENCES person_identifiers(person_uuid),
    product_sku   TEXT,
    amount        NUMERIC(10,2),
    created_at    TIMESTAMPTZ DEFAULT now()
);

-- Analytics, logs, ML features — no identifiers at all
CREATE TABLE order_events (
    event_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_uuid   UUID NOT NULL,
    event_type    TEXT,
    occurred_at   TIMESTAMPTZ DEFAULT now()
);
```

### Role-Based Access Control

```sql
-- Application service role — cannot read identifiers
CREATE ROLE app_service LOGIN PASSWORD '...';
GRANT SELECT, INSERT, UPDATE ON orders TO app_service;
GRANT SELECT, INSERT, UPDATE ON order_events TO app_service;
-- No grant on person_identifiers

-- Identity service role — sole authorized accessor
CREATE ROLE identity_service LOGIN PASSWORD '...';
GRANT SELECT, INSERT, UPDATE ON person_identifiers TO identity_service;

-- Analytics role — pseudonymised data only
CREATE ROLE analytics_reader LOGIN;
GRANT SELECT ON order_events TO analytics_reader;
-- No access to person_identifiers or orders
```

### UUID Generation

Always use non-sequential, non-guessable UUIDs. Never use auto-incrementing integers as person references — sequential IDs leak record counts and enable enumeration attacks.

```python
import uuid

def create_person(email: str, full_name: str) -> uuid.UUID:
    person_uuid = uuid.uuid4()  # random, non-guessable
    db.execute(
        "INSERT INTO person_identifiers (person_uuid, email, full_name) VALUES (%s, %s, %s)",
        (str(person_uuid), email, full_name)
    )
    return person_uuid
```

---

## Verification

```sql
-- Confirm app_service cannot access identifiers
SET ROLE app_service;
SELECT * FROM person_identifiers;
-- Expected: ERROR: permission denied for table person_identifiers

-- Confirm working dataset contains no direct identifiers
\d orders
-- Expected: columns are order_id, person_uuid, product_sku, amount, created_at
-- No email, name, or contact fields
```

---

## Audit Evidence

| Auditor Question | Evidence to Produce |
|-----------------|-------------------|
| How do you prevent re-identification? | `\dp person_identifiers` showing only `identity_service` has SELECT |
| Is the identifier table physically separate? | Schema dump showing `person_identifiers` in restricted schema |
| Could a DB breach expose customer names? | Query as `app_service` returning permission denied on identifiers |

---

## Common Mistakes

- **Using email as a foreign key** in working tables — this is not pseudonymisation, it is just a join
- **Storing UUIDs and emails in the same table** with a "pseudonymised" label — both fields are in scope
- **Giving the application role access to both tables** — defeats the purpose entirely
- **Using sequential IDs** — leaks record counts, enables enumeration

---

[← Back to README](../README.md) | [Next: Encryption at Rest →](02-encryption-at-rest.md)
