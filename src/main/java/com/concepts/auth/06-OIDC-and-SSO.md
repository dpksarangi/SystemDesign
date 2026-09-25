# OIDC & SSO

## 1. Why OIDC Exists

OAuth 2.0 primarily answers:

> Can this client access this resource?

OIDC adds identity:

> Who is the user?

OIDC = OpenID Connect, an identity layer built on OAuth 2.0.

## 2. High-Level Flow

```text
User
 ↓
Application
 ↓
OIDC Provider / Identity Provider
 ↓
Authentication
 ↓
Authorization Code
 ↓
Application
 ↓
Token Endpoint
 ↓
ID Token + Access Token
```

## 3. ID Token

The ID token contains identity information about the authenticated user.

Common claims:

```json
{
  "iss": "https://idp.example.com",
  "sub": "12345",
  "aud": "my-client",
  "exp": 1769500000
}
```

ID token is intended for the client application.

Access token is intended for APIs/resource servers.

Do not treat them as interchangeable.

## 4. SSO

Single Sign-On means a user can authenticate with an identity provider and access multiple applications without separately authenticating to every application.

```text
                Identity Provider
                 /      |                      /       |                    App A    App B    App C
```

## 5. Enterprise Example

A company may have:

```text
PingFederate / Enterprise IdP
          ↓
      OIDC / SAML
          ↓
     Internal Apps
```

An application can use OIDC while another legacy application uses SAML.

## 6. Identity Broker

A broker can translate protocols:

```text
Legacy IdP
   │ SAML
   ▼
Identity Broker
   │ OIDC
   ▼
Modern Application
```

This is useful when migrating legacy enterprise applications.

## 7. OIDC vs OAuth

```text
OAuth
→ authorization/delegation

OIDC
→ authentication + identity
→ built on OAuth 2.0
```

## Interview Questions

- OAuth vs OIDC?
- What is an ID token?
- ID token vs access token?
- How does SSO work?
- Can SAML and OIDC coexist?
- What is an identity broker?
