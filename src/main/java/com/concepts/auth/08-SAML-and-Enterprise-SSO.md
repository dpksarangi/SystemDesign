# SAML & Enterprise SSO

## 1. What Is SAML?

SAML (Security Assertion Markup Language) is an XML-based federation/SSO protocol commonly used in enterprise environments.

Core roles:

```text
User
 ↓
Service Provider (SP)
 ↓
Identity Provider (IdP)
```

## 2. Typical SSO Flow

```text
User
  ↓
Application / SP
  ↓
SAML AuthnRequest
  ↓
Identity Provider
  ↓
User authentication
  ↓
SAML Response
  ↓
Application
  ↓
Create authenticated session
```

## 3. SAML Assertion

The SAML response can contain an assertion about the user:

```text
Subject
Attributes
Authentication information
Conditions
Audience
```

The assertion is typically signed by the identity provider.

## 4. SP and IdP

### Identity Provider

Authenticates the user and issues identity assertions.

Examples:

- Enterprise IdP
- PingFederate
- Okta
- Microsoft Entra ID

### Service Provider

The application relying on the IdP.

## 5. SAML vs OIDC

| SAML | OIDC |
|---|---|
| XML-based | JSON/JWT-oriented |
| Enterprise legacy/common | Modern web/mobile/API ecosystem |
| Assertion | ID token |
| Browser SSO | Browser + mobile + modern APIs |
| More verbose | Lighter protocol model |

## 6. Can They Coexist?

Yes.

```text
                 Enterprise IdP
                /                           SAML              OIDC
              ↓                 ↓
        Legacy App         Modern App
```

An identity broker can also bridge the two.

## 7. SAML vs OAuth

They solve different problems.

SAML:

```text
Federated identity / SSO
```

OAuth:

```text
Delegated authorization
```

OIDC adds authentication on top of OAuth.

## Interview Questions

- What is SAML?
- SP vs IdP?
- What is a SAML assertion?
- SAML vs OIDC?
- Can SAML and OIDC coexist?
- Why is SAML still common in enterprises?
- What does an identity broker do?
