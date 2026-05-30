# Control 03 — Application-Layer Encryption

**Article:** 32(1)(a)
**Effort:** High
**Implementation Time:** 3–5 days

---

## Requirement

> "...the pseudonymisation and encryption of personal data"

Disk-level encryption (Control 02) protects against physical media theft. Application-layer encryption protects against **database administrator access** — someone with full DB credentials still sees only ciphertext for sensitive fields.

---

## When to Apply This Control

Not every column needs application-layer encryption. Apply it to fields with high breach impact:

| Field Type | Examples | Encrypt? |
|-----------|---------|---------|
| Government IDs | NID, passport number, SSN | Yes |
| Financial | Bank account, card number | Yes |
| Health | Diagnosis, medication, biometrics | Yes |
| Contact | Email, phone | Usually (risk-based) |
| Preferences | Theme, language | No |
| Internal | Timestamps, status flags | No |

---

## Implementation

### Key Management with HashiCorp Vault

```bash
# Enable the transit secrets engine
vault secrets enable transit

# Create an encryption key for personal data
vault write -f transit/keys/personal-data \
  type=aes256-gcm96 \
  exportable=false \
  allow_plaintext_backup=false

# Create a policy: app can encrypt/decrypt, cannot export key
vault policy write personal-data-app - <<EOF
path "transit/encrypt/personal-data" {
  capabilities = ["update"]
}
path "transit/decrypt/personal-data" {
  capabilities = ["update"]
}
EOF
```

### Python: Field-Level Encrypt/Decrypt

```python
import base64
import hvac
from dataclasses import dataclass

vault_client = hvac.Client(url="https://vault.internal")
vault_client.auth.kubernetes.login(role="personal-data-app", jwt=get_k8s_jwt())

KEY_NAME = "personal-data"

def encrypt_field(plaintext: str) -> str:
    """Returns a Vault transit ciphertext token — safe to store in DB."""
    response = vault_client.secrets.transit.encrypt_data(
        name=KEY_NAME,
        plaintext=base64.b64encode(plaintext.encode()).decode()
    )
    return response["data"]["ciphertext"]  # e.g. "vault:v1:abc123..."

def decrypt_field(ciphertext: str) -> str:
    """Decrypts a Vault transit token back to plaintext."""
    response = vault_client.secrets.transit.decrypt_data(
        name=KEY_NAME,
        ciphertext=ciphertext
    )
    return base64.b64decode(response["data"]["plaintext"]).decode()

@dataclass
class PersonRecord:
    person_uuid: str
    national_id_enc: str   # stored encrypted
    health_notes_enc: str  # stored encrypted

    @property
    def national_id(self) -> str:
        return decrypt_field(self.national_id_enc)

    @classmethod
    def create(cls, person_uuid: str, national_id: str, health_notes: str):
        return cls(
            person_uuid=person_uuid,
            national_id_enc=encrypt_field(national_id),
            health_notes_enc=encrypt_field(health_notes)
        )
```

### Database Schema

```sql
CREATE TABLE person_sensitive (
    person_uuid      UUID PRIMARY KEY REFERENCES person_identifiers(person_uuid),
    national_id_enc  TEXT,        -- vault:v1:... ciphertext
    health_notes_enc TEXT,        -- vault:v1:... ciphertext
    updated_at       TIMESTAMPTZ DEFAULT now()
);
```

A DBA querying this table directly sees:

```
person_uuid | national_id_enc               | health_notes_enc
------------+-------------------------------+------------------
3f8a2b1c... | vault:v1:8XmP3Kq2...         | vault:v1:9NrT4Lw...
```

They cannot decrypt without Vault access, which is governed by a separate policy.

### Key Rotation (Zero-Downtime)

```bash
# Rotate the key — Vault keeps old key versions for decryption
vault write -f transit/keys/personal-data/rotate

# Rewrap all existing ciphertext to latest key version (background job)
vault write transit/rewrap/personal-data ciphertext="vault:v1:..."
# Returns: vault:v2:...
```

```python
def rewrap_all_records(db):
    """Background job: upgrade all ciphertext to latest key version."""
    records = db.query("SELECT person_uuid, national_id_enc FROM person_sensitive")
    for record in records:
        new_ciphertext = vault_client.secrets.transit.rewrap_data(
            name=KEY_NAME,
            ciphertext=record["national_id_enc"]
        )["data"]["ciphertext"]
        db.execute(
            "UPDATE person_sensitive SET national_id_enc = %s WHERE person_uuid = %s",
            (new_ciphertext, record["person_uuid"])
        )
```

---

## Vault Access Policy Boundaries

```
┌─────────────────────────────────────────────────────────┐
│                    Vault Transit Engine                  │
│                                                         │
│  personal-data key                                      │
│  ├── App service pod  → encrypt + decrypt  ✓            │
│  ├── DBA user         → NO ACCESS          ✗            │
│  ├── Analytics role   → NO ACCESS          ✗            │
│  └── Security team    → audit only         ✓            │
└─────────────────────────────────────────────────────────┘
```

---

## Audit Evidence

| Auditor Question | Evidence to Produce |
|-----------------|-------------------|
| Can DBAs read sensitive fields? | Direct DB query returning `vault:v1:...` ciphertext |
| Where are the decryption keys? | Vault audit log showing only app service role has decrypt policy |
| How often are keys rotated? | `vault read transit/keys/personal-data` showing `latest_version` and `min_decryption_version` |
| What happens if Vault is down? | Runbook showing graceful degradation + Vault HA setup |

---

## Common Mistakes

- **Fernet with hardcoded keys** — the key ends up in environment variables, git history, or logs
- **Encrypting primary keys** — makes indexing and joins impossible; encrypt *values*, not identifiers
- **No rewrap process** — old key versions stay active forever, undermining rotation
- **Synchronous Vault calls on every read** — cache decrypted values in memory for the request lifecycle, not on disk

---

[← Encryption at Rest](02-encryption-at-rest.md) | [Next: Session Management →](04-session-management.md)
