# Authentication & Security

Interview-ready notes covering common authentication, authorization, token, API security, and enterprise SSO patterns.

## Topics

1. [Authentication Fundamentals](01-Authentication-Fundamentals.md)
2. [Username, Password & Sessions](02-Username-Password-and-Sessions.md)
3. [TLS & HTTPS](03-TLS-and-HTTPS.md)
4. [API Keys & HMAC](04-API-Keys-and-HMAC.md)
5. [OAuth 2.0](05-OAuth2.md)
6. [OIDC & SSO](06-OIDC-and-SSO.md)
7. [JWT & Access Tokens](07-JWT-and-Access-Tokens.md)
8. [SAML & Enterprise SSO](08-SAML-and-Enterprise-SSO.md)

## Mental Model

```text
Authentication
    └── Who are you?

Authorization
    └── What are you allowed to do?

Channel Security
    └── Can someone intercept/change the communication?

Credential / Token
    └── What evidence do you present?

Session
    └── How does the server remember an authenticated client?
```

## Quick Selection Guide

| Requirement | Common choice |
|---|---|
| Human login | OIDC |
| Traditional server-rendered app | Session + Cookie |
| Machine-to-machine | OAuth 2.0 Client Credentials |
| Simple partner API | API Key |
| Request signing | HMAC |
| Enterprise legacy SSO | SAML |
| Modern enterprise SSO | OIDC |
| Service-to-service identity | OAuth / mTLS |
| Self-contained access token | JWT |
| Centralized validation/revocation | Opaque Token |

> These are common patterns, not universal rules. The right choice depends on threat model, trust boundaries, lifecycle, and operational requirements.
