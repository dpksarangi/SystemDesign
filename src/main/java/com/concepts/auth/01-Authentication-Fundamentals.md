# Authentication Fundamentals

## 1. Authentication vs Authorization

```text
Authentication → Who are you?
Authorization   → What are you allowed to do?
```

Example:

```text
User logs in
   ↓
Authentication
   ↓
userId = 123
   ↓
Authorization
   ↓
Can user 123 access /orders/456?
```

## 2. Credential vs Token vs Session

### Credential

Something used to prove identity:

- Password
- API key
- Client secret
- Private key
- Certificate

### Token

A credential/artifact issued by an authentication or authorization system:

- JWT access token
- Opaque access token
- Refresh token
- ID token

### Session

Server-side state representing an authenticated client:

```text
session_id → userId + session metadata
```

## 3. Common Authentication Mechanisms

```text
Human
 ├── Username + Password
 ├── Session + Cookie
 ├── OIDC
 └── Passkeys / WebAuthn

Application
 ├── API Key
 ├── OAuth Client Credentials
 ├── HMAC
 └── mTLS

Enterprise SSO
 ├── OIDC
 └── SAML
```

## 4. Bearer Token

`Bearer` describes how a credential is presented.

```http
Authorization: Bearer <token>
```

Bearer does NOT mean JWT.

A bearer credential can be:

- JWT
- Opaque token
- API key

Whoever possesses a bearer credential can generally use it, so protecting it in transit and at rest matters.

## 5. Authentication Through a Gateway

```text
Client
  ↓
API Gateway
  ↓ authentication
Identity + authorization context
  ↓
Microservice
```

The gateway can perform common authentication checks, but services should still enforce authorization for their own resources.

Never trust a client-supplied identity header such as `X-User-Id` without a trusted mechanism establishing it.

## Interview Questions

- Authentication vs authorization?
- What is a bearer token?
- JWT vs session?
- API key vs access token?
- Why is TLS not authentication?
- Why shouldn't a client-supplied userId be trusted?
- Where should authorization happen in microservices?
