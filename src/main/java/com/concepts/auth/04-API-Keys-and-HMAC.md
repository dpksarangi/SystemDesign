# API Keys & HMAC

## 1. API Key

An API key is typically an application/client credential.

```http
Authorization: Bearer <API_KEY>
```

or:

```http
X-API-Key: <API_KEY>
```

Conceptually:

```text
API key
   ↓
"Which client/application is calling?"
   ↓
client identity
   ↓
permissions
```

## 2. API Key Lifecycle

```text
Generate
   ↓
Store securely
   ↓
Issue to client
   ↓
Validate
   ↓
Rotate / revoke
```

A high-entropy random API key should be generated using a cryptographically secure random generator.

## 3. Storage Design

One practical design:

```text
API key:
ak_live_<key_id>_<secret>
```

Database:

```text
api_keys
────────────────────────────────────
key_id
secret_hash
client_id
status
scopes
created_at
expires_at
last_used_at
```

For high-entropy API keys, a cryptographic hash/fingerprint can be used for lookup/verification. Password-style KDFs are another option when appropriate.

Do not assume the provider's internal storage implementation unless documented.

## 4. Validation

```text
Request
   ↓
Extract API key
   ↓
Extract key_id
   ↓
Lookup credential metadata
   ↓
Verify secret
   ↓
Check status/expiry
   ↓
Check scopes/permissions
   ↓
Allow request
```

The database is the source of truth. Cache is an optimization, not a requirement.

## 5. API Key vs Password

```text
Password → usually human credential
API key  → usually application credential
```

Both are secrets, but their lifecycle and purpose differ.

## 6. API Key vs OAuth Access Token

API key:

```text
static credential
       ↓
API
```

OAuth:

```text
client credentials / authorization
       ↓
authorization server
       ↓
access token
       ↓
API
```

OAuth can provide token expiration, scopes, refresh, consent, and delegated authorization.

## 7. HMAC

HMAC = Hash-based Message Authentication Code.

```text
signature = HMAC(shared_secret, request_data)
```

Client:

```text
request
   +
shared secret
   ↓
HMAC signature
```

Server independently calculates the signature and compares it.

HMAC provides:

- Authentication of the party possessing the secret
- Request integrity

## 8. API Key vs HMAC

API key:

```http
Authorization: Bearer <API_KEY>
```

HMAC:

```http
X-Client-Id: client123
X-Timestamp: 1769500000
X-Signature: <computed-signature>
```

The HMAC signature is derived from the request.

## 9. Replay Protection

HMAC alone does not automatically prevent replay.

Use:

- Timestamp
- Nonce/request ID
- Server-side freshness validation

Conceptually:

```text
signature = HMAC(
    secret,
    method + path + timestamp + body
)
```

## Interview Questions

- API key vs JWT?
- API key vs password?
- API key vs HMAC?
- Where should API keys be stored?
- How do you rotate an API key without downtime?
- How do you revoke a leaked key?
- Why does HMAC need a shared secret?
- How do you prevent replay attacks?
