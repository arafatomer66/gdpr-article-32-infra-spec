# Control 04 — Automatic Session Logoff

**Article:** 32(1)(b)
**Effort:** Low
**Implementation Time:** 1–2 days

---

## Requirement

> "...the ongoing confidentiality, integrity, availability and resilience of processing systems"

Unattended authenticated sessions on shared or public devices allow unauthorized access to personal data. Sessions must expire automatically at both an inactivity threshold and an absolute maximum lifetime.

---

## Token Architecture

```
┌────────────────────────────────────────────────────────┐
│  Access Token (JWT)          Refresh Token (Opaque)    │
│  ──────────────────          ───────────────────────   │
│  Lifetime: 15 minutes        Lifetime: 8 hours         │
│  Storage:  memory / JS       Storage: HTTP-only cookie │
│  Validated: every request    Used: to mint new access  │
│  Revocable: no (short TTL)   Revocable: yes (DB flag)  │
└────────────────────────────────────────────────────────┘
```

Short-lived access tokens eliminate the need for per-request revocation checks. Refresh tokens are stored in HTTP-only cookies — inaccessible to JavaScript, preventing XSS theft.

---

## Implementation

### JWT Access Token

```python
import jwt
from datetime import datetime, timedelta, timezone

ACCESS_TOKEN_TTL  = timedelta(minutes=15)
REFRESH_TOKEN_TTL = timedelta(hours=8)

def issue_access_token(user_id: str, secret: str) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": user_id,
        "iat": now,
        "exp": now + ACCESS_TOKEN_TTL,
        "type": "access"
    }
    return jwt.encode(payload, secret, algorithm="HS256")
```

### Refresh Token (DB-backed, revocable)

```python
import secrets
import hashlib

def issue_refresh_token(user_id: str, db) -> str:
    raw_token = secrets.token_urlsafe(32)
    token_hash = hashlib.sha256(raw_token.encode()).hexdigest()

    db.execute("""
        INSERT INTO refresh_tokens (token_hash, user_id, expires_at, revoked)
        VALUES (%s, %s, %s, false)
    """, (token_hash, user_id, datetime.now(timezone.utc) + REFRESH_TOKEN_TTL))

    return raw_token  # returned to client, never stored raw

def validate_refresh_token(raw_token: str, db) -> str | None:
    token_hash = hashlib.sha256(raw_token.encode()).hexdigest()
    row = db.fetchone("""
        SELECT user_id FROM refresh_tokens
        WHERE token_hash = %s
          AND revoked = false
          AND expires_at > now()
    """, (token_hash,))
    return row["user_id"] if row else None
```

### Set HTTP-Only Cookie

```python
from flask import make_response

def login_response(access_token: str, refresh_token: str):
    resp = make_response({"access_token": access_token})
    resp.set_cookie(
        "refresh_token",
        refresh_token,
        httponly=True,       # not accessible to JS
        secure=True,         # HTTPS only
        samesite="Strict",   # CSRF protection
        max_age=int(REFRESH_TOKEN_TTL.total_seconds()),
        path="/auth/refresh" # scoped to refresh endpoint only
    )
    return resp
```

### Inactivity Timeout (Frontend)

```typescript
const INACTIVITY_LIMIT_MS = 15 * 60 * 1000; // 15 minutes

let inactivityTimer: ReturnType<typeof setTimeout>;

function resetInactivityTimer() {
  clearTimeout(inactivityTimer);
  inactivityTimer = setTimeout(() => {
    revokeSession();
    redirectToLogin("Session expired due to inactivity.");
  }, INACTIVITY_LIMIT_MS);
}

// Watch all user activity signals
["mousemove", "keydown", "click", "touchstart", "scroll"].forEach(event => {
  document.addEventListener(event, resetInactivityTimer, { passive: true });
});

resetInactivityTimer(); // start on page load
```

### Revoke on Logout (and on Breach Detection)

```python
def logout(user_id: str, raw_token: str, db):
    token_hash = hashlib.sha256(raw_token.encode()).hexdigest()
    db.execute("""
        UPDATE refresh_tokens
        SET revoked = true, revoked_at = now()
        WHERE token_hash = %s AND user_id = %s
    """, (token_hash, user_id))

def revoke_all_sessions(user_id: str, db):
    """Use on password reset, account compromise, or GDPR erasure request."""
    db.execute("""
        UPDATE refresh_tokens SET revoked = true, revoked_at = now() WHERE user_id = %s
    """, (user_id,))
```

---

## Database Schema

```sql
CREATE TABLE refresh_tokens (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_hash  TEXT NOT NULL UNIQUE,   -- SHA-256 of raw token
    user_id     UUID NOT NULL,
    issued_at   TIMESTAMPTZ DEFAULT now(),
    expires_at  TIMESTAMPTZ NOT NULL,
    revoked     BOOLEAN NOT NULL DEFAULT false,
    revoked_at  TIMESTAMPTZ
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens(token_hash);
```

---

## Audit Evidence

| Auditor Question | Evidence to Produce |
|-----------------|-------------------|
| How long can a session stay active? | JWT TTL config (15 min access) + refresh token max age (8 hr) |
| What happens on inactivity? | Frontend inactivity timeout code + demo |
| Can sessions be forcibly terminated? | `revoke_all_sessions` function + DB showing `revoked = true` |
| Are session tokens protected from XSS? | Cookie config showing `httponly=true, secure=true` |

---

## Common Mistakes

- **Long-lived JWTs (days/weeks)** — once issued, cannot be revoked without a denylist
- **Storing refresh tokens in localStorage** — accessible to XSS, bypasses cookie protections
- **Inactivity timeout in backend only** — user sees no feedback; UX degrades and users work around it
- **No absolute maximum lifetime** — a session started Monday should not still be valid Friday regardless of activity

---

[← Application-Layer Encryption](03-application-layer-encryption.md) | [Next: Unique Identity →](05-unique-identity.md)
