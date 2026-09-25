# JWT & Access Tokens

## 1. JWT Structure

A JWT commonly has:

```text
HEADER.PAYLOAD.SIGNATURE
```

Example:

```text
eyJhbGciOiJSUzI1NiJ9
.
eyJzdWIiOiIxMjMifQ
.
signature
```

The header and payload are Base64URL encoded, not encrypted.

## 2. Claims

Common claims:

```text
iss → issuer
sub → subject
aud → audience
exp → expiration
nbf → not before
iat → issued at
scope → permissions/scopes
```

## 3. Signature

For RS256:

```text
Private key
    ↓
sign(header + payload)
    ↓
signature
```

Verification:

```text
Public key
    ↓
verify signature
```

The public key cannot be used to create a valid signature.

## 4. JWT Is Not Encryption

Normal signed JWT:

```text
Readable payload
+
Tamper detection
```

It does not provide confidentiality.

For encrypted JWT, JWE is used.

Never put secrets/passwords in a normal JWT.

## 5. HS256 vs RS256

HS256:

```text
Shared secret
    ↓
sign + verify
```

RS256:

```text
Private key → sign
Public key  → verify
```

Asymmetric signing is convenient when many services need verification but should not be able to mint tokens.

## 6. JWKS

A JWT issuer can publish public keys through a JWKS endpoint.

```text
Authorization Server
        │
        │ JWKS
        ▼
Gateway / Services
        │
        ▼
Verify JWT
```

Key rotation becomes manageable.

## 7. JWT Through Microservices

```text
Client
  ↓ JWT
Gateway
  ↓ validate signature/issuer/audience/expiry
  ↓
Service A
  ↓
Service B
```

Services should enforce authorization for their own resources.

## 8. Role Claims

Example:

```json
{
  "sub": "user123",
  "scope": "orders:read",
  "roles": ["ORDER_VIEWER"]
}
```

Role claims are a snapshot of authorization context.

If roles change, an already-issued JWT may continue carrying old roles until it expires.

Possible approaches:

- Short-lived access tokens
- Refresh/token exchange
- Re-check authorization
- Central authorization service
- Cache where appropriate

## 9. JWT vs Opaque Token

| JWT | Opaque Token |
|---|---|
| Self-contained | Random reference |
| Local validation | Introspection |
| Usually stateless | Server-side state |
| Easy horizontal scaling | Central control |
| Revocation harder | Revocation easier |
| Claims visible | Claims hidden from client |

## Interview Questions

- Why JWT?
- Why not put secrets in JWT?
- JWT vs opaque token?
- HS256 vs RS256?
- What is JWKS?
- What happens when signing keys rotate?
- How do you revoke JWTs?
- What happens when a user's role changes?
